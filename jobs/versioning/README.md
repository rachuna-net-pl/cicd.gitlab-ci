# 📍 Publish Version (semantic release)

Job ten odpowiada za ustalenie wersji aplikacji/procesu na podstawie `Conventional Commits`[^1] z użyciem aplikacji `Semantic Release`[^2]

---
## Jak działa job?

Job analizuje całe repozytorium to znaczy: `branches`, `merge requests`, `commits`, `tags` i na tej postawie ustala nową wersję aplikacji.

> [!warning] Ważne
> Semantic Release wersjonuje tylko na event PUSH, nie działa w przebiegu, gdy otwarty jest Merge Request

> [!warning] Wersjonowanie typu snapshot
> W sytuacji gdy `Semantic Release` nie jest w stanie określić wersji, to proces zastosuje wersjonowanie tymczasowe oparte na ostatnim tagu z `DEFAULT BRANCH` (main) + `shortCommitSHA` (przykład: `v1.0.0-02c85fa9`)

[^1]: https://www.conventionalcommits.org/en/v1.0.0/
[^2]: https://semantic-release.gitbook.io/semantic-release/