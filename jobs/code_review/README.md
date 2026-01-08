# 🕵 AI Code Review (Codex)

Job `🕵 AI Code Review` realizuje automatyczny przegląd kodu dla Merge Requestów z użyciem **Codex CLI**. Mechanizm:

1. pobiera konfigurację autoryzacji i tokeny z Vault,
2. pobiera diff MR z GitLab API,
3. buduje „pakiet wejściowy” do review (manifest + profil reviewera + prompt + diff),
4. uruchamia `codex exec` w trybie sandbox,
5. publikuje wynik jako komentarz w MR (GitLab Note),
6. zapisuje artefakty (diff, input, output, logi) do debugowania.

Job jest uruchamiany tylko w pipeline MR (`merge_request_event`) oraz tylko na self-hosted GitLab (blokada dla gitlab.com).

---
## Zakres i cel

Celem joba jest dostarczenie **spójnego, powtarzalnego i audytowalnego** code review, w tym:

* wykrywanie typowych błędów (bezpieczeństwo, logiczne, styl),
* sugestie ulepszeń i refaktoryzacji,
* zgodność z wytycznymi projektowymi opisanymi w manifestach i promptach,
* minimalizacja „szumu” (review jest wykonywane tylko gdy MR spełnia warunki wejściowe).

---
## Definicje include

Job korzysta z trzech include’ów, które dostarczają treści i konfigurację dla review:

```yaml
include:
  - local: "jobs/code_review/.codex/manifests/001-manifest.sh.yml"
  - local: "jobs/code_review/.codex/profiles/001-reviewer.sh.yml"
  - local: "jobs/code_review/.codex/prompts/001-code-review.sh.yml"
```

### Co zapewniają include’y

| Plik                     | Rola                                                                      |
| ------------------------ | ------------------------------------------------------------------------- |
| `001-manifest.sh.yml`    | „Manifest” – zasady projektu, standardy, konwencje, priorytety review.    |
| `001-reviewer.sh.yml`    | „Profil reviewera” – perspektywa, sposób oceny, kryteria, styl feedbacku. |
| `001-code-review.sh.yml` | Prompt zadania – format odpowiedzi i oczekiwania co do outputu.           |

W praktyce te referencje są dołączane do `.codex/` w trakcie `before_script` i następnie sklejane do `code-review-input.md`.

---
## Obraz i wymagania runtime

Job działa na obrazie:

* `registry.rachuna-net.pl/pl.rachuna-net/containers/codex:1.0.0`

Obraz powinien zawierać:

* `codex` CLI,
* `curl`, `jq`, podstawowe coreutils (`wc`, `head`, itp.).

---
## Zmienne joba

### Zmienne kontrolujące model i limity

| Zmienna                    | Domyślna wartość | Znaczenie                                                      |
| -------------------------- | ---------------: | -------------------------------------------------------------- |
| `CODEX_MODEL`              |    `gpt-5-codex` | Model używany przez `codex exec`.                              |
| `AI_REVIEW_MAX_DIFF_CHARS` |         `120000` | Maksymalny rozmiar diff przekazywany do AI (w znakach).        |
| `AI_REVIEW_MAX_NOTE_CHARS` |          `20000` | Maksymalny rozmiar komentarza publikowanego do MR (w znakach). |

### Zmienne środowiskowe i sekrety

Job wymaga, aby w CI były dostępne:

| Zmienna                | Wymagana | Opis                                            |
| ---------------------- | -------: | ----------------------------------------------- |
| `VAULT_ADDR`           |      Tak | URL Vault (np. `https://vault.rachuna-net.pl`). |
| `VAULT_TOKEN`          |      Tak | Token do pobrania sekretów dla Codex/GitLab.    |
| `CI_API_V4_URL`        |      Tak | Standardowa zmienna GitLab CI.                  |
| `CI_PROJECT_ID`        |      Tak | Standardowa zmienna GitLab CI.                  |
| `CI_MERGE_REQUEST_IID` |      Tak | Dostępna tylko w pipeline MR.                   |

---
## Inicjalizacja (before_script)

### 1) Helper CI

Job używa wspólnego helpera:

```yaml
- !reference [.helper_gitlab-ci.sh]
```

### 2) Konfiguracja Code Reviewer (Vault → auth + token)

Job pobiera sekrety z Vault:

* `AUTH_CONFIG` → zapis do `$HOME/.codex/auth.json`
* `GITLAB_TOKEN` → export do środowiska joba

Fragment logiki:

```bash
VAULT_SECRETS=$(curl -sS -H "X-Vault-Token: $VAULT_TOKEN" "$VAULT_ADDR/v1/kv-gitlab/data/pl.rachuna-net/auth/codex")
echo $VAULT_SECRETS | jq -r '.data.data.AUTH_CONFIG' > $HOME/.codex/auth.json
export GITLAB_TOKEN=$(echo $VAULT_SECRETS | jq -r '.data.data.GITLAB_TOKEN')
```

**Efekt:** job ma komplet danych do:

* autoryzacji Codex (auth.json),
* wywołań GitLab API (GITLAB_TOKEN).

### 3) Zapis treści manifest/profile/prompt

W `before_script` wykonywane są referencje:

```yaml
- !reference [.001-manifest.sh]
- !reference [.001-reviewer.sh]
- !reference [.001-code-review.sh]
```

Wynikiem jest przygotowana struktura `.codex/` (prompts/docs itd.) wykorzystywana później w `script`.

---
## Logika działania (script)

### 1) Warunek „wymagany reviewer”

Job pobiera szczegóły MR:

```bash
MR_JSON=$(curl -sS -H "PRIVATE-TOKEN: $GITLAB_TOKEN" "$CI_API_V4_URL/projects/$CI_PROJECT_ID/merge_requests/$CI_MERGE_REQUEST_IID")
REVIEWER_LOGINS=$(echo "$MR_JSON" | jq -r '.reviewers[].username')
```

Następnie sprawdza, czy w MR jest reviewer `aideveloper`:

```bash
if ! echo "$REVIEWER_LOGINS" | grep -Eq "aideveloper"; then
  echo "⛔ Ten MR nie ma wymaganego reviewera – pomijam AI Code Review."
  exit 0
fi
```

**Znaczenie:** AI review uruchamia się wyłącznie, gdy MR ma przypisanego reviewera `aideveloper`. To ogranicza uruchomienia do przypadków świadomie „zamówionych”.

### 2) Guard: pipeline musi być MR

Jeśli nie ma `CI_MERGE_REQUEST_IID`, job kończy się bez błędu:

```bash
if [ -z "$CI_MERGE_REQUEST_IID" ]; then
  echo "Brak CI_MERGE_REQUEST_IID – nie jesteśmy w pipeline MR. Kończę."
  exit 0
fi
```

### 3) Pobranie diff i budowa `diff.md`

Job pobiera zmiany MR:

```bash
CHANGES_JSON=$(curl -sS -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "$CI_API_V4_URL/projects/$CI_PROJECT_ID/merge_requests/$CI_MERGE_REQUEST_IID/changes")
```

Następnie generuje `.codex/docs/diff.md` w formacie markdown:

* każdy plik jako sekcja `### path`
* zawartość diff w blokach ` ```diff `

Jeśli diff przekracza limit `AI_REVIEW_MAX_DIFF_CHARS`, jest ucinany.

### 4) Budowa wejścia do review: `code-review-input.md`

Job skleja jeden plik wejściowy zawierający:

1. Manifest
2. Profil reviewera
3. Prompt zadania
4. Diff MR

To zapewnia spójny kontekst i deterministykę promptu.

### 5) Uruchomienie Codex CLI

```bash
codex exec \
  --model "${CODEX_MODEL}" \
  --sandbox read-only \
  --cd "$CI_PROJECT_DIR" \
  --output-last-message "$REVIEW_OUTPUT" \
  - < "$REVIEW_INPUT" 2> "$CODEX_LOG"
```

Kluczowe parametry:

* `--sandbox read-only` – AI nie może modyfikować repo w trakcie review.
* `--output-last-message` – zapisuje finalną odpowiedź AI do pliku.
* `2> "$CODEX_LOG"` – logi wykonania do artefaktów.

### 6) Publikacja komentarza do MR (GitLab Note)

Job publikuje wynik jako komentarz do MR:

* ucinanie do `AI_REVIEW_MAX_NOTE_CHARS`
* POST do endpointu notes:

```bash
NOTE_URL="$CI_API_V4_URL/projects/$CI_PROJECT_ID/merge_requests/$CI_MERGE_REQUEST_IID/notes"
curl -X POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" --data-urlencode "body=${BODY_TRUNC}" "$NOTE_URL"
```

Jeśli GitLab zwróci HTTP >= 400, job kończy się błędem.

---

## Artefakty

Job zawsze publikuje artefakty:

```yaml
artifacts:
  when: always
  paths:
    - .codex/docs/
  expire_in: "1 day"
```

W artefaktach znajdują się typowo:

* `diff.md` – diff przekazany do AI (ew. ucięty)
* `diff-trunc.md` – (jeśli użyte ucięcie)
* `code-review-input.md` – pełen input do Codex
* `code-review-output.md` – odpowiedź AI
* `codex-exec.log` – logi wykonania Codex

**Cel artefaktów:** audyt i debug w przypadku braku komentarza lub błędów integracji.

---
## Reguły uruchomienia (rules)

```yaml
rules:
  - if: $CI_SERVER_HOST == "gitlab.com"
    when: never
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    when: on_success
  - when: never
```

Interpretacja:

* job nigdy nie uruchomi się na `gitlab.com`
* uruchamia się tylko w pipeline typu **Merge Request**
* nie działa na push pipelines ani schedules

---
## Bezpieczeństwo i uprawnienia

1. **GITLAB_TOKEN** pobierany z Vault musi mieć uprawnienia do:

   * odczytu MR i jego changes,
   * dodawania komentarzy (notes) do MR.

2. **VAULT_TOKEN** musi mieć uprawnienia do:

   * odczytu ścieżki `kv-gitlab/data/pl.rachuna-net/auth/codex`.

3. Zalecane jest stosowanie:

   * zmiennych **masked** i **protected**,
   * ograniczeń joba do trusted runnerów (tag `gitlab-rnp` lub podobne).

---
## Typowe problemy i diagnoza

### Job „nic nie robi” i kończy się exit 0

Najczęstsze powody:

* MR nie ma reviewera `aideveloper`,
* pipeline nie jest MR (`CI_MERGE_REQUEST_IID` puste).

### Brak komentarza w MR

Sprawdź artefakt `codex-exec.log` oraz odpowiedź API `/notes` w logach joba (HTTP_CODE).

### Błąd 401/403 z GitLab API

* token z Vault jest niepoprawny albo nie ma uprawnień do notes,
* MR jest w projekcie/grupie z innymi restrykcjami.

### Output jest ucięty

* limit `AI_REVIEW_MAX_NOTE_CHARS` (20k) zadziałał – pełna treść będzie w artefakcie `code-review-output.md`.

---
## Konwencja użycia w MR

Aby uruchomić AI Review:

1. dodaj reviewera `aideveloper` do MR,
2. upewnij się, że pipeline jest MR (`merge_request_event`),
3. job uruchomi się automatycznie i opublikuje komentarz.


## Prompty i dokumenty sterujące (manifest / profile / prompt)

Job `🕵 AI Code Review` nie opiera się na jednym „gołym” promptcie. Zamiast tego wykorzystuje **warstwowy zestaw dokumentów**, które determinują zachowanie modelu. Każdy z dokumentów jest versionowany w repo i ładowany w czasie joba do katalogu `.codex/`.

### 1) Struktura katalogu `.codex/` i „single source of truth”

Wszystkie dokumenty, które sterują działaniem `codex-cli`, muszą znajdować się w:

* `.codex/manifests/` – manifesty projektu (zasady ogólne, tryby pracy, priorytety)
* `.codex/profiles/` – profile ról (np. reviewer, architekt, devops)
* `.codex/prompts/` – prompty zadań (np. code review, refactor request, security review)
* `.codex/docs/` – artefakty operacyjne joba (diff, input, output, logi)

Zgodnie z manifestem: `.codex/` jest **jedynym źródłem prawdy** dla zachowania `codex-cli` w CI. Zmiany w `.codex/` podlegają wersjonowaniu i Code Review tak samo jak kod.

---

## 2) Składanie promptu: co faktycznie trafia do modelu

W trakcie joba budowany jest plik wejściowy:

* `.codex/docs/code-review-input.md`

Jest on generowany poprzez sklejenie w tej kolejności:

1. Manifest (`.codex/manifests/*`)
2. Profil reviewera (`.codex/profiles/*`)
3. Prompt zadania (`.codex/prompts/*`)
4. Diff MR (`.codex/docs/diff.md`)

To podejście ma dwie konsekwencje projektowe:

* **deterministyczny kontekst**: model zawsze widzi te same zasady i strukturę wejścia,
* **przewidywalność zmian**: modyfikacja jednego pliku w `.codex/` wprost wpływa na zachowanie review w kolejnych pipeline.

---

## 3) Hierarchia reguł i rozstrzyganie konfliktów

W `001-code-review.md` została zdefiniowana jawna hierarchia:

1. `.codex/manifests/*` – manifest główny projektu
2. `.codex/profiles/*` – profil aktywny (tu: REVIEWER)
3. `.codex/prompts/*` – prompty ogólne
4. `.codex/prompts/001-code-review.md` – prompt konkretnego zadania

Oznacza to, że:

* manifest determinuje „konstytucję” przeglądu (jak myśleć, co jest priorytetem),
* profil determinuje rolę i ograniczenia (statyczna analiza, bez uruchamiania kodu),
* prompt zadania determinuje format, publikację oraz reguły zgłaszania problemów.

Jeżeli wystąpi sprzeczność, obowiązuje kolejność powyżej.

---

## 4) Manifest projektu (`.001-manifest.sh` → `.codex/manifests/001-manifest.md`)

Manifest definiuje nadrzędny tryb pracy:

* podstawą jest diff MR,
* ale model ma prawo analizować **kontekst gałęzi źródłowej** (snapshot source branch),
* priorytetem jest wykrywanie **braków i niespójności** w repo (call-site’y, testy, konfiguracje, dokumentacja).

Istotne elementy manifestu:

* rozszerzony zakres oceny: diff + powiązane pliki + testy + konfiguracja + dokumentacja,
* wskazanie typowych „missing pieces” (np. nowy kontrakt bez aktualizacji call-site’ów),
* formalizacja roli `.codex/` jako źródła prawdy i zakaz ignorowania tych dokumentów.

W praktyce manifest przenosi review z „linii w diffie” na „diff + konsekwencje w source branch”.

---

## 5) Profil reviewera (`.001-reviewer.sh` → `.codex/profiles/001-reviewer.md`)

Profil REVIEWER jest rolą „Senior Developer / Code Reviewer” w trybie **analizy statycznej**:

* brak uruchamiania kodu, testów, narzędzi CLI,
* ocena wyłącznie na podstawie: diff + snapshot source branch + artefakty z CI (jeśli są).

Profil wprowadza:

* checklisty (logika, błędy, dane wejściowe, zależności, bezpieczeństwo, wydajność),
* twarde reguły klasyfikacji problemów (BLOCKER/MAJOR/MINOR/NIT),
* granice kompetencji (np. brak audytu infrastruktury).

To zabezpiecza proces przed „review w ciemno” i przed próbą interpretacji runtime bez danych.

---

## 6) Prompt Code Review (`.001-code-review.sh` → `.codex/prompts/001-code-review.md`)

Ten prompt jest „wykonawczy” — określa:

### a) Warunki wejściowe z CI

Wymienia minimalny zestaw zmiennych (MR IID, SHA, source/target branch, pipeline id itd.) i wymaga:

* jeśli czegoś brakuje → publikuj **jeden komentarz BLOCKER** o błędzie konfiguracji.

### b) Zakazy i ograniczenia

Powtarza i wzmacnia zasady:

* brak uruchamiania czegokolwiek,
* brak modyfikacji repo,
* zasady bezpieczeństwa tokenów (nie logować, nie publikować).

### c) Anty-halucynacje i ograniczenie severity

Prompt wprowadza twarde reguły:

* nie wolno zgłaszać problemów bez potwierdzenia w kodzie (diff lub source branch),
* BLOCKER wymaga cytatu problematycznej linii kodu,
* zakaz „fałszywych blockerów” typu „pipeline może się wysypać” bez dowodu w kodzie.

To jest kluczowe, bo wymusza dowodowość i obniża ryzyko „przeszacowania” wagi uwag.

### d) Format komentarza do MR

Prompt definiuje obowiązkową strukturę komentarza:

* Mocne strony
* Obszary do poprawy
* Blockery
* Lista problemów w ujednoliconym formacie
* Status rekomendowany (READY / WARUNKOWO / NIE MERGOWAĆ)

Dodatkowo wprowadza marker sekcji:

* `=== MR_COMMENT_START ===` / `=== MR_COMMENT_END ===`

który wskazuje modelowi, że ma wygenerować **wyłącznie treść komentarza**, bez instrukcji, separatorów i metadanych.

---

## 7) Jak rozwijać prompty i wersjonować zmiany

Rekomendowana praktyka dla Twojego układu:

* każda istotna zmiana w zachowaniu = nowy numer pliku (np. `002-code-review.md`),
* nie nadpisywać wstecznie semantyki bez powodu (łatwiejszy audit i rollback),
* manifest i profile powinny ewoluować wolniej niż prompt zadania (stabilność zasad),
* każda zmiana `.codex/` podlega normalnemu Code Review.

---

## 8) Konsekwencje dla działania joba (praktyczne)

1. Review nie jest „tylko o diffie” — model może zaglądać do plików w source branch, ale nadal musi wiązać uwagi z MR.
2. Review jest bezpieczniejsze (dowodowe) dzięki regułom:

   * anty-halucynacyjnym,
   * wymaganiu cytatu dla BLOCKER,
   * zakazowi blockerów „na wyczucie”.
3. Format komentarza jest zawsze spójny, co ułatwia czytanie i decyzję „merge / nie merge”.
