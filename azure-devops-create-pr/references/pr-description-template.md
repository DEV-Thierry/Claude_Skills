# Template de Descrição do Pull Request

Use este template como base para gerar a descrição do PR no Passo 3 da skill.
Substitua todos os placeholders `{...}` com os dados reais coletados.

---

## Template Markdown (copie e preencha)

```markdown
## 📋 Descrição

* ✅ **O que foi feito neste PR:** {RESUMO_DAS_ALTERAÇÕES}

* 🔗 **Task que o PR resolve:** [#{WORK_ITEM_ID} — {WORK_ITEM_TITLE}]({WORK_ITEM_URL})

## 🌐 Ambiente de Homologação

| Serviço | URL |
|---------|-----|
| Portal | [{PORTAL_URL}]({PORTAL_URL}) |
| Admin  | [{ADMIN_URL}]({ADMIN_URL}) |

* 📝 **Commits incluídos:**

| Hash | Mensagem |
|------|----------|
{LINHAS_DE_COMMITS}

```

---

## Como Preencher Cada Campo

### `{RESUMO_DAS_ALTERAÇÕES}`

Gere 2 a 4 frases em português descrevendo as alterações.

**Fontes para o resumo:**
1. Mensagens dos commits (principal fonte)
2. Título da work item
3. Descrição da work item (se disponível, remova o HTML)

**Exemplos de bons resumos:**

> Implementada a inativação automática de associados com status cancelado na tabela
> `dbo.SituacaoAssociados`. Adicionada verificação de data de cancelamento para evitar
> inativações indevidas. Criado endpoint de auditoria para consultar as inativações realizadas.

> Corrigido o cálculo de juros em parcelas retroativas para contratos pré-2020.
> O bug causava divergência no valor total exibido no extrato do associado.

> Adicionada listagem paginada de associados filtrada por categoria e status,
> com suporte a ordenação por nome e data de cadastro.

**Evite:**
- ❌ "Foram feitas alterações no código."
- ❌ "Commits adicionados à branch."
- ❌ Reproduzir literalmente as mensagens de commit sem síntese.

---

### `{WORK_ITEM_ID}`, `{WORK_ITEM_TITLE}`, `{WORK_ITEM_URL}`

Obtidos diretamente da resposta do `wit_get_work_item`:
- ID: campo `id`
- Título: `fields["System.Title"]`
- URL: `_links.html.href`

Se a work item não foi encontrada:
```markdown
* 🔗 **Task que o PR resolve:** #{WORK_ITEM_ID} *(work item não encontrada)*
```

---

---

### `{PORTAL_URL}` e `{ADMIN_URL}`

Obtidos no **Passo 3** da skill, lendo o log do step `"Exibir URLs do ambiente"` (job `PrintURLs`, stage `Summary`) da pipeline de feature.

O log contém linhas no formato:
```
Portal Frontend: https://feature-minha-branch.up.railway.app
Admin Frontend:  https://feature-minha-branch-admin.up.railway.app
```

**Se as URLs não estiverem disponíveis** (pipeline não executou, falhou, ou variáveis não resolvidas), **omita toda a seção** `## 🌐 Ambiente de Homologação` da descrição.

---

### `{LINHAS_DE_COMMITS}`

Gere uma linha por commit no formato de tabela Markdown:

```markdown
| `a1b2c3d` | feat: inativar associados cancelados |
| `e4f5a6b` | fix: corrigir query de busca por status |
| `c7d8e9f` | chore: adicionar migration de inativação |
```

**Regras:**
- Use os primeiros 7 caracteres do `commitId` (short hash)
- Use apenas a primeira linha da mensagem do commit
- Limite a mensagem a ~80 caracteres; truncate com `...` se necessário
- Ordene do mais recente para o mais antigo (ordem natural da API)

---

## Exemplos de PR Completo

### Exemplo 1 — Feature de Inativação

**Título:** `Task #5053: Inativar associados cancelados`

**Descrição:**
```markdown
## 📋 Descrição

* ✅ **O que foi feito neste PR:** Implementada a inativação automática de associados
  com situação "cancelado" na tabela `dbo.SituacaoAssociados`. Adicionada regra de negócio
  para verificar se o cancelamento tem mais de 30 dias antes de inativar. Criado job agendado
  para execução diária da rotina de inativação.

* 🔗 **Task que o PR resolve:** [#5053 — Inativar associados cancelados](https://dev.azure.com/annexus/Projeto/_workitems/edit/5053)

## 🌐 Ambiente de Homologação

| Serviço | URL |
|---------|-----|
| Portal | [https://feature-inativar-cancelados.up.railway.app](https://feature-inativar-cancelados.up.railway.app) |
| Admin  | [https://feature-inativar-cancelados-admin.up.railway.app](https://feature-inativar-cancelados-admin.up.railway.app) |

* 📝 **Commits incluídos:**

| Hash | Mensagem |
|------|----------|
| `f3a1b2c` | feat: adicionar job de inativação de associados cancelados |
| `d4e5f6a` | feat: implementar regra dos 30 dias para inativação |
| `b7c8d9e` | feat: criar endpoint de auditoria de inativações |
| `a1b2c3d` | chore: migration - adicionar coluna DataInativacao em SituacaoAssociados |

---
*PR criado automaticamente pela skill azure-devops-create-pr* 🤖
```

---

### Exemplo 2 — Bug Fix

**Título:** `Bug #4892: Corrigir cálculo de mensalidade em parcelas retroativas`

**Descrição:**
```markdown
## 📋 Descrição

* ✅ **O que foi feito neste PR:** Corrigido erro no cálculo de juros compostos aplicados
  a parcelas retroativas de associados com contrato anterior a 2020. A divergência causava
  exibição incorreta do valor total no extrato. Adicionados testes unitários para os casos
  de borda identificados durante a investigação.

* 🔗 **Task que o PR resolve:** [#4892 — Corrigir cálculo de mensalidade em parcelas retroativas](https://dev.azure.com/annexus/Projeto/_workitems/edit/4892)

## 🌐 Ambiente de Homologação

| Serviço | URL |
|---------|-----|
| Portal | [https://feature-corrigir-calculo-mensalidade.up.railway.app](https://feature-corrigir-calculo-mensalidade.up.railway.app) |
| Admin  | [https://feature-corrigir-calculo-mensalidade-admin.up.railway.app](https://feature-corrigir-calculo-mensalidade-admin.up.railway.app) |

* 📝 **Commits incluídos:**

| Hash | Mensagem |
|------|----------|
| `9f8e7d6` | fix: corrigir cálculo de juros em parcelas com data retroativa |
| `5c4b3a2` | test: adicionar testes para parcelas pré-2020 |
| `1a2b3c4` | refactor: extrair cálculo de juros para service dedicado |

---
*PR criado automaticamente pela skill azure-devops-create-pr* 🤖
```
