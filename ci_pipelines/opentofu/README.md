# OpenTofu pipeline

Ten pipeline realizuje standardowy przepływ IaC dla OpenTofu (Terraform-compatible) w GitLab CI:

1. Walidacja formatowania (`tofu fmt`)
2. Walidacja konfiguracji (`tofu validate`)
3. Linting (`tflint`)
4. Plan (`tofu plan`)
5. Apply (`tofu apply`) – kontrolowane regułami oraz zależne od publikacji wersji

Pipeline wykorzystuje **GitLab Terraform State** jako backend dla OpenTofu.

---
## Wymagania

* Repozytorium zawiera kod OpenTofu (np. `*.tf`).
* Dostęp do obrazu:

  * `registry.rachuna-net.pl/pl.rachuna-net/containers/opentofu:1.1.1`
* Repozytorium ma dostęp do API GitLab (standardowo zapewnia `CI_JOB_TOKEN`).
* Zdefiniowane etapy w `.gitlab-ci.yml` obejmują co najmniej:

  * `validate`
  * `unit-test`
  * `deploy`
* W pipeline istnieje job `📍 Publish Version` (wymagany przez `needs` w `opentofu apply`).

---
## Struktura pipeline

### Zmienne

| Zmienna          | Domyślna wartość                                                   | Opis                                                              |
| ---------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------- |
| `OPENTOFU_IMAGE` | `registry.rachuna-net.pl/pl.rachuna-net/containers/opentofu:1.1.1` | Obraz kontenera z OpenTofu oraz narzędziami (`tflint`).           |
| `TF_STATE_NAME`  | `production` (fallback)                                            | Nazwa stanu w GitLab Terraform State; używana w adresie backendu. |

**Uwaga:** `TF_STATE_NAME` nie jest wymagane — jeśli nie ustawisz, pipeline użyje `production`.

---
## Include: `tofu-init.sh.yml`

Pipeline dołącza lokalny plik:

```yaml
include:
  - local: "pipelines/opentofu/tofu-init.sh.yml"
```

W nim znajduje się snippet `.!tofu-init.sh.yml`, który wykonuje `tofu init` skonfigurowany pod GitLab Terraform State.

### Co robi `.tofu-init.sh.yml`

* wypisuje wersję OpenTofu (`tofu version`)
* wykonuje `tofu init` z parametrami backendu GitLab:

**Backend URL (state):**

* `address=${CI_SERVER_URL}/api/v4/projects/${CI_PROJECT_ID}/terraform/state/${TF_STATE_NAME:-production}`

**Locking:**

* `lock_address=.../lock`
* `unlock_address=.../lock`
* `lock_method=POST`
* `unlock_method=DELETE`

**Autoryzacja:**

* `username=gitlab-ci-token`
* `password=${CI_JOB_TOKEN}`

**Retry:**

* `retry_wait_min=5`

### Dlaczego to jest ważne

Dzięki temu:

* stan Terraform/OpenTofu jest przechowywany centralnie w GitLab (zamiast lokalnie lub w S3),
* lock działa przez GitLab API, co ogranicza ryzyko równoległych `apply`.

---

## Joby: opis i zachowanie

### 1) `🕵 opentofu fmt` (stage: `validate`)

**Cel:** Weryfikacja formatowania plików OpenTofu bez modyfikacji kodu.

**Komenda:**

```bash
tofu fmt -recursive -check
```

**Kiedy się uruchamia:** zawsze (na sukces), na każdym pipeline.

**Efekt:** pipeline failuje, jeśli formatowanie nie jest zgodne.

---

### 2) `✅ opentofu validate` (stage: `validate`)

**Cel:** Walidacja składni i spójności konfiguracji.

**Wymaga init:** tak (żeby pobrać providerów/moduły).

**Komenda:**

```bash
tofu validate
```

**Efekt:** pipeline failuje, jeśli konfiguracja jest błędna.

---

### 3) `✅ tflint` (stage: `validate`)

**Cel:** Linting kodu IaC (wykrywanie antywzorców i problemów jakościowych).

**Wymaga init:** tak (często potrzebne do kontekstu providerów/modułów).

**Komenda:**

```bash
tflint
```

**Ważne:** jeśli `tflint` ma być użyte sensownie, repo zwykle posiada `.tflint.hcl` lub konfigurację domyślną.

---

### 4) `🧪 opentofu plan` (stage: `unit-test`)

**Cel:** Wygenerowanie planu zmian dla infrastruktury.

**Komenda:**

```bash
tofu plan
```

**Zachowanie:** plan jest generowany, ale w aktualnej konfiguracji:

* nie jest zapisywany jako artifact,
* nie jest publikowany jako raport,
* `apply` nie konsumuje planu (wykonuje niezależne `apply`).

**Rekomendacja (opcjonalna):**
Jeśli chcesz deterministycznego wdrożenia, rozważ:

* `tofu plan -out=plan.bin`
* artifact `plan.bin`
* `tofu apply plan.bin`

---

### 5) `💥 opentofu apply` (stage: `deploy`)

**Cel:** Zastosowanie zmian w infrastrukturze.

**Komenda:**

```bash
tofu apply -auto-approve
```

**Zależności:**

```yaml
needs:
  - job: 📍 Publish Version
```

To wymusza, aby `apply` wykonało się dopiero po ukończeniu joba publikującego wersję (np. release/tagging).

#### Reguły uruchomienia (`rules`)

1. **Jeżeli pipeline jest uruchomiony z taga (`CI_COMMIT_TAG`) → apply jest blokowane:**

   ```yaml
   - if: $CI_COMMIT_TAG
     when: never
   ```

   Skutek: nie ma wdrożeń „z taga”.

2. **Jeżeli branch = domyślny (`main/master`) → apply odpala się automatycznie:**

   ```yaml
   - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
     when: on_success
   ```

3. **Jeżeli są otwarte MR (`CI_OPEN_MERGE_REQUESTS`) → apply jest manualne:**

   ```yaml
   - if: $CI_OPEN_MERGE_REQUESTS
     when: manual
   ```

   Skutek: dla gałęzi z MR nie ma automatycznego wdrażania; ktoś musi ręcznie kliknąć job.

---

## Jak ustawić nazwę stanu (TF_STATE_NAME)

### Domyślnie

Jeśli nic nie ustawisz, backend użyje:

* `production`

### Przykład: środowiska

Możesz zdefiniować `TF_STATE_NAME` per environment:

* `production`
* `staging`
* `dev`

Przykład ustawienia w pipeline (np. w `variables:` lub w UI projektu):

```yaml
variables:
  TF_STATE_NAME: "staging"
```

---

## Bezpieczeństwo i uprawnienia

* `tofu init` używa `CI_JOB_TOKEN` jako hasła do backendu GitLab Terraform State.
* Uprawnienia `CI_JOB_TOKEN` są zależne od ustawień projektu i GitLab.
* Jeśli wdrożenia mają wykonywać akcje na chmurze/infra (np. Proxmox, Vault, DNS, itp.), wymagane są odpowiednie sekrety (np. tokeny providerów) dostarczane przez:

  * GitLab CI Variables (masked/protected),
  * Vault, jeśli masz integrację,
  * dedykowane service accounts.

---

## Typowe problemy i diagnoza

### `tofu validate` / `tofu plan` padają, bo brak providerów

Sprawdź, czy `.tofu-init.sh.yml` jest wywoływany w `before_script` dla jobów:

* `validate`
* `tflint`
* `plan`
* `apply`

W Twojej konfiguracji jest poprawnie.

### Locking / backend nie działa

Zweryfikuj:

* czy GitLab ma włączoną funkcję Terraform State dla projektu,
* czy ścieżka API jest dostępna,
* czy `CI_JOB_TOKEN` ma dostęp do projektu (zwykle ma).

### `apply` uruchamia się w niechcianych sytuacjach

Sprawdź warunki:

* Domyślny branch wdraża automatycznie.
* Gałęzie z MR: manual.
* Tagi: nigdy.

---

## Sugestie ulepszeń (opcjonalne, ale praktyczne)

1. **Zapisywanie planu jako artifact i używanie go w apply**

   * ogranicza drift pomiędzy planem i apply.

2. **Environment + TF_STATE_NAME per environment**

   * łatwiejsza separacja stanów: `dev/staging/prod`.

3. **Cache providerów/pluginów**

   * skraca czas `init` i `plan`.

4. **Raport z planu jako komentarz do MR**

   * poprawia review IaC.

---

## Referencja: definicje z pipeline

### `.gitlab-ci.yml` (fragment)

* `🕵 opentofu fmt`: format check
* `✅ opentofu validate`: walidacja
* `✅ tflint`: lint
* `🧪 opentofu plan`: plan
* `💥 opentofu apply`: wdrożenie kontrolowane regułami

### `pipelines/opentofu/tofu-init.sh.yml`

* `.tofu-init.sh.yml`: init + backend GitLab Terraform State + locking
