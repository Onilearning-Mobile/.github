# Convenções de contribuição — Onilearning-Mobile

## Vincular o PR a uma task do ClickUp

Todo PR aberto contra a branch `dev` deve preencher o campo `**ClickUp:**` no
template do PR com o link ou o ID da task correspondente:

    **ClickUp:** 86e29ax99

ou

    **ClickUp:** https://app.clickup.com/t/86e29ax99

Por quê: uma automação (`clickup-sync.yml`, reusable workflow deste repositório)
lê esse campo e:

- **Ao abrir o PR:** move a task para o status "code review" e cria uma
  subtask de revisão (prioridade alta) vinculada ao PR, além de notificar
  o canal "Dev. Mobile" no chat do ClickUp.
- **Ao mergear o PR em `dev`:** move a task e a subtask de revisão para
  "concluído", atribui a subtask a quem fez o merge, e notifica o canal
  novamente.

Sem esse campo preenchido, a automação comenta um aviso no PR e nada é
movido no ClickUp — o merge não é bloqueado.
