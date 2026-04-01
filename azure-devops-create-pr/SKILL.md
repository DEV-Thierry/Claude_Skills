---
name: azure-devops-create-pr
description: >
  Automatiza a criação de Pull Requests no Azure DevOps usando o MCP azure-devops (conexão dev-annexus).
  Use esta skill sempre que o usuário pedir para "criar um PR", "abrir pull request", "criar pull request
  para a branch X", "PR da feature/X para develop", "cria PR", "abre PR", "gera PR", ou qualquer variação
  pedindo a criação de um Pull Request no Azure DevOps — mesmo que o usuário não mencione todos os detalhes
  (branch de origem, destino ou task). A skill busca os commits da branch, obtém informações da work item
  vinculada e cria o PR com descrição formatada em Markdown, vinculando a task automaticamente.
compatibility: "Requer MCP azure-devops (conexão dev-annexus) com as ferramentas: repo_create_pull_request, repo_search_commits, wit_get_work_item, wit_link_work_item_to_pull_request, repo_list_pull_requests"
---

# Criar Pull Request no Azure DevOps

## Visão Geral

Esta skill executa o fluxo completo de criação de PR no Azure DevOps:

1. **Coleta parâmetros** — branch de origem, destino e ID da task
2. **Busca commits** — diferença entre as branches (commits exclusivos da source)
3. **Obtém contexto da task** — título, descrição e tipo da work item
4. **Cria o PR** — com descrição Markdown estruturada e título derivado da task
5. **Vincula a work item** — ao PR recém-criado automaticamente

---

## Parâmetros de Entrada

Extraia do `$ARGUMENTS` ou da mensagem do usuário:

| Parâmetro         | Exemplo                          | Obrigatório |
|-------------------|----------------------------------|-------------|
| `sourceBranch`    | `feature/Inativar-cancelados`    | ✅           |
| `targetBranch`    | `develop`                        | ✅ (default: `develop`) |
| `workItemId`      | `5053`                           | ✅           |
| `repositoryId`    | nome ou ID do repositório        | ❌ (pergunte se não souber) |
| `projectName`     | nome do projeto Azure DevOps     | ❌ (pergunte se não souber) |

> Se algum parâmetro obrigatório estiver faltando, **pergunte ao usuário antes de continuar**.
> O `targetBranch` pode ser omitido; use `develop` como padrão.

---

## Passo 1 — Buscar Commits da Branch de Origem

Use `repo_search_commits` para listar os commits presentes em `sourceBranch` que **não** estão em `targetBranch`.

```
Ferramenta: repo_search_commits
Parâmetros relevantes:
  - branch: <sourceBranch>
  - itemVersion / compareVersion: <targetBranch>  (diferença entre branches)
  - top: 100  (máximo razoável)
```

**Extraia de cada commit:**
- `commitId` (primeiros 7 caracteres — short hash)
- `comment` (mensagem do commit, primeira linha)
- `author.name` e `author.date`

Se **nenhum commit** for encontrado, informe o usuário e **não crie o PR**.

> Consulte `references/mcp-tools.md` → seção "repo_search_commits" para detalhes dos parâmetros exatos.

---

## Passo 2 — Obter Informações da Work Item

Use `wit_get_work_item` para buscar título, tipo e descrição da task.

```
Ferramenta: wit_get_work_item
Parâmetros:
  - id: <workItemId>
```

**Extraia:**
- `fields["System.Title"]` → título da task
- `fields["System.WorkItemType"]` → tipo (Task, User Story, Bug, etc.)
- `fields["System.Description"]` → descrição (opcional, para enriquecer o resumo)
- `_links.html.href` → URL direta para a task no Azure DevOps

Se a work item **não for encontrada**, continue com o ID apenas (sem título) e avise o usuário.

---

## Passo 3 — Compor o Pull Request

### Título do PR

Use o padrão:
```
{WorkItemType} #{workItemId}: {WorkItemTitle}
```

Exemplos:
- `Task #5053: Inativar associados cancelados`
- `Bug #4892: Corrigir cálculo de mensalidade em parcelas`
- `User Story #4100: Listagem de associados por categoria`

Se não tiver título da work item: `feat: {sourceBranch sem prefixo}`

### Descrição do PR (Markdown)

Monte o template abaixo, preenchendo cada seção com os dados coletados:

```markdown
## 📋 Descrição

* ✅ **O que foi feito neste PR:** {resumo gerado a partir dos commits e contexto da task — 2 a 4 linhas descritivas, em português, explicando as alterações realizadas}

* 🔗 **Task que o PR resolve:** [#{workItemId} — {WorkItemTitle}]({urlDaTask})

* 📝 **Commits incluídos:**

| Hash | Mensagem |
|------|----------|
{linhas de commits no formato: | `{shortHash}` | {mensagem} |}

---
*PR criado automaticamente pela skill azure-devops-create-pr* 🤖
```

#### Como gerar o resumo de "O que foi feito"

Analise as mensagens dos commits e o título/descrição da work item para produzir **2 a 4 frases** explicando:
- O que foi implementado / corrigido / alterado
- Quais entidades ou tabelas foram impactadas (se inferível dos commits)
- O objetivo funcional da mudança

Exemplos de bom resumo:
> Implementada a inativação automática de associados com status "cancelado" na tabela `dbo.SituacaoAssociados`. Adicionada lógica de verificação de data de cancelamento antes da inativação. Criado endpoint de consulta para auditoria das inativações realizadas.

Exemplos de resumo ruim (evitar):
> "Commits foram adicionados." / "Vários arquivos foram modificados."

---

## Passo 4 — Criar o Pull Request

Use `repo_create_pull_request` com os dados compostos.

```
Ferramenta: repo_create_pull_request
Parâmetros:
  - title: <título gerado no Passo 3>
  - description: <descrição Markdown do Passo 3>
  - sourceBranch: refs/heads/<sourceBranch>
  - targetBranch: refs/heads/<targetBranch>
  - repositoryId: <repositoryId>
  - isDraft: false
```

**Capture da resposta:**
- `pullRequestId` — necessário para vincular a work item no próximo passo
- `url` ou `_links.web.href` — link direto para o PR

> ⚠️ As branches devem ser passadas com prefixo `refs/heads/`. Consulte `references/mcp-tools.md`
> → seção "repo_create_pull_request" para todos os campos disponíveis.

---

## Passo 5 — Vincular a Work Item ao PR

Use `wit_link_work_item_to_pull_request` para criar o vínculo oficial.

```
Ferramenta: wit_link_work_item_to_pull_request
Parâmetros:
  - workItemId: <workItemId>
  - pullRequestId: <pullRequestId do Passo 4>
  - repositoryId: <repositoryId>
  - projectId: <projectName ou projectId>
```

Se a ferramenta falhar (ex: permissão), **continue** e avise o usuário para vincular manualmente pelo portal.

---

## Passo 6 — Relatório Final

Após concluir todos os passos, exiba o sumário:

```
✅ Pull Request criado com sucesso!

📌 Título:     {título do PR}
🔀 Branch:     {sourceBranch} → {targetBranch}
🆔 PR ID:      #{pullRequestId}
🔗 Link:       {url do PR}
🎫 Task:       #{workItemId} — {WorkItemTitle}
📦 Commits:    {total de commits incluídos}

🔗 Vínculo com work item: ✅ criado  |  ⚠️ falhou (vincule manualmente)
```

---

## Tratamento de Erros

| Situação                                   | Ação                                                                    |
|--------------------------------------------|-------------------------------------------------------------------------|
| Branch de origem não encontrada            | Informe o usuário e aborte                                              |
| Nenhum commit diferente encontrado         | Informe que as branches estão sincronizadas e aborte                    |
| Work item não encontrada                   | Continue sem contexto da task; avise o usuário                         |
| Falha ao criar o PR (ex: PR duplicado)     | Verifique se já existe um PR aberto para a mesma branch; mostre o link |
| Falha ao vincular a work item              | Continue e oriente o usuário a vincular manualmente                     |

---

## Referências

- Detalhes dos parâmetros de cada ferramenta MCP: `references/mcp-tools.md`
- Template de descrição completo: `references/pr-description-template.md`
