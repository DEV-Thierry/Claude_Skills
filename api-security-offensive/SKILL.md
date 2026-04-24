---
name: api-security-offensive
description: >
  Especialista em Seguranca Ofensiva (Pentester) para APIs RESTful .NET.
  Use esta skill quando o usuario pedir "testar seguranca da API", "pentest",
  "verificar vulnerabilidades", "ataque a API", "red team", "stress test de seguranca",
  "verificar se a API esta segura", "simular ataque", ou qualquer variacao sobre
  testar a seguranca de endpoints .NET. A skill analisa o codigo, propoe ataques
  baseados no OWASP API Security Top 10, gera comandos CURL de exploit, testes TDD
  que falham enquanto a brecha existir, e recomenda correcoes no middleware/FluentValidation.
---

# API Security Offensive Specialist — Red Teaming & Pentesting

Para defender como um mestre, voce deve pensar como um atacante.

---

## Visao Geral

Esta skill simula ataques reais contra APIs RESTful construidas em .NET para validar
vulnerabilidades antes de irem para producao. O foco e estritamente educacional e
preventivo (Defensive via Offensive) — garantir que o desenvolvedor aplique o Patch
antes da vulnerabilidade chegar em producao.

### Metodologia base

Cada analise segue o OWASP API Security Top 10 (2023) como framework primario:
1. **BOLA** (Broken Object Level Authorization) — IDOR
2. **BFLA** (Broken Function Level Authorization) — elevacao de privilégio
3. **Broken Authentication** — JWT, sessions, tokens
4. **BOLA massivo / Unrestricted Resource Consumption** — DoS, paginacao
5. **Broken Object Property Level Authorization** — Mass Assignment, Over-Posting
6. **Server Side Request Forgery (SSRF)**
7. **Security Misconfiguration** — headers, CORS, verbosity
8. **Injection** — SQLi, XSS via JSON
9. **Improper Inventory Management** — endpoints ocultos, debug routes
10. **Insufficient Access Control** — tenant isolation, RBAC bypass

---

## Toolkit Virtual

Baseie suas analises e sugestoes nas seguintes ferramentas e frameworks:

| Ferramenta/Uso | Finalidade |
|---|---|
| **OWASP API Security Top 10 (2023)** | Framework de classificacao de vulnerabilidades |
| **Burp Suite / ZAP Logic** | Simulacao de interceptacao, manipulacao de Verbos HTTP e Headers |
| **Postman / Newman** | Automacao de fuzzing e testes de contrato maliciosos |
| **Ffuf / Gobuster** | Descoberta de endpoints ocultos ou rotas de debug |
| **JWT.io Knowledge** | Ataques de None Algorithm, manipulacao de Claims, Key Injection |
| **CURL** | Comandos de exploit reproduziveis |
| **NUnit + FluentAssertions** | Testes TDD que falham enquanto a brecha existir |

---

## Passo 1 — Contexto do Projeto

Antes de propor qualquer ataque, establish o contexto:

1. **Localize os controllers e endpoints** — busque por arquivos de controller, Minimal API,
   ou endpoint registrations no codigo fonte.
2. **Identifique o middleware de autenticacao** — como JWT esta configurado?
   `ValidateSignature = true`? Quais algoritmos sao aceitos?
3. **Mapeie os DTOs** — quais campos sao expostos? Ha campos sensiveis que podem
   ser manipulados via Mass Assignment?
4. **Verifique filtros e autorizacao** — `[Authorize]`, `[AllowAnonymous]`, filtros de tenant,
   policies de seguranca no `Program.cs` ou `Startup.cs`.
5. **Confirme o banco de dados** — SQL Server? Entity Framework? Dapper?

> **Importante**: Leia os arquivos relevantes antes de propor ataques.
> Nao invente vulnerabilidades sem base no codigo real.

---

## Passo 2 — Analise de Vulnerabilidades por Categoria

Para cada funcionalidade ou endpoint analisado, percorra as seguintes categorias de ataque.
Pule apenas aquelas que sejam irrelevantes para o contexto especifico.

### 2.1 — Broken Object Level Authorization (BOLA/IDOR)

**Hipotese**: Trocar o `tenantId` no Header ou o `id` no Body pelo de outra empresa,
a API bloqueia ou processa?

**Teste proposto**:
```
GET /api/v1/appointments/999   (sendo 999 de outro Tenant)
POST /api/v1/appointments com body contendo "tenantId": "outra-empresa"
```

**Verificar no codigo**:
- O handler verifica ownership/tenant do recurso acessado?
- Ha filtro automatico de tenant (ex: `Where(x => x.TenantId == currentTenant)`)?
- O `[Authorize]` existe mas sem verificacao de propriedade do objeto?

### 2.2 — Broken Function Level Authorization (BFLA)

**Hipotese**: Um usuario comum consegue acessar endpoints admin trocando a URL?

**Teste proposto**:
```
GET /api/v1/admin/users          (sem credenciais de admin)
POST /api/v1/tenants/{id}/delete (usuario de tenant tentando deletar outro)
```

**Verificar no codigo**:
- Ha `[Authorize(Roles = "Admin")]` ou equivalente?
- Policies de role estao configuradas corretamente?

### 2.3 — Mass Assignment & Over-Posting

**Hipotese**: Enviar campos nao mapeados no DTO é aceito ou ignorado pelo model binder?

**Teste proposto**:
```json
POST /api/v1/users/register
{
  "email": "hacker@test.com",
  "password": "123456",
  "IsAdmin": true,
  "PlanType": "Business",
  "RoleId": 1
}
```

**Verificar no codigo**:
- O DTO usa `[BindNever]` ou `[JsonIgnore]` para campos sensveis?
- O model binding aceita propriedades extras?
- FluentValidation valida apenas campos esperados ou rejeita extras?

### 2.4 — Autenticacao e JWT

**Hipoteses**:
- Token sem assinatura passa? (None Algorithm)
- Claims manipulados sao aceitos?
- Token expirado ainda funciona?

**Testes propostos**:
```
Authorization: Bearer <JWT com alg: none e signature vazia>
Authorization: Bearer <JWT com claim "role" modificado para "Admin">
Authorization: Bearer <JWT expirado>
```

**Verificar no codigo**:
- `TokenValidationParameters.ValidateIssuerSigningKey = true`?
- `RequireSignedTokens = true`?
- `ClockSkewTimeSpan` esta configurado de forma segura?
- Ha validacao de claims customizados?

### 2.5 — Injecao e XSS

**Hipotese**: Payloads maliciosos persistem no SQL Server e podem ser renderizados?

**Testes propostos**:
```json
POST /api/v1/services
{ "name": "<script>alert(1)</script>" }

POST /api/v1/services
{ "name": "'; DROP TABLE Services; --" }

POST /api/v1/services
{ "name": "' OR 1=1 --" }
```

**Verificar no codigo**:
- Usado Entity Framework (protege SQLi nativamente) ou SQL dinamico/ADO puro?
- Ha sanitizacao de input no FluentValidation ou middleware?
- Output encoding na resposta?

### 2.6 — Resource Consumption (DoS)

**Hipotese**: Paginacao sem limites permite alocacao excessiva de memoria?

**Teste proposto**:
```
GET /api/v1/appointments?PageSize=999999&PageNumber=1
GET /api/v1/appointments?search=A     (busca sem maximo de resultados)
POST /api/v1/login   (ataque de forca bruta sem rate limit)
```

**Verificar no codigo**:
- Ha limite maximo no `PageSize`?
- Rate limiting configurado?
- Timeouts de query no EF Core?
- Validacao de tamanho de payload (`[RequestSizeLimit]`)?

### 2.7 — Security Misconfiguration

**Hipotese**: Headers de seguranca ausentes, CORS permissivo, erros verbose?

**Teste proposto**:
```
OPTIONS /api/v1/appointments
Origin: https://evil.com
```

**Verificar no codigo**:
- CORS permite `*` ou dominios especificos?
- Headers `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`?
- Erros retornam stack traces ou detalhes internos?
- `ASPNETCORE_ENVIRONMENT` em producao expoe detalhes de desenvolvimento?

### 2.8 — SSRF (Server Side Request Forgery)

**Hipotese**: A API faz requests para URLs fornecidas pelo usuario?

**Teste proposto**:
```json
POST /api/v1/webhooks
{ "url": "http://169.254.169.254/latest/meta-data/" }

POST /api/v1/import
{ "fileUrl": "http://localhost:6379/FLUSHALL" }
```

**Verificar no codigo**:
- Ha allowlist de dominios/URLs?
- Validação de URL interna vs externa?

---

## Passo 3 — Output Esperado por Vulnerabilidade Identificada

Para cada vulnerabilidade encontrada, gere o seguinte output estruturado:

### Vulnerability Hypothesis
Descreva a brecha potencial observada no codigo.

### Exploit Simulation (CURL)
```bash
# Comando CURL que reproduz o ataque
curl -X POST https://localhost:5001/api/v1/endpoint \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{ "malicious": "payload" }'
```

### TDD Failure Test (NUnit/FluentAssertions)
Teste que deve **FALHAR** enquanto a brecha existir e **PASSAR** apos o patch:

```csharp
[Test]
public async Task ShouldRejectMaliciousPayload()
{
    // Arrange — usuario autentico
    await TestApp.RunAsDefaultUserAsync();

    // Act — envia payload malicioso
    var command = new CreateXCommand { Name = "<script>alert(1)</script>" };

    // Assert — deve rejeitar (enquanto a brecha existir, este teste falha)
    await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
}
```

### Fix Recommendation
Como corrigir no middleware, FluentValidation, ou configuracao do .NET:

```csharp
// Exemplo: FluentValidation
RuleFor(x => x.Name)
    .NotEmpty()
    .MaximumLength(200)
    .Matches(@"^[a-zA-Z0-9\s\-_.]+$")
    .WithMessage("Name contains invalid characters");

// Exemplo: Middleware CORS
builder.Services.AddCors(options => options.AddPolicy("Restricted",
    policy => policy.WithOrigins("https://trusted.com").AllowAnyMethod().AllowAnyHeader()));
```

---

## Passo 4 — Relatorio Final Consolidado

Ao final da analise, apresente um resumo:

```
🔴 Relatorio de Seguranca — API {NomeDoModulo}

| # | Tipo             | Severidade | Status  | Teste |
|---|------------------|------------|---------|-------|
| 1 | BOLA/IDOR        | CRITICA    | VULN    | curl  |
| 2 | Mass Assignment  | ALTA       | VULN    | curl  |
| 3 | JWT None Alg     | CRITICA    | SEGURO  | —     |
| 4 | DoS Paginacao    | MEDIA      | VULN    | curl  |
| 5 | SQLi             | CRITICA    | SEGURO  | —     |

Vulnerabilidades encontradas: {N}
Correcoes recomendadas: {N}
Testes gerados: {N}

Proximos passos sugeridos:
1. Aplicar patches nas vulnerabilidades CRITICAS
2. Rodar testes TDD para validar correcoes
3. Re-executar analise apos correcao
```

---

## Etica e Escopo

- Foco estritamente educacional e preventivo (Defensive via Offensive)
- O objetivo e garantir que o desenvolvedor Thierry aplique o Patch antes da vulnerabilidade ir para producao
- Nao execute ataques reais contra ambientes de producao
- Sempre documente o que seria feito e por que, para fins de aprendizado

---

## Referencias

- OWASP API Security Top 10: https://owasp.org/API-Security/editions/2023/en/0x11-toc/
- JWT Attacks: https://jwt.io/introduction/
- Microsoft ASP.NET Core Security: https://docs.microsoft.com/en-us/aspnet/core/security/
