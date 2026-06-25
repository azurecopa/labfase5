# PORTAL GUIDE — Quartas de Final: Identidade dois-mundos + Migração v1→CIAM

> **Hands-on dos Blocos 2, 3 e 4** · Demo guiada: o instrutor projeta, você replica.
> **Objetivo:** sair daqui com (1) o tenant CIAM com login funcionando, (2) sua App Registration SPA plugada no MSAL, (3) o admin no workforce com App Role `Admin`, e (4) um usuário v1 **migrado** para o CIAM com a coexistência provada em SQL.
> **Story:** [2.11](../../stories/2.11.story.md) (AC-7, AC-8, AC-11, AC-12, AC-13, AC-15, AC-16, AC-17) · **Decisões:** [ADE-007 v1.1](../../architecture/ade-007-identity-external-id.md) · [migração](../../architecture/migration-v1-ciam-design.md)
> **Portais usados:** [entra.microsoft.com](https://entra.microsoft.com) (Entra admin center) e [portal.azure.com](https://portal.azure.com) (Container App / App Settings).

---

## Pré-requisitos

- Sua **Function App F1** e seu **gateway YARP F2** funcionando (a identidade entra na frente deles).
- Login no **Microsoft Entra admin center** ([entra.microsoft.com](https://entra.microsoft.com)) e no **Portal Azure** ([portal.azure.com](https://portal.azure.com)).
- Suas **iniciais** definidas (ex.: `jds`) — usamos em todos os nomes.

### O que o INSTRUTOR pré-provisiona (fora do relógio da aula)

> Estes itens **não** são hands-do aluno — o instrutor os prepara antes para o trial caber no tempo. O lab só fica viável porque o atrito do CIAM foi empurrado para fora da aula (ADE-007 Rationale).

| Item | Quem | Detalhe |
|---|---|---|
| Tenant **Entra External ID** (trial 30d, sem subscription/cartão) | Instrutor | Criado via VS Code (extensão Microsoft Entra External ID) ou portal |
| **1 user flow** sign-up/sign-in | Instrutor | Self-service, reusável por várias apps |
| **Google IdP** plugado no user flow | Instrutor | App Google OAuth2 (client id/secret) registrada; **email + OTP** é o fallback de zero dependência |
| Migration **`phase-04-ciam-link.sql`** aplicada | Instrutor / @devops | Cria `users.entra_oid` **vazia** (o **preenchimento** é o hands-on do Bloco 4) |
| ~3–5 contas de demonstração em `users` (email conhecido) | Instrutor | Alvos da migração |

> ⚠️ **Checklist de produto (AC-17):** ao trabalhar no tenant CIAM, confirme que a URL de login é **`<tenant>.ciamlogin.com`** (External ID). Se você vir **`b2clogin.com`**, é o **Azure AD B2C legado** — **pare e revise**. O lab usa **exclusivamente** o External ID; nenhum recurso B2C é criado.

---

# BLOCO 2 — Cliente Entra External ID (CIAM)

> **~90–120min · Hands-on.** Você cria sua App Registration SPA no tenant CIAM, pluga a authority no MSAL, faz sign-up + login (Google ou OTP) e vê o `entra_oid` aterrissar no SQL. **Este bloco fecha com o clímax do lab — o gateway validando o JWT que o CIAM emitiu.**

## Step 2.1 — Criar a App Registration SPA no tenant CIAM (15min)

> Trabalhe no **tenant CIAM** (não no workforce). No Entra admin center, confirme o tenant ativo no topo: deve ser o **External ID**, com domínio `*.ciamlogin.com`.

1. Em [entra.microsoft.com](https://entra.microsoft.com), confirme que está no **tenant externo** (canto superior direito → "Switch tenant" se necessário).
2. Menu **Identity → Applications → App registrations** → **`+ New registration`**.
3. **Name:** `student-<iniciais>-v2` (ex.: `student-jds-v2`).
4. **Supported account types:** **Accounts in this organizational directory only** (single-tenant — é o tenant CIAM).
5. **Redirect URI:** selecione a plataforma **Single-page application (SPA)** e informe `http://localhost:5173` (dev Vite). Você adiciona a URL de prod depois.
6. **`Register`**.

> `[PRINT 1: formulário New registration com platform=SPA e redirect localhost:5173]`

7. Na **Overview** da app, anote:
   - **Application (client) ID** → vira o `clientId` do MSAL e o `aud` esperado do token.
   - **Directory (tenant) ID** → vira o `<tenantId>` do gateway.

8. Em **Authentication**, confirme que a plataforma é **Single-page application** (Authorization Code + PKCE — **sem** client secret no browser). **Não** crie client secret: SPA é client público.

9. (runtime) Garanta que sua app foi **adicionada ao user flow** que o instrutor pré-provisionou (Entra admin center → **External Identities → User flows** → o flow → **Applications** → `+ Add application` → selecione `student-<iniciais>-v2`).

✅ **Checkpoint:** App Registration SPA criada no tenant CIAM, **client ID** e **tenant ID** anotados, redirect SPA `localhost:5173` configurado, app vinculada ao user flow.

> ⚠️ **Armadilha:** criar a app no tenant **workforce** por engano (em vez do CIAM). Confirme o domínio do tenant no topo (`*.ciamlogin.com`). App no tenant errado = login do cliente nunca funciona.

---

## Step 2.2 — Plugar a authority CIAM no MSAL (`authV2.ts`) (15min)

O código do frontend já usa MSAL (`@azure/msal-browser`) desde a F3. A **única** mudança das Quartas é a **authority**: ela deixa de apontar para o workforce e passa a apontar para o CIAM.

1. Em `Lovable/World Cup Tickets Hub/src/lib/authV2.ts`, a authority do cliente passa a ser a do **External ID**:

```ts
// ANTES (F3 — workforce):
//   const authority = `https://login.microsoftonline.com/${tenantId}`;
// AGORA (Quartas — CIAM External ID):
const ciamAuthority = import.meta.env.VITE_CIAM_AUTHORITY;   // https://<tenant>.ciamlogin.com/
const clientId      = import.meta.env.VITE_CIAM_CLIENT_ID;

const msalConfig = {
  auth: {
    clientId,
    authority: ciamAuthority,                  // <tenant>.ciamlogin.com (NÃO microsoftonline)
    knownAuthorities: ['<tenant>.ciamlogin.com'], // necessário p/ authority non-AAD
    redirectUri: 'http://localhost:5173',
  },
  cache: { cacheLocation: 'sessionStorage', storeAuthStateInCookie: false },
};
```

> **Anti-hallucination (AC-19):** o formato da authority CIAM para o MSAL é **`https://<tenant-subdomain>.ciamlogin.com/`** (subdomínio do tenant + `ciamlogin.com`). Isto é o que a doc oficial do MSAL mostra no `authConfig.js` do quickstart de SPA para tenant externo. `PublicClientApplication`, `acquireTokenSilent`, `acquireTokenPopup` e `InteractionRequiredAuthError` já estão no `authV2.ts` da F3 e **não mudam** (MSAL é issuer-agnóstico).

2. Configure as variáveis Vite no front (App Service → Configuration, ou `.env` em dev):

| Variável Vite | Valor | O que faz |
|---|---|---|
| `VITE_CIAM_AUTHORITY` | `https://<tenant>.ciamlogin.com/` | Authority do MSAL para o cliente CIAM |
| `VITE_CIAM_CLIENT_ID` | Application (client) ID da App Reg SPA (Step 2.1) | Identifica a app cliente |

3. O `apiV2.ts` (que faz `acquireTokenSilent` e envia `Authorization: Bearer`) **não muda** — o MSAL continua funcionando com a nova authority.

✅ **Checkpoint:** `authV2.ts` com `authority` apontando para `<tenant>.ciamlogin.com`, `VITE_CIAM_*` configuradas.

> ⚠️ **Armadilha nº1 das Quartas: authority errada.** Se `VITE_CIAM_AUTHORITY` ainda apontar para `login.microsoftonline.com`, o login do cliente falha com **`AADSTS50011`** (authority/redirect mismatch). **Checklist:** a authority do cliente DEVE conter **`ciamlogin.com`**, nunca `microsoftonline.com`. Se usar uma authority non-AAD (`ciamlogin.com`), o MSAL pode exigir `knownAuthorities` — sem ele, o MSAL recusa a authority como "não confiável".

---

## Step 2.3 — Configurar o gateway para validar o JWT do CIAM (15min)

Agora o gateway. O `AddJwtBearer` que validava o workforce passa a apontar o **discovery do CIAM**. Em código, **só muda a string da authority** (issuer-agnóstico).

App Settings do **Container App do gateway** (Portal Azure → seu `gateway-<iniciais>` → Containers → Edit and deploy → Environment variables):

| App Setting | Valor | O que faz |
|---|---|---|
| `Jwt__CiamTenantId` | GUID do tenant CIAM (Step 2.1) | Compõe a authority/issuer CIAM no gateway |
| `Jwt__CiamClientId` | Client ID da App Reg SPA CIAM (Step 2.1) | `aud` esperado do token CIAM |
| `Gateway:FrontendOrigin` | URL do front (ex.: `https://fifa2026-web-<iniciais>.azurewebsites.net`) | CORS restrito ao front (AC-4) |

> **Como o gateway monta o discovery CIAM (AC-19 — validado contra docs Microsoft Entra External ID):**
> - **Authority:** `https://<tenant>.ciamlogin.com/<Jwt__CiamTenantId>`
> - **Issuer (`iss`) validado:** `https://<tenant>.ciamlogin.com/<Jwt__CiamTenantId>/v2.0`
> - **Discovery:** `https://<tenant>.ciamlogin.com/<Jwt__CiamTenantId>/v2.0/.well-known/openid-configuration`
>
> Desse documento o `AddJwtBearer` busca o `jwks_uri` (chaves públicas RS256) e o `issuer`. Validação **fail-closed** herdada da F3: ausência de tenant ou `"common"` → a app não sobe. `ClockSkew=Zero` (token expirado → 401 imediato). `ValidIssuer`/`ValidAudiences` explícitos (não inferidos só do metadata).

> 🔒 **Duplo underscore:** em variável de ambiente, `Jwt:CiamTenantId` vira **`Jwt__CiamTenantId`** (dois underscores = separador de seção do .NET). A connection string do SQL **NÃO** vai no gateway — fica na Function.

✅ **Checkpoint:** App Settings `Jwt__CiamTenantId` + `Jwt__CiamClientId` + `Gateway:FrontendOrigin` no Container App; nova revisão ativa.

---

## Step 2.4 — Login CIAM no browser + smoke test ponta-a-ponta (20min)

Agora a prova: sign-up self-service, login (Google ou OTP), e o `entra_oid` aterrissando no SQL.

1. Abra o SPA em `http://localhost:5173` (ou a URL de prod). Clique **Entrar (v2)**.
2. Você é redirecionado para a tela do **External ID** (`<tenant>.ciamlogin.com`). Faça **sign-up self-service**:
   - **Opção A — Google:** "Continuar com Google" (IdP social pré-configurado pelo instrutor).
   - **Opção B — email + OTP (fallback):** informe um email; o External ID envia um **código de uso único (OTP)**; digite o código.
3. Após autenticar, o MSAL completa o **Authorization Code + PKCE** e o SPA recebe o **access token** (JWT do CIAM).
4. Faça uma compra (`POST /purchase`). O SPA envia `Authorization: Bearer <token-CIAM>` ao gateway.
5. O gateway valida o JWT CIAM, extrai o claim **`oid`**, injeta **`X-Entra-OID`** downstream; a Function grava **`purchases.entra_oid`**.

Verifique no SQL que o `oid` do CIAM aterrissou no eixo de compra:

```sql
-- O entra_oid da compra v2 recém-feita (eixo COMPRA — transacional).
SELECT TOP 5 id, user_id, entra_oid, status, created_at
FROM dbo.purchases
WHERE entra_oid IS NOT NULL
ORDER BY id DESC;
```

> **Como ver o `oid` no token (didático):** cole o access token em [jwt.ms](https://jwt.ms) (ou DevTools → Network) e localize o claim **`oid`** (GUID), o **`iss`** (termina em `<tenant>.ciamlogin.com/<tenantId>/v2.0`) e o **`aud`** (= o client ID da sua App Reg SPA). **Nunca** cole tokens de produção real em ferramentas públicas — em sala é token de trial descartável.

✅ **Checkpoint (AC-11):** login CIAM no browser → access token `ciamlogin.com` → gateway valida → `X-Entra-OID` propagado → `purchases.entra_oid` (origem CIAM) gravado em SQL, **ao lado** de registros v1.

> ☕ **PONTO DE PAUSA NATURAL.** Se a turma dividir o lab em dois encontros, **encerra aqui**. Você tem o cliente CIAM real validado pelo gateway — uma aula completa por si. O Bloco 3 (admin) é uma camada por cima de um cliente CIAM já funcionando.

> ⚠️ **Armadilhas do Bloco 2:**
> - **`AADSTS50011`** no login → authority do MSAL aponta para `microsoftonline.com` em vez de `ciamlogin.com` (Step 2.2).
> - **401 "Invalid issuer"** no gateway → `Jwt__CiamTenantId` ausente/errado, ou `ValidIssuer` ainda aponta o workforce. Confirme o App Setting e a authority CIAM (Step 2.3).
> - **Google falha (redirect inválido)** → o redirect URI do Google OAuth2 não inclui o callback do CIAM. Instrutor pré-configura; **fallback: email + OTP**.
> - **`oid` ausente no token** → claim padrão do CIAM; se sumir, verifique os atributos do user flow. Fallback de leitura no gateway: URI `http://schemas.microsoft.com/identity/claims/objectidentifier`.

---

# BLOCO 3 — Admin workforce + App Role `Admin`

> **~75–90min · Hands-on.** Você constrói (não só conceitua) o login de admin no **workforce** (`login.microsoftonline.com`), com **uma** App Role `Admin`. O gateway aceita o **segundo** emissor com a mesma mecânica issuer-agnóstica.

> 🎯 **Slide de desambiguação obrigatório aqui** (ver SPEAKER-NOTES): antes de começar, o instrutor projeta o slide Connect / Entra ID / External ID. Você está saindo do mundo `ciamlogin.com` (cliente) e entrando no mundo `login.microsoftonline.com` (funcionário). São dois tenants, dois produtos.

## Step 3.1 — App Registration admin no workforce (20min)

> Agora trabalhe no **tenant workforce** (`login.microsoftonline.com`), **não** no CIAM. Troque de tenant no topo do Entra admin center se necessário.

1. Em [entra.microsoft.com](https://entra.microsoft.com), confirme que está no **tenant workforce** (domínio `*.onmicrosoft.com`, não `ciamlogin.com`).
2. **Identity → Applications → App registrations** → **`+ New registration`**.
3. **Name:** `student-<iniciais>-admin`.
4. **Supported account types:** **Accounts in this organizational directory only** (single-tenant — fail-closed, nunca `common`/multi-tenant).
5. **`Register`**. Anote o **Application (client) ID** e o **Directory (tenant) ID** (este é o tenant **workforce**, diferente do CIAM).

✅ **Checkpoint:** App Reg `student-<iniciais>-admin` criada no **workforce**, client/tenant IDs anotados.

## Step 3.2 — Definir a App Role `Admin` (15min)

> **Decisão do owner (2026-06-25):** **uma única App Role**, `Admin`. Simplicidade didática intencional — o lab cria só uma role.

1. Na App Reg admin → **App roles** → **`+ Create app role`**:
   - **Display name:** `Admin`
   - **Allowed member types:** **Users/Groups**
   - **Value:** `Admin` (este é o valor que aparece no claim `roles` do token)
   - **Description:** "Operador administrativo do app de ingressos"
   - **Do you want to enable this app role?** ✅ marcado
2. **`Apply`**.

3. Atribua a role a um usuário: **Enterprise applications** → `student-<iniciais>-admin` → **Users and groups** → **`+ Add user/group`** → selecione seu usuário admin → role **`Admin`** → **`Assign`**.

> `[PRINT 2: formulário Create app role com Value=Admin]`

✅ **Checkpoint:** App Role `Admin` criada e **atribuída** a um usuário; o token desse usuário trará `roles: ["Admin"]`.

> **Anti-hallucination (AC-19):** o claim **`roles`** é um array de strings com os App Roles atribuídos ao usuário (docs Microsoft Identity Platform — access token claims reference). É o mecanismo padrão; nada inventado.

## Step 3.3 — Gateway aceita o segundo emissor (workforce) (15min)

O gateway ganha um **segundo** `AddJwtBearer` (esquema `"Admin"`) apontando o discovery para o **workforce** — o mesmo mecanismo do CIAM, só muda a authority.

App Settings adicionais no Container App do gateway:

| App Setting | Valor | O que faz |
|---|---|---|
| `Jwt__AdminTenantId` | GUID do tenant **workforce** (Step 3.1) | Authority/issuer do admin |
| `Jwt__AdminClientId` | Client ID da App Reg admin (Step 3.1) | `aud` esperado do token admin |

> **Authority/issuer do admin (workforce — AC-19):**
> - **Authority:** `https://login.microsoftonline.com/<Jwt__AdminTenantId>/v2.0`
> - **Issuer (`iss`):** `https://login.microsoftonline.com/<Jwt__AdminTenantId>/v2.0`
> - **Discovery:** `https://login.microsoftonline.com/<Jwt__AdminTenantId>/v2.0/.well-known/openid-configuration`
>
> O gateway valida o JWT admin com a **mesma** `AddJwtBearer` mecânica do CIAM — a prova de que é issuer-agnóstico (ADE-004). Cliente e admin são **emissores diferentes, validados igual**.

## Step 3.4 — Login de admin separado do cliente (15min)

1. No SPA, o login de admin usa `authority = https://login.microsoftonline.com/<AdminTenantId>` (workforce) — **completamente distinto** do CIAM do cliente.
2. Faça login com o usuário admin. Em [jwt.ms](https://jwt.ms), confirme no token: `iss` = `login.microsoftonline.com/.../v2.0` (workforce, não `ciamlogin.com`) e `roles: ["Admin"]`.
3. O gateway aceita o JWT do workforce no handler `"Admin"`.

✅ **Checkpoint (AC-13):** login de admin funcional via workforce (`login.microsoftonline.com`), `roles:["Admin"]` no token, **separado** do login do cliente (CIAM). Dois mundos coexistindo.

> ⚠️ **Armadilhas do Bloco 3:**
> - **Admin login rejeita (401)** → o gateway não tem o segundo handler/`Jwt__AdminTenantId` configurado. Confirme os dois `AddJwtBearer` (CIAM + Admin).
> - **Confundir os tenants** → App Reg admin criada no CIAM por engano. Admin é **workforce** (`*.onmicrosoft.com`), cliente é **CIAM** (`*.ciamlogin.com`).
> - **`roles` ausente no token** → role não **atribuída** ao usuário (Step 3.2.3) ou não habilitada. Atribuir via Enterprise applications.

---

# BLOCO 4 — Migração `users` v1 → CIAM (hands-on)

> **~60–90min · Hands-on.** O ápice didático. Você migra contas de demonstração de `users` (v1/bcrypt) para o CIAM de forma **aditiva** e prova a coexistência **na mesma linha de `users`**. Mecanismo: **Opção C híbrida** (self-service sign-up + link SQL por email) — resolvido pelo @data-engineer.

> **Pré-condição (instrutor):** a migration `phase-04-ciam-link.sql` **já foi aplicada** (a coluna `users.entra_oid` existe **vazia**). Criar a coluna não é hands-on; **preencher** é.

## Step 4.1 — Listar os alvos da migração (SQL read-only) (5min)

Veja quem ainda não migrou:

```sql
SELECT id, name, email, entra_oid
FROM dbo.users
WHERE entra_oid IS NULL
ORDER BY id;
```

✅ **Checkpoint:** você vê as ~3–5 contas de demonstração com `entra_oid = NULL`.

## Step 4.2 — Sign-up no CIAM com o MESMO email do v1 (15min)

Para cada conta de demonstração, faça **sign-up self-service no CIAM** (a competência do Bloco 2) usando **o email idêntico** ao de `users`:

- Se o email for Google → "Continuar com Google".
- Senão → email + **OTP**.

> **A lição (não esconda):** a senha bcrypt do v1 **NÃO** vai para o CIAM. O External ID não importa hash bcrypt. O usuário **estabelece credencial nova** no CIAM. O `users.password` (bcrypt) permanece **intacto** no caminho v1. **Mesmo humano, duas credenciais independentes** — e isso é a lição: "no mundo gerenciado, a Microsoft cuida da credencial; você só guarda o `oid`".

## Step 4.3 — Capturar o `oid` emitido pelo CIAM (10min)

Duas vias equivalentes:

- **Via app (recomendada em sala):** logue com a conta recém-criada → o SPA chama o gateway → o gateway extrai o `oid` e injeta `X-Entra-OID`. Veja o `oid` completo no token decodificado (jwt.ms / DevTools).
- **Via Portal:** Entra admin center (tenant CIAM) → **Users** → selecione o usuário → copie o **Object ID**.

> O **Object ID** do Portal **é** o claim `oid` do token — são o mesmo GUID (docs Microsoft: o `oid` é o `id` do usuário no diretório).

## Step 4.4 — Vincular o `oid` ao registro v1 — o coração da migração (15min)

Execute o UPDATE **idempotente** que liga o `oid` do CIAM ao usuário v1 de mesmo email:

```sql
-- Liga o oid CIAM ao usuário v1 de mesmo email. Idempotente: só atua em quem ainda
-- não tem vínculo (entra_oid IS NULL). Re-rodar não muda nada (0 linhas afetadas).
UPDATE dbo.users
SET    entra_oid = @oid          -- ex.: 'a1b2c3d4-....' (oid do Step 4.3)
WHERE  email = @email            -- ex.: 'demo@contoso.com' (MESMO email do v1)
  AND  entra_oid IS NULL;        -- guard de idempotência
```

Repita para cada conta. A idempotência é garantida por três mecanismos: o `WHERE entra_oid IS NULL` (2ª execução = 0 linhas), o índice **UNIQUE filtrado** `UQ_users_entra_oid` (um `oid` ↔ um usuário), e o sign-up nativo do CIAM (email já existente não duplica).

✅ **Checkpoint (AC-15):** `entra_oid` preenchido para as contas migradas; `users.password` (bcrypt) **intacto**.

## Step 4.5 — Provar a coexistência v1/v2 — o clímax (10min)

A query da prova ([migration-v1-ciam-design.md §6](../../architecture/migration-v1-ciam-design.md)) — mostra, **na mesma linha**, bcrypt v1 **e** `entra_oid` CIAM:

```sql
-- Prova de coexistência v1 (bcrypt) + v2 (CIAM) no MESMO registro de usuário.
-- NÃO expõe o hash em texto (só prova que existe) — boa higiene de PII.
SELECT
    u.id                                   AS user_id_v1,
    u.email,
    CASE WHEN u.password LIKE '$2%'                          -- bcrypt começa com $2a/$2b/$2y
         THEN 'bcrypt-presente' ELSE 'sem-bcrypt' END AS credencial_v1,
    u.entra_oid                            AS oid_ciam_v2,
    CASE
        WHEN u.password IS NOT NULL AND u.entra_oid IS NOT NULL
            THEN 'COEXISTE (v1 bcrypt + v2 CIAM)'           -- o ápice didático
        WHEN u.entra_oid IS NULL
            THEN 'so v1 (nao migrou)'
        ELSE 'estado inesperado'
    END                                    AS status_migracao
FROM dbo.users u
WHERE u.email = @email;
```

Resultado esperado: `credencial_v1 = bcrypt-presente`, `oid_ciam_v2 = <guid>`, `status_migracao = COEXISTE (v1 bcrypt + v2 CIAM)`.

✅ **Checkpoint (AC-16):** uma linha de `users` com **as duas identidades** — homegrown (bcrypt) e gerenciada (CIAM oid) — vivas lado a lado. Modernização sem destruição, provada em banco.

> **(Opcional) Prove que o v1 ainda funciona:** logue pelo caminho v1 (Express, senha antiga) → funciona. Logue pelo CIAM → funciona. Mesmo humano, dois mundos, ambos vivos.

> **(Opcional, faixa avançada) Migração em lote via Graph:** o instrutor pode demonstrar `POST https://graph.microsoft.com/v1.0/users` para falar de import em massa em produção. **Não é o caminho avaliado da AC** (exige `User.ReadWrite.All` + consentimento admin — cerimônia alta); o caminho do lab é o self-service + UPDATE acima.

> ⚠️ **Armadilhas do Bloco 4:**
> - **Usuário não encontrado no CIAM pós-migração** → sign-up não feito com o mesmo email, ou `entra_oid` não vinculado. Re-executar o UPDATE (idempotente).
> - **`status_migracao = 'so v1'`** → o UPDATE não rodou para esse email, ou o `oid` foi colado em email diferente. Confira a query de alvos (Step 4.1).
> - **Trial CIAM expirado entre turmas** → trial de 30 dias encerrou. Recriar via VS Code Extension ou converter para tenant com subscription (free 50K MAU).

---

## Rollback (aditivo ⇒ trivial)

- **Desfazer um vínculo:** `UPDATE dbo.users SET entra_oid = NULL WHERE email = @email;` — não destrói nada, só limpa o ponteiro.
- **Reverter a migration inteira (raro):** `DROP INDEX UQ_users_entra_oid ON dbo.users; ALTER TABLE dbo.users DROP COLUMN entra_oid;` — seguro (coluna aditiva; bcrypt/`users` nunca tocados).
- **CIAM:** tenant trial é descartável; basta deletar o usuário no Portal se desejado. **Nunca** crie backup table no SQL (regra do projeto).

---

## Validação final (DoD do aluno)

- [ ] ✅ App Registration SPA criada no tenant **CIAM** (`ciamlogin.com`), redirect SPA configurado
- [ ] ✅ `authV2.ts` com `authority` = `<tenant>.ciamlogin.com` (não `microsoftonline.com`); `VITE_CIAM_*` setadas
- [ ] ✅ Login CIAM no browser (sign-up + Google **ou** email+OTP) funcionando
- [ ] ✅ Gateway valida o JWT CIAM (`Jwt__CiamTenantId`/`Jwt__CiamClientId`) e propaga `X-Entra-OID`
- [ ] ✅ `purchases.entra_oid` (origem CIAM) gravado ao lado de registros v1
- [ ] ✅ App Reg admin no **workforce** com App Role `Admin` atribuída; login admin separado do cliente
- [ ] ✅ Gateway aceita o **segundo** emissor (`Jwt__AdminTenantId`/`Jwt__AdminClientId`)
- [ ] ✅ Migração v1→CIAM executada (sign-up mesmo email + UPDATE idempotente)
- [ ] ✅ Coexistência provada: `status_migracao = COEXISTE (v1 bcrypt + v2 CIAM)` na mesma linha de `users`
- [ ] ✅ Nenhum recurso **Azure AD B2C** (`b2clogin.com`) provisionado

---

## Apêndice — Mapa rápido de troubleshooting (consulta em sala)

| Sintoma | Causa provável | Mitigação |
|---|---|---|
| **`AADSTS50011`** no login do cliente | `authV2.ts`/`VITE_CIAM_AUTHORITY` com `login.microsoftonline.com` | Authority deve ser `<tenant>.ciamlogin.com` (Step 2.2) — erro nº1 das Quartas |
| MSAL recusa authority "não confiável" | authority non-AAD sem `knownAuthorities` | Adicionar `knownAuthorities: ['<tenant>.ciamlogin.com']` no msalConfig |
| **401 "Invalid issuer"** no gateway (cliente) | `Jwt__CiamTenantId` ausente/errado; `ValidIssuer` aponta workforce | Conferir App Setting CIAM + authority `ciamlogin.com` (Step 2.3) |
| Google IdP falha (redirect inválido) | Redirect URI do Google OAuth2 não inclui o callback CIAM | Instrutor pré-configura; **fallback: email + OTP** |
| **401** no admin login | Gateway sem 2º handler / `Jwt__AdminTenantId` | Confirmar os dois `AddJwtBearer` (CIAM + Admin) no gateway |
| `roles` ausente no token admin | Role não atribuída ao usuário | Enterprise applications → Users and groups → atribuir `Admin` (Step 3.2.3) |
| `oid` claim ausente | Scope errado / user flow sem o atributo | `oid` é claim padrão CIAM; fallback URI `objectidentifier` |
| Usuário não migra / `so v1` | UPDATE não rodou ou email divergente | Re-executar UPDATE idempotente; conferir alvos (Step 4.1) |
| Trial CIAM expirado | 30 dias encerraram | Recriar via VS Code Extension ou tenant com subscription (free 50K MAU) |

> **Em produção (e no CI/CD do lab):** push, provisão de tenant/App Reg e smoke test em Azure real são responsabilidade do **@devops**. Em sala você faz à mão para entender cada peça.
