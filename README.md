# Onilearning-Mobile/.github

Reusable workflows e arquivos de configuração compartilhados entre os repositórios da organização.

- `.github/workflows/clickup-sync.yml` — sincroniza status de tasks do ClickUp com o ciclo de vida de PRs (abertura → "code review" + subtask; merge → "concluído" + notificação no chat "Dev. Mobile"). Chamado pelos repos via `uses:`.
- `CONTRIBUTING.md` / `PULL_REQUEST_TEMPLATE.md` — herdados automaticamente por qualquer repo do org sem os próprios arquivos.
