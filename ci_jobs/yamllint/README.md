# 🕵 YAML lint

Job ten waliduje wszystkie pliki YAML w repozytorium z użyciem narzędzia `yamllint`.

---
## Jak działa job?

- Pobiera konfigurację lintowania z pliku `gitlab-ci/ci_jobs/yamllint/.yamllint.yml` z bieżącego brancha (wymaga poprawnego `GITLAB_TOKEN` z dostępem do repozytorium).
- Uruchamia `yamllint .`, więc sprawdza cały projekt łącznie z katalogami CI.
- Odpala się na branchach oraz w Merge Requestach tylko wtedy, gdy względem `main` zmieniono pliki `*.yml` lub `*.yaml`.
- Nie uruchamia się dla tagów.
