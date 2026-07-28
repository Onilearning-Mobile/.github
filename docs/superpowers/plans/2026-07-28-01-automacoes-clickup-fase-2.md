# Automações ClickUp — Fase 2 (draft/review/branch/stale-PR)

**Closure:** full

## Context

O workflow reutilizável `Onilearning-Mobile/.github/.github/workflows/clickup-sync.yml` já move a
task pra "code review" quando o PR abre e pra "concluído" quando mergeia. Esta fase fecha 2 gaps
reais encontrados na auditoria (PR draft dispara `opened` cedo demais; PR fechado sem merge não
reverte nada) e adiciona 4 automações novas: reação a review do PR, comentário na task com link do
PR, transição pra "em andamento" na criação da branch, e lembrete de PR parado em code review.

**Nota da auditoria:** o bug "falta `repo:` no caller" descrito antes da escrita deste plano **já
foi corrigido em `origin/dev`** dos 4 repos consumidores (commit `fix(ci): pass repo name to
ClickUp sync for Sistema field inference`) — confirmado lendo o arquivo fresco pós-`fetch`. Não
faz parte do escopo abaixo.

## Scope

- In: `Onilearning-Mobile/.github` (workflow reutilizável + novo workflow de cron) e os 4 callers
  (`oniapp`, `oniapp-api`, `seja-api`, `seja-moderation`).
- Out: `oniapp-admin` e `seja-app` — não têm caller workflow hoje; ligar a automação nesses repos
  é uma decisão de wiring separada, não pedida pelo usuário.
- Out: mudar como o mapeamento `GH_TO_CLICKUP` (login GitHub → user id ClickUp) é mantido — reusa
  como está, hardcoded.

## Architecture decisions

- Statuses da lista "Dev. Mobile" (`901715099907`), confirmados via `clickup_get_list` ao vivo:
  `para fazer`, `em andamento`, `code review`, `em alteração/teste`, `pendente`, `concluído`,
  `descartado`, `entregue`.
- PR draft não deve mover a task — só quando o PR é aberto já pronto OU quando sai de draft
  (`ready_for_review`). Isso exige um novo input `pr_draft` no `workflow_call` e adicionar
  `ready_for_review` aos `types` do trigger `pull_request` nos 4 callers.
- PR fechado sem merge reverte a task pai pra "em andamento" (não "para fazer" — trabalho foi
  feito) e move a subtask de Code Review pra "descartado".
- Review do PR (`pull_request_review`, `submitted`): `changes_requested` move a task pra
  "em alteração/teste" + comenta + notifica chat; `approved` não muda status (só comenta +
  notifica) — o status só muda de fato no merge.
- Push de uma branch nova (`push`, `github.event.created == true`) com padrão `ONIL-<n>` no nome
  move a task pra "em andamento". Sem criar subtask, sem notificar chat (evita ruído por commit) e
  sem comentar PR (ainda não existe PR nesse ponto).
- Novo workflow standalone `stale-pr-reminder.yml` (não é `workflow_call` — roda direto no repo
  `.github` num cron) avisa no chat quando uma subtask "Code Review — PR #" passa de 2 dias no
  status "code review". **Requer secret `CLICKUP_API_TOKEN` configurado manualmente como Actions
  secret no repo `Onilearning-Mobile/.github`** (hoje só existe nos 4 consumidores, herdado via
  `secrets: inherit` nas chamadas `workflow_call` — esse workflow novo não é chamado via
  `workflow_call`, então não herda nada; é uma ação fora do git que o usuário precisa fazer no
  GitHub, settings → secrets → actions).
- IDs reaproveitados do arquivo atual (não mudam): `LIST_ID=901715099907`,
  `FIELD_EQUIPE=bf0b9d1c-7c65-410a-a74e-ed75009003f5`,
  `OPTION_MOBILE=a3145da6-72d6-4324-9413-33b1060d4231`,
  `FIELD_SISTEMA=2424f2c6-fe24-4b4f-a766-b8c7cb17a157`,
  `OPTION_SEJA=0cd17024-51b1-47a5-8f27-e190fe63726c`,
  `OPTION_ONIAPP=4eefe1db-bbc9-4c91-b154-0e54d923db6d`, chat workspace `90171059720` canal
  `6-901715099907-8`.

## Project rules that apply

- Golden Rule #1/#2: implementar em branch/worktree isolado, sequencial (não paralelo entre
  repos).
- Golden Rule #3: um commit por mudança lógica — cada repo recebe seus próprios commits; o
  workflow central recebe 1 commit por job/feature novo (5 commits) em vez de um único commit
  gigante.
- Golden Rule #8: nenhum débito novo sem avisar — o requisito do secret manual do
  `stale-pr-reminder.yml` é reportado ao usuário, não assumido silenciosamente.
- Golden Rule #13: nenhum trailer de co-autoria nos commits.

## Patterns to follow

**Padrão de extração de task ID** (já existe 2x no arquivo, reaproveitar literalmente nos novos
jobs `pr-closed-unmerged`, `pr-review`; versão simplificada — só branch, sem fallback de corpo —
no job `branch-created`):

```bash
normalize() {
  local id="$1"
  if [[ "$id" =~ ^[A-Za-z]+-[0-9]+$ ]]; then
    echo "${id^^}"
  else
    echo "$id"
  fi
}

TASK_ID=""
shopt -s nocasematch
if [[ "$PR_BRANCH" =~ (ONIL-[0-9]+) ]]; then
  TASK_ID=$(normalize "${BASH_REMATCH[1]}")
elif [[ "$PR_BODY" =~ \*\*ClickUp:\*\*[[:space:]]*\<?(https://app\.clickup\.com/t/)?([A-Za-z0-9]+-[0-9]+|[A-Za-z0-9]+)\>? ]]; then
  TASK_ID=$(normalize "${BASH_REMATCH[2]}")
fi
shopt -u nocasematch

if [[ -n "$TASK_ID" ]]; then
  echo "task_id=${TASK_ID}" >> "$GITHUB_OUTPUT"
  echo "found=true" >> "$GITHUB_OUTPUT"
  if [[ "$TASK_ID" =~ ^[A-Za-z]+-[0-9]+$ ]]; then
    echo "qs=custom_task_ids=true&team_id=90171059720" >> "$GITHUB_OUTPUT"
  else
    echo "qs=" >> "$GITHUB_OUTPUT"
  fi
else
  echo "found=false" >> "$GITHUB_OUTPUT"
fi
```

**Padrão de notificação no chat** (reaproveitar literalmente, só troca `CONTENT`):

```bash
CONTENT="<mensagem>"
PAYLOAD=$(jq -n --arg content "$CONTENT" '{type: "message", content: $content, content_format: "text/md"}')
for attempt in 1 2 3; do
  RESPONSE=$(curl -s -X POST "https://api.clickup.com/api/v3/workspaces/90171059720/chat/channels/6-901715099907-8/messages" \
    -H "Authorization: ${CLICKUP_API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "$PAYLOAD")
  echo "$RESPONSE" | jq -e '.id' >/dev/null 2>&1 && break
  sleep 2
done
if ! echo "$RESPONSE" | jq -e '.id' >/dev/null 2>&1; then
  echo "::warning::Falha ao notificar chat após 3 tentativas: $RESPONSE"
fi
```

> Dispatch note: envie a cada worker o cabeçalho acima (Context, Scope, Architecture decisions,
> Project rules, Patterns to follow) + só a seção da(s) sua(s) task(s).

## Tasks

### Task 1 — Novo input `pr_draft` + corrigir job `pr-opened` pra draft/ready_for_review

**Tier:** T3-Full
**Depends on:** none
**File:** `.github/workflows/clickup-sync.yml`
**What:** No bloco `on.workflow_call.inputs` (depois de `merged_by`, antes de `secrets:`),
adicionar:

```yaml
      pr_draft:
        required: false
        type: boolean
        default: false
```

Trocar a linha `if:` do job `pr-opened` (hoje `if: inputs.action == 'opened' && inputs.base_ref == 'dev'`) por:

```yaml
    if: inputs.base_ref == 'dev' && ((inputs.action == 'opened' && inputs.pr_draft != true) || inputs.action == 'ready_for_review')
```

Nada mais nesse job muda — os steps existentes (`extract`, `move_status`, criar subtask, notificar
chat, avisar PR) continuam iguais.
**Verify:** `yamllint .github/workflows/clickup-sync.yml` (ou, se não tiver yamllint instalado,
`python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync.yml'))"`) →
exit 0, sem erro de sintaxe.

### Task 2 — Novo job `pr-closed-unmerged`

**Tier:** T3-Full
**Depends on:** Task 1
**File:** `.github/workflows/clickup-sync.yml`
**What:** Adicionar um job novo `pr-closed-unmerged` (irmão de `pr-opened`/`pr-merged`, mesmo
nível de indentação sob `jobs:`), com condição
`if: inputs.action == 'closed' && inputs.merged == false && inputs.base_ref == 'dev'`,
`runs-on: ubuntu-latest`, `permissions: pull-requests: write`. Steps, em ordem:

1. `Extrair ID da task do ClickUp (branch primeiro, descrição como fallback)` — usa o padrão de
   extração de "Patterns to follow" literalmente (env `PR_BRANCH: ${{ inputs.pr_branch }}`,
   `PR_BODY: ${{ inputs.pr_body }}`).
2. `Mover task de volta para "em andamento"` (id: `move_status`, `if: steps.extract.outputs.found == 'true'`) — mesmo curl `PUT` das outras transições de status, trocando o body pra
   `-d '{"status": "em andamento"}'`.
3. `Descartar subtask de Code Review` (`if: steps.extract.outputs.found == 'true'`) — busca a task
   com `include_subtasks=true`, filtra subtask cujo nome contém `PR #${PR_NUMBER}:` (mesmo `jq`
   usado no job `pr-merged` linha ~294), e se achar, `PUT` status `"descartado"` nela. Não precisa
   assignee aqui (diferente do job `pr-merged`).
4. `Notificar no chat "Dev. Mobile"` (`if: steps.extract.outputs.found == 'true' && steps.move_status.outputs.success == 'true'`) — usa o padrão de notificação de "Patterns to follow" com:
   `CONTENT="❌ **[${PR_TITLE}](${PR_URL})** foi fechado sem merge — [task](https://app.clickup.com/t/${TASK_ID}) voltou para \"em andamento\""`.
5. `Avisar no PR — link do ClickUp não encontrado` (`if: steps.extract.outputs.found == 'false'`) —
   mesmo `actions/github-script@v7` dos outros jobs, mensagem
   `'⚠️ PR fechado sem merge, mas nenhuma task do ClickUp foi encontrada (branch/descrição sem ID).'`.

**Verify:** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync.yml'))"` → exit 0.

### Task 3 — Novo job `pr-review` (reação a `pull_request_review`)

**Tier:** T3-Full
**Depends on:** Task 1
**File:** `.github/workflows/clickup-sync.yml`
**What:** Adicionar 2 novos inputs no `workflow_call` (junto com `pr_draft` da Task 1):

```yaml
      review_state:
        required: false
        type: string
        default: ""
      reviewer:
        required: false
        type: string
        default: ""
```

Adicionar job novo `pr-review`, condição
`if: inputs.action == 'review' && inputs.base_ref == 'dev' && (inputs.review_state == 'changes_requested' || inputs.review_state == 'approved')`,
`runs-on: ubuntu-latest`, `permissions: pull-requests: write`. Steps:

1. Extração de task ID — mesmo padrão.
2. `Mover task para "em alteração/teste" (mudanças solicitadas)` — só roda se
   `steps.extract.outputs.found == 'true' && inputs.review_state == 'changes_requested'` — curl
   `PUT` status `"em alteração/teste"`.
3. `Comentar na task do ClickUp` (`if: steps.extract.outputs.found == 'true'`) — `POST
   /api/v2/task/${TASK_ID}/comment${QS:+?$QS}` com `{"comment_text": "<texto>"}`; texto é
   `"🔁 ${REVIEWER} solicitou mudanças no PR: ${PR_URL}"` se `changes_requested`, senão
   `"✅ ${REVIEWER} aprovou o PR: ${PR_URL}"`.
4. `Notificar no chat "Dev. Mobile"` (`if: steps.extract.outputs.found == 'true'`) — padrão de
   notificação; conteúdo:
   - `changes_requested`: `"🔁 **${REVIEWER}** solicitou mudanças em **[${PR_TITLE}](${PR_URL})** — [task](https://app.clickup.com/t/${TASK_ID}) voltou para \"em alteração/teste\""`
   - `approved`: `"✅ **${REVIEWER}** aprovou **[${PR_TITLE}](${PR_URL})** — [task](https://app.clickup.com/t/${TASK_ID}) pronta pra merge"`
5. `Avisar no PR — link do ClickUp não encontrado` (`if: steps.extract.outputs.found == 'false'`) —
   mesmo padrão `github-script`, mensagem
   `'⚠️ Review registrada, mas nenhuma task do ClickUp foi encontrada (branch/descrição sem ID).'`.

**Verify:** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync.yml'))"` → exit 0.

### Task 4 — Novo job `branch-created` (push cria branch → "em andamento")

**Tier:** T2-Standard
**Depends on:** none
**File:** `.github/workflows/clickup-sync.yml`
**What:** Job novo `branch-created`, condição `if: inputs.action == 'branch_created'`,
`runs-on: ubuntu-latest` (sem bloco `permissions:` — não comenta em PR). Steps:

1. `Extrair ID da task do ClickUp a partir do nome da branch` — versão **simplificada** do padrão
   (só branch, sem fallback de corpo, já que não existe PR ainda):

```bash
normalize() {
  local id="$1"
  if [[ "$id" =~ ^[A-Za-z]+-[0-9]+$ ]]; then
    echo "${id^^}"
  else
    echo "$id"
  fi
}

TASK_ID=""
shopt -s nocasematch
if [[ "$PR_BRANCH" =~ (ONIL-[0-9]+) ]]; then
  TASK_ID=$(normalize "${BASH_REMATCH[1]}")
fi
shopt -u nocasematch

if [[ -n "$TASK_ID" ]]; then
  echo "task_id=${TASK_ID}" >> "$GITHUB_OUTPUT"
  echo "found=true" >> "$GITHUB_OUTPUT"
  echo "qs=custom_task_ids=true&team_id=90171059720" >> "$GITHUB_OUTPUT"
else
  echo "found=false" >> "$GITHUB_OUTPUT"
fi
```

   (env: `PR_BRANCH: ${{ inputs.pr_branch }}`)
2. `Mover task para "em andamento"` (`if: steps.extract.outputs.found == 'true'`) — curl `PUT`
   status `"em andamento"` (sem step de sucesso/falha separado, sem chat, sem comentário — ver
   "Architecture decisions").

**Verify:** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync.yml'))"` → exit 0.

### Task 5 — Comentário na task do ClickUp com link do PR (jobs `pr-opened` e `pr-merged`)

**Tier:** T2-Standard
**Depends on:** Task 1
**File:** `.github/workflows/clickup-sync.yml`
**What:** Em **cada** um dos jobs `pr-opened` e `pr-merged`, adicionar um step novo
`Comentar na task do ClickUp` logo depois do step `Notificar no chat "Dev. Mobile"`, mesma
condição de `if` que o step de chat correspondente.

Em `pr-opened`:
```bash
COMMENT=$(printf '🔍 PR [%s](%s) (`%s`) entrou em code review.' "$PR_TITLE" "$PR_URL" "$PR_BRANCH")
PAYLOAD=$(jq -n --arg text "$COMMENT" '{comment_text: $text}')
curl -s -X POST "https://api.clickup.com/api/v2/task/${TASK_ID}/comment${QS:+?$QS}" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" > /dev/null
```
(env: `TASK_ID`, `QS`, `PR_TITLE: ${{ inputs.pr_title }}`, `PR_URL: ${{ inputs.pr_url }}`,
`PR_BRANCH: ${{ inputs.pr_branch }}`)

Em `pr-merged`:
```bash
COMMENT=$(printf '✅ PR [%s](%s) foi mergeado por **%s**.' "$PR_TITLE" "$PR_URL" "$MERGED_BY")
PAYLOAD=$(jq -n --arg text "$COMMENT" '{comment_text: $text}')
curl -s -X POST "https://api.clickup.com/api/v2/task/${TASK_ID}/comment${QS:+?$QS}" \
  -H "Authorization: ${CLICKUP_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" > /dev/null
```
(env: `TASK_ID`, `QS`, `PR_TITLE: ${{ inputs.pr_title }}`, `PR_URL: ${{ inputs.pr_url }}`,
`MERGED_BY: ${{ inputs.merged_by }}`)

**Verify:** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync.yml'))"` → exit 0.

### Task 6 — Novo workflow `stale-pr-reminder.yml`

**Tier:** T3-Full
**Depends on:** none
**File:** `.github/workflows/stale-pr-reminder.yml` (novo arquivo)
**What:** Workflow standalone (não `workflow_call`):

```yaml
name: Lembrete de PR parado em Code Review

on:
  schedule:
    - cron: '0 13 * * 1-5'
  workflow_dispatch:

jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - name: Buscar subtasks de Code Review paradas há mais de 2 dias
        shell: bash
        env:
          CLICKUP_API_TOKEN: ${{ secrets.CLICKUP_API_TOKEN }}
        run: |
          LIST_ID="901715099907"
          NOW_MS=$(($(date +%s) * 1000))
          THRESHOLD_MS=$((2 * 24 * 60 * 60 * 1000))

          RESPONSE=$(curl -s "https://api.clickup.com/api/v2/list/${LIST_ID}/task?subtasks=true&include_closed=false&statuses[]=code%20review" \
            -H "Authorization: ${CLICKUP_API_TOKEN}")

          echo "$RESPONSE" | jq -c '.tasks[]? | select(.name | startswith("Code Review — PR #"))' | while read -r task; do
            NAME=$(echo "$task" | jq -r '.name')
            URL=$(echo "$task" | jq -r '.url')
            CREATED=$(echo "$task" | jq -r '.date_created')
            AGE=$((NOW_MS - CREATED))

            if [ "$AGE" -gt "$THRESHOLD_MS" ]; then
              DAYS=$((AGE / 86400000))
              CONTENT="⏰ **${NAME}** está em code review há ${DAYS} dia(s) — [ver task](${URL})"
              PAYLOAD=$(jq -n --arg content "$CONTENT" '{type: "message", content: $content, content_format: "text/md"}')
              curl -s -X POST "https://api.clickup.com/api/v3/workspaces/90171059720/chat/channels/6-901715099907-8/messages" \
                -H "Authorization: ${CLICKUP_API_TOKEN}" \
                -H "Content-Type: application/json" \
                -d "$PAYLOAD" > /dev/null
            fi
          done
```

`0 13 * * 1-5` = 13:00 UTC (10:00 BRT) em dias úteis. `workflow_dispatch` permite disparo manual
pra testar.

**Verify:** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/stale-pr-reminder.yml'))"` → exit 0.

### Task 7 — Atualizar os 4 callers (triggers + inputs novos)

**Tier:** T2-Standard
**Depends on:** Task 1, Task 3, Task 4 (precisa dos inputs/jobs novos já existirem no central
antes do caller apontar pra eles — mas como o caller usa `@main`, só importa a ordem de merge, não
a ordem de commit local)
**File:** `.github/workflows/clickup-sync-caller.yml` em **cada um** dos 4 repos: `oniapp`,
`oniapp-api`, `seja-api`, `seja-moderation` — mesmo conteúdo exato nos 4, cada um seu próprio
commit no seu próprio repo/worktree.
**What:** Substituir o arquivo inteiro por:

```yaml
name: ClickUp Sync

on:
  pull_request:
    types: [opened, closed, ready_for_review]
  pull_request_review:
    types: [submitted]
  push:
    branches-ignore: [main, dev]

permissions:
  pull-requests: write

jobs:
  sync:
    if: github.event_name == 'pull_request'
    uses: Onilearning-Mobile/.github/.github/workflows/clickup-sync.yml@main
    with:
      action: ${{ github.event.action }}
      base_ref: ${{ github.event.pull_request.base.ref }}
      pr_number: ${{ github.event.pull_request.number }}
      pr_title: ${{ github.event.pull_request.title }}
      pr_url: ${{ github.event.pull_request.html_url }}
      pr_branch: ${{ github.event.pull_request.head.ref }}
      pr_author: ${{ github.event.pull_request.user.login }}
      pr_body: ${{ github.event.pull_request.body }}
      repo: ${{ github.repository }}
      pr_draft: ${{ github.event.pull_request.draft }}
      merged: ${{ github.event.pull_request.merged }}
      merged_by: ${{ github.event.pull_request.merged_by.login }}
    secrets: inherit

  review-sync:
    if: github.event_name == 'pull_request_review'
    uses: Onilearning-Mobile/.github/.github/workflows/clickup-sync.yml@main
    with:
      action: review
      base_ref: ${{ github.event.pull_request.base.ref }}
      pr_number: ${{ github.event.pull_request.number }}
      pr_title: ${{ github.event.pull_request.title }}
      pr_url: ${{ github.event.pull_request.html_url }}
      pr_branch: ${{ github.event.pull_request.head.ref }}
      pr_author: ${{ github.event.pull_request.user.login }}
      pr_body: ${{ github.event.pull_request.body }}
      repo: ${{ github.repository }}
      review_state: ${{ github.event.review.state }}
      reviewer: ${{ github.event.review.user.login }}
    secrets: inherit

  branch-created:
    if: github.event_name == 'push' && github.event.created == true
    uses: Onilearning-Mobile/.github/.github/workflows/clickup-sync.yml@main
    with:
      action: branch_created
      base_ref: ""
      pr_number: 0
      pr_title: ${{ github.ref_name }}
      pr_url: ""
      pr_branch: ${{ github.ref_name }}
      pr_author: ${{ github.actor }}
      repo: ${{ github.repository }}
    secrets: inherit
```

Todos os 4 repos já têm `repo: ${{ github.repository }}` no job `sync` atual (bug antigo já
corrigido em `origin/dev` antes deste plano) — o conteúdo acima preserva isso e só acrescenta
`pr_draft`, o job `review-sync` e o job `branch-created`.
**Verify (por repo):** `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/clickup-sync-caller.yml'))"` → exit 0.

### Task 8 — Verificação final (fechamento)

**Tier:** T1-Nano
**Depends on:** Tasks 1–7
**File:** N/A (verificação, não edita)
**What:** Rodar a validação YAML de todos os 5 arquivos alterados/criados de uma vez.
**Verify:** para cada um dos 5 repos/worktrees, `python3 -c "import yaml,sys; yaml.safe_load(open('<arquivo>'))"` → exit 0 em todos; nenhum arquivo com erro de sintaxe.

## Out of scope (explicit)

- Configurar o secret `CLICKUP_API_TOKEN` no repo `Onilearning-Mobile/.github` — ação manual no
  GitHub (Settings → Secrets → Actions), fora do alcance de commits/git. Reportar ao usuário como
  pendência pós-plano.
- Push das branches e abertura de PRs nos 5 repos — ação visível/compartilhada, pedir confirmação
  ao usuário antes de executar (não presumida pelo "implementar tudo").
- `oniapp-admin`, `seja-app` — sem caller workflow hoje, fora de escopo.
