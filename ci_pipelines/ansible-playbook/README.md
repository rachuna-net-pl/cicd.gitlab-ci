# ▶️ Ansible Playbook

Ten pipeline służy do uruchamiania **playbooków Ansible** w repozytoriach wdrożeniowych. Zawiera walidację (lint + check), przygotowanie dynamicznego deploymentu oraz trigger do uruchomienia jobów per środowisko.

---
## Wymagania

* Repozytorium zawiera playbooki w `playbooks/*.yml`.
* Istnieje inventory: `inventory/hosts.yml` (domyślna ścieżka).
* Jeżeli używasz ról z Galaxy, dostępny jest `requirements.yml`.
* Obrazy kontenerów:
  * `$IMAGE_ANSIBLE` (Ansible + ansible-lint)
  * `$IMAGE_PYTHON` (generowanie dynamicznego pipeline)
* Dostępne są helpery:
  * `.helper_gitlab-ci.sh` (konfiguracja środowiska i dostępu do repo)
  * `.helper_readme.sh` (wskazanie dokumentacji po wykonaniu joba)

---
## Struktura pipeline

### Include: `ansible_init.sh.yml`

Pipeline dołącza lokalny plik:

```yaml
include:
  - local: "ci_pipelines/ansible-playbook/ansible_init.sh.yml"
```

W nim znajduje się snippet `.ansible_init.sh`, który:

* ustawia zmienne środowiskowe Ansible,
* wyłącza sprawdzanie kluczy hostów,
* wymusza kolorowy output,
* instaluje role z `requirements.yml` do `playbooks/roles`.

**Domyślne wartości ustawiane przez `.ansible_init.sh`:**

* `ANSIBLE_INVENTORY=inventory/hosts.yml`
* `ANSIBLE_HOST_KEY_CHECKING=false`
* `ANSIBLE_FORCE_COLOR=true`
* `ANSIBLE_USER=techuser`

---
## Zmienne

| Zmienna             | Domyślna wartość                                                           | Opis                                                                 |
| ------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `IMAGE_ANSIBLE`     | *(zdefiniowane w CI)*                                                      | Obraz kontenera z Ansible i ansible-lint.                            |
| `IMAGE_PYTHON`      | *(zdefiniowane w CI)*                                                      | Obraz do generowania dynamicznego pipeline.                          |
| `ANSIBLE_INVENTORY` | `inventory/hosts.yml`                                                      | Ścieżka do inventory.                                                |
| `ANSIBLE_VARS`      | *(brak)*                                                                   | Dodatkowe `--extra-vars` przekazywane do playbooka.                  |
| `ENVIRON`           | *(brak)*                                                                   | Limit hostów/środowiska przekazywany jako `--limit`.                 |
| `DOCS_MD_FILE_PATH` | `ci_pipelines/ansible-playbook/README.md`                                  | Ścieżka do dokumentacji używana przez `.helper_readme.sh`.           |
| `DEPLOY_ON`         | *(brak)*                                                                   | Jeśli ustawione, automatycznie uruchamia deploy tylko dla tego env.  |
| `PARENT_PIPELINE_ID`| *(ustawiane w triggerze)*                                                  | ID pipeline nadrzędnego dla dynamicznego deploymentu.                |

---
## Joby: opis i zachowanie

### 1) `🕵 Prepare for dynamic deployment` (stage: `prepare`)

**Cel:** generuje dynamiczny pipeline dla środowisk zdefiniowanych w GitLab Environments.

**Co robi:**

* pobiera listę środowisk z GitLab API,
* buduje `.ci/deployment.yml` z jobami `💥 ansible playbook:<env>`,
* wystawia artefakt `.ci/deployment.yml`.

**Kiedy się uruchamia:**

* uruchamia się w normalnych pipeline (nie w `schedule`).

---
### 2) `🧪 ansible-lint` (stage: `validate`)

**Cel:** uruchomienie `ansible-lint` dla całego repo.

**Komenda:**

```bash
ansible-lint --force-color .
```

---
### 3) `✅ ansible-playbook check` (stage: `validate`)

**Cel:** uruchomienie wszystkich playbooków w trybie `--check`.

**Komenda:**

```bash
for file in playbooks/*.yml; do
  ansible-playbook -i $ANSIBLE_INVENTORY $file --check --extra-vars "$ANSIBLE_VARS"
done
```

---
### 4) `💥 dynamic deployment` (stage: `deploy`)

**Cel:** triggeruje pipeline z artefaktu `.ci/deployment.yml`.

**Zachowanie:**

* uruchamia się tylko na `default branch`,
* nie uruchamia się dla tagów ani pipeline `schedule`,
* pomija jeśli są otwarte MR do branchy.

---
### 5) `💥 ansible playbook:<env>` (stage: `deploy`, w pipeline dynamicznym)

**Cel:** uruchomienie playbooka dla konkretnego środowiska.

**Ważne:**

* joby są tworzone dynamicznie per środowisko z GitLab,
* standardowo są `manual`, ale jeśli `DEPLOY_ON == <env>`, uruchamiają się automatycznie.

---
## Typowe problemy i diagnoza

### Brak ról lub błędy `ansible-galaxy`

* Upewnij się, że `requirements.yml` istnieje w repozytorium.
* Sprawdź, czy role są instalowane do `playbooks/roles`.

### Brak hostów / puste inventory

* Zweryfikuj, czy `inventory/hosts.yml` istnieje i zawiera hosty.
* Sprawdź, czy `ENVIRON` wskazuje poprawną grupę w inventory.

### Brak informacji o dokumentacji po jobie

* Sprawdź, czy zmienna `DOCS_MD_FILE_PATH` jest ustawiona.
* W tym pipeline jest ustawiona na `pipelines/ansible-playbook/README.md`.

---
## Referencja: definicje z pipeline

* `.ansible_init.sh`: ustawienie środowiska + instalacja ról
* `🕵 Prepare for dynamic deployment`: generowanie `.ci/deployment.yml`
* `🧪 ansible-lint`: lint repo
* `✅ ansible-playbook check`: check dla `playbooks/*.yml`
* `💥 dynamic deployment`: trigger dynamicznego deploymentu
