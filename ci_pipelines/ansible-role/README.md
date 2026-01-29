# 🧩 Ansible Role

Ten pipeline jest przeznaczony dla **repozytoriów z rolami Ansible**. Jego celem jest utrzymanie jakości i zgodności kodu roli poprzez uruchomienie `ansible-lint` w etapie walidacji.

---
## Wymagania

* Repozytorium zawiera poprawną strukturę roli Ansible (np. `tasks/`, `handlers/`, `defaults/`, `meta/`).
* Obraz kontenera zawiera narzędzia:
  * `ansible-lint`
* Dostępne są wspólne helpery:
  * `.helper_gitlab-ci.sh` (konfiguracja środowiska i dostępu do repo)
  * `.helper_readme.sh` (wskazanie dokumentacji po wykonaniu joba)

---
## Struktura pipeline

### Zmienne

| Zmienna            | Domyślna wartość                                                | Opis                                                                 |
| ------------------ | --------------------------------------------------------------- | -------------------------------------------------------------------- |
| `IMAGE_ANSIBLE`    | `registry.gitlab.com/pl.rachuna-net/artifacts/containers/ansible:v1.0.0-4558b65b` | Obraz narzędziowy z `ansible-lint`.                                    |
| `DOCS_MD_FILE_PATH`| `ci_pipelines/ansible-role/README.md`                           | Ścieżka do dokumentacji używana przez `.helper_readme.sh`.            |

---
## Joby: opis i zachowanie

### 1) `🧪 ansible-lint` (stage: `validate`)

**Cel:** Wykrywanie błędów i antywzorców w roli Ansible.

**Komenda:**

```bash
ansible-lint --force-color .
```

**Kiedy się uruchamia:** zawsze (na sukces), na każdym pipeline.

**Efekt:** pipeline failuje, jeżeli `ansible-lint` wykryje problemy.

---
## Konfiguracja `ansible-lint`

Jeśli chcesz dostosować reguły, dodaj do repozytorium plik konfiguracji, np.:

* `.ansible-lint`
* `pyproject.toml` (sekcja `tool.ansible-lint`)

`ansible-lint` automatycznie wykryje konfigurację w katalogu repozytorium.

---
## Typowe problemy i diagnoza

### `ansible-lint` zwraca błędy dla roli, która działa lokalnie

* Upewnij się, że struktura roli jest kompletna (np. istnieje `meta/main.yml`).
* Rozważ dodanie konfiguracji `.ansible-lint`, aby wyłączyć wybrane reguły lub dostosować standardy.

### Brak informacji o dokumentacji po jobie

* Sprawdź, czy zmienna `DOCS_MD_FILE_PATH` jest ustawiona.
* W tym pipeline jest ustawiona na `ci_pipelines/ansible-role/README.md`.

---
## Referencja: definicje z pipeline

* `🧪 ansible-lint`: lint roli Ansible
