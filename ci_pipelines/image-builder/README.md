# Image Builder

Pipeline służy do budowy obrazów kontenerów przy użyciu **Buildah**, zapisania obrazu jako artefaktu (`container-image.tar`), a następnie opublikowania go do GitLab Container Registry pod tagiem kandydata wydania (`$RELEASE_CANDIDATE_VERSION`). Dodatkowo pipeline oferuje:
- testy integracyjne obrazu (uruchomienie `container_test.sh`),
- skan podatności (Trivy) z rozróżnieniem progów HIGH/CRITICAL.

---
## 1. Integracja z projektem

W projekcie, który ma budować obraz, dodaj do `.gitlab-ci.yml`:

```yaml
include:
  - project: pl.rachuna-net/cicd
    ref: main
    file:
      - /gitlab-ci/pipelines/image-builder/.gitlab-ci.yml
```

Następnie:

1. Dodaj plik konfiguracyjny Buildah (domyślnie `config.yml`).
2. Upewnij się, że pipeline zawiera job `🕵 Set Version` ustawiający `RELEASE_CANDIDATE_VERSION`.
3. (Opcjonalnie) Dodaj `container_test.sh`, jeśli chcesz uruchamiać testy obrazu.
4. (Opcjonalnie) Dodaj job lintujący YAML (`🕵 YAML lint`) lub inne walidacje.

---
## 2. Jak działa pipeline (przepływ)

### Etap `build`

**`🚀 build container image`**

* Uruchamia `image-builder.sh` zdefiniowany jako `.image-builder.sh`.
* Tworzy obraz na podstawie `base_image` z `config.yml`.
* Konfiguruje obraz (labels/env/user/entrypoint/cmd), kopiuje pliki, instaluje pakiety.
* Eksportuje obraz jako artefakt: `container-image.tar`.

Artefakty:

* `container-image.tar`

### Etap `publish`

**`🌐 publish container image`**

* Importuje artefakt:

  * `buildah pull "docker-archive:container-image.tar"`
* Taguje obraz jako:

  * `$CI_REGISTRY_IMAGE:$RELEASE_CANDIDATE_VERSION`
* Loguje się do registry i publikuje obraz:

  * `buildah push "$FULL_IMAGE_NAME" "docker://$FULL_IMAGE_NAME"`

### Etap `integration-test`

**`🧪 test docker image`**

* Uruchamia job na zbudowanym obrazie (`image: $CI_REGISTRY_IMAGE:$RELEASE_CANDIDATE_VERSION`)
* Odpala testy:

  * `bash container_test.sh` (ścieżka kontrolowana przez `DOCKER_TEST_SCRIPT_PATH`)

**`🔬 trivy (dast)`**

* Skanuje obraz:

  * HIGH -> exit-code 0 (nie przerywa)
  * CRITICAL -> exit-code 1 (oznacza job jako failed)
* `allow_failure: true` powoduje, że pipeline może przejść nawet przy błędach skanu.

---
## 3. Struktura pliku `config.yml`

### Minimalny przykład

```yaml
base_image: debian:12-slim
```

### Przykład pełny (zalecany)

```yaml
base_image: debian:12-slim

before_build_commands:
  - echo "Preparing build context..."
  - ls -la

labels:
  org.opencontainers.image.source: "$CI_PROJECT_URL"
  org.opencontainers.image.revision: "$CI_COMMIT_SHA"

custom_repos:
  - name: hashicorp
    key_url: https://apt.releases.hashicorp.com/gpg
    key_path: /usr/share/keyrings/hashicorp-archive-keyring.gpg
    repo_entry: deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com noble main

packages:
  - curl
  - ca-certificates

copy:
  - source: ./bin/app
    destination: /usr/local/bin/app

env:
  APP_ENV: production

extra_commands:
  - chmod +x /usr/local/bin/app
  - /usr/local/bin/app --version || true

user:
  name: app
  shell: /bin/bash
  home: /home/app
  chown: /usr/local/bin/app
  uid: 1000
  enable_sudo: false

set_user: app
entrypoint: ["/usr/local/bin/app"]
cmd: ["--help"]
```

---
## 4. Semantyka pól `config.yml` (co dokładnie robią)

### `base_image` (wymagane)

Obraz bazowy przekazywany do `buildah from`.

### `before_build_commands` (opcjonalne)

Polecenia wykonywane **na runnerze (poza kontenerem)**, zanim Buildah utworzy kontener roboczy.
Zastosowanie: przygotowanie artefaktów, generowanie plików, diagnostyka kontekstu.

### `labels` (opcjonalne)

Mapowanie `key: value` używane jako:

* `buildah config --label key=value`

### `custom_repos` (opcjonalne, Debian/Ubuntu)

Lista repozytoriów dodawanych do `/etc/apt/sources.list.d/*.list`.
Obsługiwane pola:

* `name` – nazwa pliku listy (np. `hashicorp.list`)
* `key_url` – URL do klucza GPG (pobierany przez `curl`)
* `key_path` – ścieżka docelowa wewnątrz obrazu (np. `/usr/share/keyrings/...gpg`)
* `repo_entry` – wpis APT z `signed-by=...`

Uwagi:

* Ten mechanizm jest wprost pod `/etc/apt/...` i jest przeznaczony dla Debian/Ubuntu.
* Dla RHEL/Fedora/Alpine repozytoria należy konfigurować w `extra_commands`.

### `packages` (opcjonalne)

Lista pakietów instalowanych wewnątrz obrazu, zależnie od rodziny OS:

* Debian/Ubuntu: `apt-get install -y`
* RHEL/Fedora/CentOS: `dnf install -y`
* Alpine: `apk add --no-cache`

### `copy` (opcjonalne)

Lista operacji kopiowania:

* `buildah copy "$ctr" "$src" "$dst"`

Uwaga: `source` odnosi się do plików w repozytorium (workspace runnera).

### `env` (opcjonalne)

Zmienne środowiskowe obrazu:

* `buildah config --env KEY=VALUE`

### `extra_commands` (opcjonalne)

Polecenia wykonywane **wewnątrz kontenera** po `copy` i `packages`.
Wykonywane jako:

* `buildah run "$ctr" -- bash -c "$cmd"`

Uwaga: to zakłada dostępność `bash` w obrazie bazowym. Dla Alpine (gdzie standardem jest `sh`) należy:

* doinstalować `bash` w `packages`, albo
* zmodyfikować skrypt buildera do użycia `/bin/sh -c`.

### `user` / `set_user` (opcjonalne)

Tworzenie użytkownika i ustawienie go jako domyślnego w obrazie.

* `user.name` powoduje `useradd ...`
* `user.chown` wykona `chown -R user:user <path>`
* `set_user` ustawia finalnego użytkownika w obrazie (`buildah config --user`)

### `entrypoint` / `cmd` (opcjonalne)

Ustawiane przez:

* `buildah config --entrypoint`
* `buildah config --cmd`

Zalecane formatowanie jako lista (JSON array) w YAML.

---
## 5. Zmienne CI/CD (konfiguracja pipeline)

Najważniejsze zmienne używane przez pipeline:

* `BUILDAH_IMAGE` – obraz narzędziowy buildah (job + service)
* `TRIVY_IMAGE` – obraz narzędziowy trivy
* `BUILDAH_CONFIG_FILE_PATH` – ścieżka do `config.yml` (domyślnie `config.yml`)
* `RELEASE_CANDIDATE_VERSION` – tag publikacji (ustawiany przez `🕵 Set Version`)
* `BUILDAH_STORAGE_ROOT`, `BUILDAH_STORAGE_RUNROOT`, `STORAGE_DRIVER` – konfiguracja buildah storage (domyślnie `vfs` w `/tmp`)
* `DOCKER_TEST_SCRIPT_PATH` – ścieżka do skryptu testów (domyślnie `container_test.sh`)
* `DOCS_MD_FILE_PATH` – ścieżka do dokumentacji (jeśli używasz helpera do README)

---
## 6. Wymagania runnera i ograniczenia

### Runner / wykonanie

* Pipeline uruchamia Buildah z `--storage-driver vfs` i izolacją `chroot`.
* Wymaga możliwości uruchomienia kontenera narzędziowego oraz `services:` (buildah-dind/trivy-dind).
* Runner musi mieć dostęp do:

  * `$CI_REGISTRY_IMAGE` (push/pull),
  * `registry.rachuna-net.pl/...` (obrazy narzędziowe).

### Ograniczenia funkcjonalne wynikające ze skryptu

* Detekcja OS odbywa się przez `/etc/os-release` i `ID_LIKE/ID`.
* Instalacja `ca-certificates` jest obsłużona dla Debian/Ubuntu, RHEL/Fedora/CentOS oraz Alpine.
* `custom_repos` w obecnej implementacji zapisuje wpisy do `/etc/apt/sources.list.d/` (czyli jest Debian/Ubuntu-centrical).
* `extra_commands` wykonują `bash -c` (wymóg `bash`).

---
## 7. Typowe problemy i diagnostyka

### `bash: not found` (często dla Alpine)

Rozwiązania:

* dodaj `bash` do `packages`, albo
* przerób buildera, by wykonywał `sh -c` dla alpine.

### Błędy GPG/repozytoriów APT

* upewnij się, że `key_url` jest dostępny i zwraca klucz,
* używaj `signed-by=<key_path>` zgodnego z faktyczną ścieżką,
* w `before_build_commands` możesz dodać diagnostykę (np. `curl -I <key_url>`).

### `copy: no such file or directory`

* `source` odnosi się do ścieżki w repozytorium w job workspace,
* upewnij się, że plik jest generowany przed buildem (np. przez wcześniejszy stage lub `before_build_commands`).

### `permission denied` przy publikacji do registry

* sprawdź, czy `CI_REGISTRY_USER/CI_REGISTRY_PASSWORD` są dostępne w jobie,
* sprawdź uprawnienia projektu do pushowania do registry.

---
## 8. Wzorzec użycia w repo (szybki start)

1. Dodaj `config.yml` (minimum: `base_image`).
2. (Opcjonalnie) dodaj `container_test.sh`.
3. Uruchom pipeline; po sukcesie obraz jest dostępny jako:

   * `$CI_REGISTRY_IMAGE:$RELEASE_CANDIDATE_VERSION`

---
## 9. Rozszerzenia (rekomendowane)

* Dodaj OCI label-e (source/revision/created).
* Dodaj cache dla trivy (`.trivycache/` już jest).
* Rozważ osobny job „validate config.yml” (yq schema / conftest), jeśli chcesz twardo walidować strukturę.

## 10. Przykładowy plik

```yaml
base_image: debian:12-slim
before_build_commands:
  - apt-get update
labels:
  maintainer: dev@example.com
custom_repos:
  - name: my-repo
    key_url: https://example.com/repo.gpg
    key_path: /usr/share/keyrings/my-repo.gpg
    repo_entry: deb [signed-by=/usr/share/keyrings/my-repo.gpg] https://example.com stable main
packages:
  - curl
  - ca-certificates
copy:
  - source: ./bin/app
    destination: /usr/local/bin/app
env:
  APP_ENV: production
extra_commands:
  - chmod +x /usr/local/bin/app
user:
  name: app
  shell: /bin/bash
  home: /home/app
  chown: /usr/local/bin/app
set_user: app
entrypoint: ["/usr/local/bin/app"]
cmd: ["--help"]
```

---
### Obsługiwane pola

**Obowiązkowe:**

* `base_image` – obraz bazowy dla `buildah from`.

**Opcjonalne:**

* `before_build_commands` – polecenia wykonywane *poza kontenerem*, przed rozpoczęciem budowy.
* `labels` – etykiety przekazywane do `buildah config --label`.
* `custom_repos` – dodatkowe repozytoria APT (obsługiwane wyłącznie dla Debian/Ubuntu).
* `packages` – lista pakietów instalowanych automatycznie zależnie od OS (apt/dnf/apk).
* `copy` – pliki lub katalogi kopiowane do obrazu.
* `env` – zmienne środowiskowe obrazu.
* `extra_commands` – polecenia wykonywane *wewnątrz kontenera* po skopiowaniu plików.
* `user` – definicja użytkownika tworzonego podczas budowy.
* `set_user` – użytkownik ustawiony jako domyślny w obrazie.
* `entrypoint` / `cmd` – przekazywane do `buildah config`. Zalecane w formie listy.
