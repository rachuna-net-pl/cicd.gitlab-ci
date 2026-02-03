# Renovate pipeline

Ten pipeline uruchamia **Renovate** w GitLab CI i automatyzuje aktualizacje zależności w repozytorium (tworzy Merge Requesty z podbiciami wersji).

Repozytorium korzysta z pliku `renovate.json`, który rozszerza centralny preset:

```
gitlab>pl.rachuna-net/platform-policies//renovate/preset/gitlab-ci.json
```

Konfiguracje Renovate są utrzymywane w repozytorium polityk:

* `pl.rachuna-net/platform-policies` – katalog `renovate/preset/`

---
## Wymagania

* W repozytorium istnieje `renovate.json` z konfiguracją.
* Dostęp do obrazu:

  * `registry.gitlab.com/pl.rachuna-net/artifacts/containers/renovate:1.1.0`
* Token do API GitLab z uprawnieniami do tworzenia MR i pushowania gałęzi.
* Najlepiej uruchamiać pipeline jako **schedule** (np. codziennie w nocy).

---
## Zmienne

| Zmienna | Domyślna wartość | Opis |
| --- | --- | --- |
| `IMAGE_RENOVATE` | `registry.gitlab.com/pl.rachuna-net/artifacts/containers/renovate:1.1.0` | Obraz z Renovate. |
| `RENOVATE_TOKEN` | brak | Token do API GitLab (PAT lub token bota). |
| `RENOVATE_ENDPOINT` | brak | URL API GitLab (np. `https://gitlab.com/api/v4`). |
| `RENOVATE_PLATFORM` | `gitlab` | Docelowa platforma SCM. |
| `RENOVATE_LOG_LEVEL` | `info` | Poziom logowania. |

**Uwaga:** `RENOVATE_ENDPOINT` jest wymagane w przypadku self-managed GitLab.

---
## Przykładowy job

```yaml
renovate:
  stage: maintenance
  image: $IMAGE_RENOVATE
  variables:
    RENOVATE_PLATFORM: "gitlab"
    RENOVATE_ENDPOINT: "https://gitlab.com/api/v4"
    RENOVATE_LOG_LEVEL: "info"
  script:
    - renovate
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: on_success
```

**Co robi job:**

* uruchamia Renovate w kontekście bieżącego repozytorium,
* tworzy gałęzie z aktualizacjami,
* otwiera Merge Requesty zgodnie z presetem w `renovate.json`.

---
## Zalecenia operacyjne

* Uruchamiaj Renovate jako schedule, żeby nie obciążać pipeline'ów MR.
* Trzymaj token w `CI/CD Variables` jako masked/protected.
* Jeśli masz wiele repozytoriów, rozważ osobny token bota.

---
## Typowe problemy i diagnoza

### `401/403` w logach Renovate

* Sprawdź token (`RENOVATE_TOKEN`) i jego scope.
* Upewnij się, że token ma dostęp do repozytorium oraz do tworzenia MR.

### Renovate nie znajduje repozytorium

* Zweryfikuj `RENOVATE_ENDPOINT` oraz czy uruchamiasz job w poprawnym projekcie.
* Sprawdź, czy `renovate.json` jest w root repozytorium.

### MR się nie tworzą

* Upewnij się, że preset nie blokuje aktualizacji (np. reguły schedule).
* Sprawdź logi Renovate pod kątem filtrowania pakietów.
