# Referência das Ferramentas MCP — Azure DevOps (dev-annexus)

Este arquivo documenta os parâmetros exatos para cada ferramenta MCP usada pela skill
`azure-devops-create-pr`. Consulte esta referência se houver dúvida sobre campos ou comportamento.

---

## `repo_search_commits`

Busca commits de uma branch. Use para obter os commits da branch de origem que não estão na de destino.

### Parâmetros principais

| Campo              | Tipo     | Descrição                                                                 |
|--------------------|----------|---------------------------------------------------------------------------|
| `repositoryId`     | string   | Nome ou ID do repositório                                                 |
| `project`          | string   | Nome ou ID do projeto Azure DevOps                                        |
| `searchCriteria.itemVersion.version` | string | Branch de origem (ex: `feature/Inativar-cancelados`)   |
| `searchCriteria.compareVersion.version` | string | Branch de destino para diff (ex: `develop`)          |
| `searchCriteria.compareVersion.versionType` | string | `"branch"` (padrão)                                |
| `searchCriteria.itemVersion.versionType` | string | `"branch"` (padrão)                                   |
| `searchCriteria.top` | number | Máximo de commits a retornar (recomendado: `100`)                        |
| `searchCriteria.historyMode` | string | Use `"fullHistory"` para incluir merges, ou omita para linear    |

### Campos relevantes na resposta

```json
{
  "value": [
    {
      "commitId": "a1b2c3d4e5f6...",
      "comment": "feat: inativar associados cancelados",
      "author": {
        "name": "Thierry",
        "date": "2024-01-15T10:30:00Z"
      },
      "committer": { ... }
    }
  ],
  "count": 5
}
```

### Notas

- Se `compareVersion` não for suportado diretamente, tente buscar os commits da `sourceBranch`
  separadamente e os da `targetBranch`, depois filtre manualmente (commits em source que não aparecem em target).
- O campo `comment` pode ser multilinha; use apenas a primeira linha como mensagem do commit.

---

## `repo_create_pull_request`

Cria um novo Pull Request no repositório.

### Parâmetros principais

| Campo              | Tipo     | Obrigatório | Descrição                                                      |
|--------------------|----------|-------------|----------------------------------------------------------------|
| `repositoryId`     | string   | ✅           | Nome ou ID do repositório                                      |
| `project`          | string   | ✅           | Nome ou ID do projeto                                          |
| `title`            | string   | ✅           | Título do PR                                                   |
| `description`      | string   | ✅           | Descrição em Markdown                                          |
| `sourceRefName`    | string   | ✅           | Branch de origem com prefixo: `refs/heads/feature/minha-branch`|
| `targetRefName`    | string   | ✅           | Branch de destino com prefixo: `refs/heads/develop`           |
| `isDraft`          | boolean  | ❌           | `false` para PR pronto, `true` para rascunho                  |
| `reviewers`        | array    | ❌           | Lista de revisores `[{ "id": "guid" }]`                       |
| `workItemRefs`     | array    | ❌           | Work items vinculadas `[{ "id": "5053" }]` (alternativa ao step 5) |

### Campos relevantes na resposta

```json
{
  "pullRequestId": 142,
  "title": "Task #5053: Inativar associados cancelados",
  "status": "active",
  "sourceRefName": "refs/heads/feature/Inativar-cancelados",
  "targetRefName": "refs/heads/develop",
  "_links": {
    "web": {
      "href": "https://dev.azure.com/org/project/_git/repo/pullrequest/142"
    }
  }
}
```

### Notas

- **Sempre use o prefixo `refs/heads/`** nas branches — sem o prefixo, a API retorna erro 400.
- Se um PR já existir para a mesma combinação source/target, a API retorna erro. Use
  `repo_list_pull_requests` para verificar antes (ou trate o erro e exiba o PR existente).

---

## `wit_get_work_item`

Obtém informações completas de uma work item pelo ID.

### Parâmetros principais

| Campo              | Tipo     | Obrigatório | Descrição                             |
|--------------------|----------|-------------|---------------------------------------|
| `id`               | number   | ✅           | ID da work item (ex: `5053`)          |
| `project`          | string   | ❌           | Nome do projeto (melhora resolução)   |
| `$expand`          | string   | ❌           | `"all"` para trazer todos os campos   |

### Campos relevantes na resposta

```json
{
  "id": 5053,
  "fields": {
    "System.Title": "Inativar associados cancelados",
    "System.WorkItemType": "Task",
    "System.State": "Active",
    "System.Description": "<p>Implementar a inativação automática...</p>",
    "System.TeamProject": "Annexus"
  },
  "_links": {
    "html": {
      "href": "https://dev.azure.com/org/project/_workitems/edit/5053"
    }
  }
}
```

### Notas

- `System.Description` pode conter HTML — extraia apenas o texto plano para o resumo do PR.
- Se a task não existir, a API retorna 404. Trate graciosamente (continue sem o contexto).

---

## `wit_link_work_item_to_pull_request`

Vincula uma work item a um Pull Request criado.

### Parâmetros principais

| Campo              | Tipo     | Obrigatório | Descrição                                      |
|--------------------|----------|-------------|------------------------------------------------|
| `workItemId`       | number   | ✅           | ID da work item                                |
| `pullRequestId`    | number   | ✅           | ID do PR retornado por `repo_create_pull_request` |
| `repositoryId`     | string   | ✅           | Nome ou ID do repositório                      |
| `projectId`        | string   | ✅           | Nome ou ID do projeto                          |

### Resposta esperada

```json
{
  "id": 5053,
  "relations": [
    {
      "rel": "ArtifactLink",
      "url": "vstfs:///Git/PullRequestId/...",
      "attributes": {
        "name": "Pull Request"
      }
    }
  ]
}
```

### Notas

- Se falhar com erro de permissão ou `404`, oriente o usuário a vincular manualmente:
  no portal Azure DevOps → PR → aba "Work Items" → buscar pelo ID.

---

## `repo_list_pull_requests`

Lista PRs existentes. Use para verificar se já existe um PR aberto para a branch de origem.

### Parâmetros principais

| Campo              | Tipo     | Descrição                                                        |
|--------------------|----------|------------------------------------------------------------------|
| `repositoryId`     | string   | Nome ou ID do repositório                                        |
| `project`          | string   | Nome do projeto                                                  |
| `searchCriteria.sourceRefName` | string | Filtra por branch de origem (`refs/heads/feature/...`) |
| `searchCriteria.status` | string | `"active"` para PRs abertos                               |

---

## `pipelines_get_build_changes`

**Alternativa** para listar commits de uma build/pipeline associada à branch.
Use apenas se `repo_search_commits` não retornar resultados satisfatórios.

### Parâmetros principais

| Campo         | Tipo   | Descrição                           |
|---------------|--------|-------------------------------------|
| `buildId`     | number | ID da build (requer pesquisa prévia)|
| `project`     | string | Nome do projeto                     |
| `top`         | number | Máximo de registros                 |

---

## Mapeamento do Fluxo Completo

```
Usuário pede PR
       │
       ▼
repo_search_commits          ← commits exclusivos da source branch
       │
       ▼
wit_get_work_item             ← título, tipo, URL da task
       │
       ▼
[Compor título + descrição Markdown]
       │
       ▼
repo_create_pull_request      ← cria o PR, retorna pullRequestId
       │
       ▼
wit_link_work_item_to_pull_request  ← vincula task ao PR
       │
       ▼
[Exibir relatório final]
```
