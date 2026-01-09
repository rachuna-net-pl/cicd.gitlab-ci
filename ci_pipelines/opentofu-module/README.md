# OpenTofu Module

Ten pipeline jest przeznaczony dla **repozytoriów z modułami OpenTofu/Terraform**, czyli takich, które nie wykonują wdrożeń infrastruktury (`apply`), a jedynie dostarczają wielokrotnego użytku kod IaC. Celem jest utrzymanie:

* spójnego formatowania (`tofu fmt`)
* jakości kodu (`tflint`)
* aktualnej dokumentacji modułu (`terraform-docs`) oraz wymuszenie, aby README nie „dryfowało” względem kodu

Pipeline działa wyłącznie w trybie walidacyjnym (stage: `validate`).

---
## Wymagania

* Repozytorium zawiera moduł OpenTofu/Terraform (`*.tf`).
* W repo istnieje `README.md` (wymagane przez krok diff).
* Obraz kontenera zawiera narzędzia:

  * `tofu`
  * `tflint`
  * `terraform-docs`
* Dostępny jest wspólny helper:

  * `.helper_gitlab-ci.sh` (referencja w `before_script`).

---
## Struktura pipeline

### Zmienne

| Zmienna          | Domyślna wartość                                                   | Opis                                                      |
| ---------------- | ------------------------------------------------------------------ | --------------------------------------------------------- |
| `OPENTOFU_IMAGE` | `registry.rachuna-net.pl/pl.rachuna-net/containers/opentofu:1.0.0` | Obraz narzędziowy z OpenTofu i narzędziami walidacyjnymi. |

---

## Joby: opis i zachowanie

### 1) `🕵 opentofu fmt` (stage: `validate`)

**Cel:** Sprawdzenie formatowania plików modułu bez modyfikacji repo.

**Komenda:**

```bash
tofu fmt -recursive -check
```

**Kiedy się uruchamia:** zawsze (na sukces), na każdym pipeline.

**Efekt:** pipeline failuje, jeżeli pliki nie są sformatowane zgodnie z `tofu fmt`.

---

### 2) `✅ tflint` (stage: `validate`)

**Cel:** Linting modułu – wykrywanie problemów jakościowych oraz antywzorców.

**Komenda:**

```bash
tflint
```

**Uwagi praktyczne:**

* Jeżeli w repozytorium jest `.tflint.hcl`, to `tflint` użyje tej konfiguracji.
* W pipeline nie ma `tofu init`, więc jeśli używasz reguł wymagających pełnego kontekstu providerów/modułów, rozważ dopisanie init (opcjonalnie). W większości repo modułowych `tflint` jest używany w trybie statycznym i to jest akceptowalne.

---

### 3) `✅ terraform-docs` (stage: `validate`)

**Cel:** Wymuszenie, aby dokumentacja modułu była aktualna, a README nie odbiegało od tego, co wygenerowałby `terraform-docs`.

Job działa jako „check”, a nie jako „formatter” – nie commitujemy zmian automatycznie, tylko **pipeline failuje, jeśli README wymaga aktualizacji**.

#### Logika joba

1. Tworzy kopię README:

   ```bash
   cp README.md README.md.bak
   ```

2. Generuje dokumentację jedną z dwóch metod:

   * jeśli istnieje `.terraform-docs.yml`:

     ```bash
     terraform-docs -c .terraform-docs.yml .
     ```

     (zwykle generuje/aktualizuje README wg konfiguracji)
   * jeśli nie ma `.terraform-docs.yml`:

     ```bash
     terraform-docs md . > README.md
     ```

     (nadpisuje README wygenerowanym markdownem)

3. Porównuje pliki:

   ```bash
   diff README.md README.md.bak
   ```

#### Jak interpretować wynik

* **Brak różnic** → README jest aktualne → job przechodzi.
* **Są różnice** → README wymaga aktualizacji → `diff` zwróci kod != 0 → job failuje.

#### Co ma zrobić developer, gdy job failuje

Lokalnie uruchomić ten sam mechanizm generowania dokumentacji:

* jeśli repo ma `.terraform-docs.yml`:

  ```bash
  terraform-docs -c .terraform-docs.yml .
  ```
* jeśli repo nie ma `.terraform-docs.yml`:

  ```bash
  terraform-docs md . > README.md
  ```

Następnie commit/push zmian w README.

---

## Zakres pipeline i intencja projektowa

Ten pipeline celowo:

* **nie wykonuje `tofu init`**
* **nie wykonuje `tofu validate`**
* **nie wykonuje `plan/apply`**
* **nie używa backendu state**

To jest właściwe dla modułów, bo:

* moduł nie jest „wdrażalny” sam w sobie,
* jego wartość to jakość, konwencje i dokumentacja,
* wdrożenia odbywają się w repozytoriach „root module / live infra”, a nie w repo modułu.

---

## Typowe problemy i diagnoza

### `terraform-docs` failuje przez brak README.md

Ten pipeline zakłada, że `README.md` istnieje. Jeśli moduł go nie ma, dodaj minimalny README (nawet pusty nagłówek) albo zmodyfikuj job tak, aby tworzył plik, gdy nie istnieje.

### `terraform-docs` nadpisuje README, a diff zawsze wykrywa zmiany

Najczęstsze powody:

* generacja zależy od wersji `terraform-docs` (różne wersje = inny output),
* README zawiera ręczne sekcje, które są kasowane przez domyślny tryb.

Rekomendacja: używać `.terraform-docs.yml` i/lub trybu z markerami (`<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->`) – wtedy `terraform-docs` aktualizuje tylko sekcję, a reszta README zostaje.

### `tflint` zgłasza błędy bez sensu dla modułu

Dodaj `.tflint.hcl` i skonfiguruj reguły modułowe (np. wyłączając te, które wymagają init/provider context).

---

## Referencja: definicje z pipeline

* `🕵 opentofu fmt`: format check
* `✅ tflint`: lint check
* `✅ terraform-docs`: README drift check (fail, jeśli README wymaga regeneracji)
