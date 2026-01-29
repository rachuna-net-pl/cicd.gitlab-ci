# ▶️ Ansible Playbook

Ten pipeline służy do uruchamiania **playbooka Ansible** w repozytoriach wdrożeniowych. W obecnej konfiguracji wykonuje `playbooks/install.yml` z przekazanymi zmiennymi i limitem hostów/środowiska.

---
## Wymagania

* Repozytorium zawiera playbook: `playbooks/install.yml`.
* Istnieje inventory: `inventory/hosts.yml` (domyślna ścieżka).
* Jeżeli używasz ról z Galaxy, dostępny jest `requirements.yml`.
* Obraz kontenera zawiera Ansible:
  * `registry.rachuna-net.pl/pl.rachuna-net/containers/ansible:1.0.0`
* Dostępne są helpery:
  * `.helper_gitlab-ci.sh` (konfiguracja środowiska i dostępu do repo)
  * `.helper_readme.sh` (wskazanie dokumentacji po wykonaniu joba)

---
## Struktura pipeline

### Include: `ansible_init.sh.yml`

Pipeline dołącza lokalny plik:

```yaml
include:
  - local: "pipelines/ansible-playbook/ansible_init.sh.yml"
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
| `IMAGE_ANSIBLE`     | `registry.rachuna-net.pl/pl.rachuna-net/containers/ansible:1.0.0`          | Obraz kontenera z Ansible.                                          |
| `ANSIBLE_INVENTORY` | `inventory/hosts.yml`                                                      | Ścieżka do inventory (ustawiana w `.ansible_init.sh`).               |
| `ANSIBLE_VARS`      | *(brak)*                                                                   | Dodatkowe `--extra-vars` przekazywane do playbooka.                  |
| `ENVIRON`           | *(brak)*                                                                   | Limit hostów/środowiska przekazywany jako `--limit`.                 |
| `DOCS_MD_FILE_PATH` | `pipelines/ansible-playbook/README.md`                                     | Ścieżka do dokumentacji używana przez `.helper_readme.sh`.           |

---
## Joby: opis i zachowanie

### 1) `🧾 ansible-playbook` (stage: `deploy`)

**Cel:** wykonanie playbooka instalacyjnego.

**Komenda:**

```bash
ansible-playbook -i $ANSIBLE_INVENTORY playbooks/install.yml --extra-vars "$ANSIBLE_VARS" --limit $ENVIRON
```

**Kiedy się uruchamia:**

* automatycznie dla pipeline uruchomionego przez `schedule`,
* manualnie w pozostałych przypadkach.

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
* `🧾 ansible-playbook`: wykonanie `playbooks/install.yml`
