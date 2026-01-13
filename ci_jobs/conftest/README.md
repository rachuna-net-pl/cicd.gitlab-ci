# 🔬 Validate files (conftest)

Job uruchamia **Conftest** i waliduje wskazane pliki względem polityk (OPA/Rego) pobieranych z zewnętrznego repozytorium. Wyniki są zapisywane jako raport JUnit.

---
## Jak działa job?

1. **Pobiera polityki** z repozytorium wskazanego przez zmienne CI (`.policies/`).
2. **Uruchamia `conftest test`** dla plików wskazanych w `PLATFORM_POLICIES_FILES`.
3. **Wykonuje testy w dwóch formatach**:
   * `table` — czytelny output w logach,
   * `junit` — zapis do `conftest-junit.xml` (artefakt).
4. **Zwraca błąd**, jeżeli wykryto naruszenia polityk.

Jeżeli plik z polityką (`.policies/$PLATFORM_POLICIES_PATH/$PLATFORM_POLICIES_NAMESPACE`) **nie istnieje**, job kończy się komunikatem „Nie ma nic do roboty”.

---
## Zmienne joba

| Zmienna                             | Domyślna wartość | Znaczenie |
|-------------------------------------|------------------|----------|
| `IMAGE_CONFTEST`                    | *(w CI)*         | Obraz z narzędziem `conftest`. |
| `REPOSITORY_PLATFORM_POLICIES_BRANCH` | `feat/init`     | Branch repozytorium z politykami. |
| `PLATFORM_POLICIES_PATH`            | `conftest/policies` | Ścieżka do polityk w repozytorium. |
| `PLATFORM_POLICIES_NAMESPACE`       | *(w CI)*         | Namespace polityk, uruchamiany przez `conftest`. |
| `PLATFORM_POLICIES_FILES`           | *(w CI)*         | Pliki/ścieżki do walidacji. |

---
## Artefakty

Job publikuje raport:

* `conftest-junit.xml` — wynik testów w formacie JUnit.

---
## Reguły uruchomienia

W definicji joba ustawiono:

```yaml
rules:
  - when: never
```

Oznacza to, że job **nie uruchamia się automatycznie**. Jeśli ma być używany, należy nadpisać reguły w include lub w pipeline.

---
## Uwagi

* Zmienna `DOCS_MD_FILE_PATH` w jobie wskazuje na `ci_jobs/confest/README.md` — wygląda na literówkę w ścieżce (`confest` vs `conftest`).
