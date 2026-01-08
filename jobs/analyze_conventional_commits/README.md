# 🔍 Analyze Conventional Commits

Job ten sprawdza, czy utworzone commits są zgodne z standardem `Conventional Commits`[^1] poprzez walidacje ich za pomocą wyrażenia regularnego. Jeśli commits nie są zgodne z tym standardem to job zakończy się z statusem `❌ failed`

---
## Jak działa job?

Job analizuje wyniki z polecania, który zwraca listę commitów i na jej podstawie dokonywana jest analiza pod kątem zgodności z regexp.

```bash
CI_DEFAULT_BRANCH="main"
git --no-pager log origin/$CI_DEFAULT_BRANCH..HEAD --pretty=format:"%s"
```

---
## Jak naprawić błąd walidacji
- Popraw tytuł commita: `git commit --amend` (dla ostatniego) lub `git rebase -i` (dla wielu), zmień message na zgodny ze wzorcem, a następnie `git push --force-with-lease`.

---
## Przykłady

> [!important] Opis polecenia
> To polecenie wyświetla listę commitów z lokalnej gałęzi (HEAD) względem domyślnego brancha - `main`, czyli badany jest tylko przyrost.

> [!tip] Przykłady poprawnych commitów
> - ✔️ feat(api): Zmiana biznesowa
> - ✔️ feat: Zmiana biznesowa
> - ✔️ feat!: Zmiana biznesowa

> [!caution] Przykłady nieporawnych commitów
> - ❌ Zmiana biznesowa
> - ❌ feat : Zmiana biznesowa
> - ❌ feat :Zmiana biznesowa
> - ❌ !feat :Zmiana biznesowa
> - ❌ Feat :Zmiana biznesowa
> - ❌ feature :Zmiana biznesowa

[^1]: https://www.conventionalcommits.org/en/v1.0.0/