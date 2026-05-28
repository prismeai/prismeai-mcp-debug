---
name: app-mcp-implement
description: Scaffold a brand-new Prisme.ai App + MCP workspace for a third-party SaaS (REST or GraphQL). Produces index.yml, security.yml, .import.yml, helpers, Custom Code, MCP Core, tool/method automations, public App-mode instructions, and pushes to prod. Use when the user says "build an app+mcp for X", "créer une app+mcp pour X", or similar. Reuses patterns from the existing workspaces in `./prismeai-workspaces/workspaces/`.
argument-hint: '[service-name] [?api-docs-url]'
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, AskUserQuestion, Agent, mcp__prisme-ai-builder__get_prisme_documentation, mcp__prisme-ai-builder__validate_automation, mcp__prisme-ai-builder__push_workspace, mcp__prisme-ai-builder__search_workspaces
---

# App + MCP workspace builder

You are scaffolding a **Prisme.ai workspace** that exposes a third-party SaaS both as:

- A **Prisme.ai App** — tenants can call `<ServiceName>.operation:` directly from their automations
- An **MCP server** — external AI agents (Claude Desktop, etc.) can call tools via JSON-RPC 2.0 on a central endpoint

Reference implementations live in `./prismeai-workspaces/workspaces/`. List the folder and inspect whichever existing workspace matches the target API shape (auth model, REST vs GraphQL, hybrid) before writing new code.

The definitive auth pattern is `mcp-auto-install.md` (copied into this skill folder) — read it first if you haven't. It explains the HMAC-signed, single-header MCP key scheme used by all modern app+mcp workspaces.

---

## Output layout

The final workspace must look like this, anchored at `prismeai-workspaces/workspaces/<slug>/`:

```
<slug>/
├── .import.yml
├── index.yml                   # config + secrets + entity-grouped mcpTools
├── security.yml                # standard ruleset (copy verbatim)
├── swagger.yml                 # full OpenAPI 3.0 — must be generated first
├── automations/
│   ├── buildAppAuth.yml        # helper — reads config.* (App) or args (MCP)
│   ├── executeApiCall.yml      # helper
│   ├── handleApiError.yml      # helper
│   ├── formatToolOutput.yml    # helper
│   ├── routeToolCall.yml       # helper — resolves (toolName=entity, action) → operationName, dispatches
│   ├── mcp.yml                 # endpoint — JSON-RPC 2.0
│   ├── onInstall.yml           # event-driven — calls generateKey
│   ├── generateKey.yml         # endpoint — central-only
│   ├── getConfig.yml           # endpoint — tenant-only
│   ├── method-restOp.yml       # generic REST/Pulse/VizQL dispatcher (uses CC registry)
│   ├── method-graphqlOp.yml    # generic GraphQL dispatcher
│   ├── tool-restOp.yml         # MCP wrapper around method-restOp
│   ├── tool-graphqlOp.yml      # MCP wrapper around method-graphqlOp
│   └── <op>.yml                # PUBLIC App-mode — one per op (kept for tenant ergonomics)
├── imports/
│   ├── Custom Code.yml         # JS helpers + ENTITY_OPS + getOperation + buildRequest + resolveToolAction
│   └── MCP Core.yml            # entity-grouped mcpTools (mirror of index.yml)
└── pages/
    └── _doc.yml                # optional doc page (TabsView with "As App" / "As MCP")
```

**Naming rules**:

- `<slug>` = folder name = workspace slug = lowercased-with-dashes (e.g. `example-service`, `my-saas`). Workspace `name:` (display) can stay literal with spaces/accents — no constraint there.
- **`app.slug` in `.import.yml` MUST be PascalCase** — no spaces, no hyphens, no suffix like `MCP`/`APP`. Examples: `GoogleDocs`, `GoogleDrive`, `Hubspot`, `SageX3`, `Storage`. NOT `Google Docs`, `Google Drive APP MCP`, `HubspotMCP`, `Sage X3`, `Prismeai Storage`. **The app slug is immutable once published** (the platform rejects republish with a different slug — `Once published, an app slug cannot be updated`), so picking the right form on day one avoids a full delete+republish migration across the whole fleet (mcpApiKey re-issuance + tenant reinstalls). `app.name` (the human-readable label shown in the marketplace UI) is free — keep it literal with spaces if natural (`Google Docs`, `Sage X3`).
- **Everything except the public App-mode "instructions" must be `private: true`.** That includes the helpers (`buildAppAuth`, `executeApiCall`, `handleApiError`, `formatToolOutput`, `routeToolCall`), the dispatchers (`tool-restOp`, `tool-graphqlOp`, `method-restOp`, `method-graphqlOp`), and the entire `00_MCP/*` set (`mcp`, `generateKey`, `getConfig`, `onInstall`). `private: true` only hides automations from the App's instructions list — it does NOT block `endpoint: true` HTTP webhook access nor event-triggered execution. So the webhook automations (`mcp`, `generateKey`, `getConfig`) stay reachable, and `onInstall` still fires on `workspaces.apps.installed`/`apps.configured`.
- Public App-mode automations (the "instructions") use the bare operation name (e.g. `getTests.yml`, `createTest.yml`) and are **NOT** `private:`. They stay 1-per-op so tenants can keep calling `<Service>.<operation>:` from their automations.

**No more 1-per-op `tool-*` / `method-*` files.** All ops are dispatched through the 4 generic dispatchers above. The `(toolName, action) → operationName` map lives in Custom Code (`ENTITY_OPS`).

**No `triggerSync.yml`.** `MCP Core.syncMcpTools` scans `tool-*.yml` automations on disk; with the dispatcher pattern only 2 such files exist, so it would overwrite the real mcpTools array with 2 entries. The mcpTools array is set once via `imports/MCP Core.yml` at install time and that's it. (See memory entry `feedback_mcp_core_dispatcher_incompatible.md`.)

---

## Workflow — 6 phases

Run phases sequentially. Pause after each for confirmation when a decision affects the contract (service URL, slug, logo, swagger scope).

### Phase 1 — Gather the service identity

**Goal**: lock in service name, slug, base URL, auth model.

1. If `$ARGUMENTS` is empty, ask via `AskUserQuestion`:
   - "Quel service SaaS veux-tu intégrer ?"
   - "Quelle est l'URL de la doc de l'API REST (ou GraphQL) ?" (can be skipped if you can find it)
2. Propose **two slugs** and confirm both with the user before creating the folder:
   - **folder/workspace slug** in `kebab-case` (e.g. `google-docs`, `sage-x3`) — used as folder name + workspace `slug:`.
   - **app slug** in **PascalCase** (e.g. `GoogleDocs`, `SageX3`) — used as `app.slug` in `.import.yml`. **No spaces, no hyphens, no `MCP`/`APP` suffix.** This slug is **immutable once published** on the marketplace; the platform refuses republish under a different slug. Derive `<<SERVICE_SLUG>>` (camelCase) from it by lowercasing the first letter.
3. Identify the **base URL** of the API (ask if ambiguous — e.g. on-prem vs. cloud, v1 vs. v2). Typical shapes:
   - Cloud SaaS → `https://api.<service>.com/v1` or `https://<service>.cloud.<vendor>.app/api/v2`
   - On-prem → user-provided deployment URL
4. Identify the **auth model** (one of):
   - **Static token**: pass as `Authorization: Bearer <token>` or a custom header
   - **OAuth2 / client-credentials**: exchange `client_id + client_secret` for a short-lived JWT via a dedicated endpoint — requires caching in session
   - **OAuth2 / authorization-code + PKCE** (user-delegated): user logs in with their own provider account and we store a refresh token per (user × tenant). **See Phase 4.5** — requires the full OAuth scaffolding block. Detect this when the docs mention: "OAuth 2.0 Authorization Code flow", endpoints like `/oauth/authorize` + `/oauth/token`, scopes granted by the end user, or when the user explicitly wants "each user signs in with their own account".
   - **Basic auth**: `Basic base64(email:token)`
5. **Workspace ID** — `push_workspace` does NOT auto-create a workspace; it requires an existing target. Two paths:
   - **User already created one**: ask for the ID, fill `<<WORKSPACE_ID>>` immediately in `.import.yml` + `index.yml`.
   - **Brand-new workspace**: use the **`create_workspace`** MCP tool **early** (Phase 1 or Phase 6, just before the first `push_workspace`) with `{name, slug, description, labels}` to mint a real ID, then substitute it into `<<WORKSPACE_ID>>` placeholders. **`labels` MUST include `app-mcp`** (plus `MCP`, `production:app`, and the human service name) — `app-mcp` is the canonical discovery label for the connector fleet, used by the `/app-mcp-fleet-sync` skill to find every connector. Do NOT push first and hope it creates the workspace — that's not how `push_workspace` works (it returns `Unknown workspace name` and aborts).

**Do NOT** proceed to phase 2 without these 4 facts confirmed.

**When the service offers OAuth2 authorization-code**, always ask the user whether to enable it (in addition to or instead of a static PAT). If yes, Phase 4.5 is mandatory. If unsure, default to offering both — central OAuth (no `mcp-api-key`) + tenant PAT (with `mcp-api-key`) coexist at runtime, see Phase 4.5.

**When Phase 4.5 is enabled**, also capture **`<<PROVIDER_APP_URL>>`** — the provider's OAuth-application management page where the SERVICE workspace admin will create the ONE central OAuth Application (e.g. `https://gitlab.com/-/user_settings/applications`, `https://github.com/settings/applications/new`, `https://auth.monday.com/oauth2/*`). The URL is embedded in the `<<SERVICE_SLUG>>OauthClient*` secret descriptions. Ask the user in Phase 1 if you can't confidently derive it from the service docs.

### Phase 2 — Generate `swagger.yml`

**Goal**: concrete, validated OpenAPI 3.0 listing every endpoint/operation we'll expose.

1. Read the API docs. Prefer authoritative sources:
   - Official doc site (usually the canonical source)
   - GitHub Postman collections (`<service>/postman-collections` repos are common)
   - Public OpenAPI spec if one exists
2. For **pure REST APIs**, list every endpoint with path + method + request/response schemas.
3. For **GraphQL APIs**, expose each operation as a logical pseudo-endpoint under `/graphql/ops/<operationName>` with extensions `x-graphql-operation: <name>` and `x-graphql-kind: query|mutation`. The actual HTTP call will always be `POST /graphql` — the MCP layer maps the logical operation name to the right GraphQL query (see Phase 5).
4. For **hybrid APIs** (REST + GraphQL on the same domain): include both — real REST endpoints as plain operations, GraphQL ops as `/graphql/ops/*`.
5. Save at `<slug>/swagger.yml`. Validate with `python3 -c "import yaml; yaml.safe_load(open('swagger.yml'))"`.

**Checkpoint**: report the number of REST endpoints + GraphQL ops to the user. If an API is huge (>100 ops), ask which subset to expose initially.

### Phase 3 — Fetch the service logo

**Goal**: get a square, high-quality logo and **download it locally** — do NOT hotlink an external URL into `photo:`. The logo is uploaded to the workspace's own file store in Phase 6 so the workspace owns a stable `uploads.prisme.ai` URL. Every existing connector (`gitlab`, `data-galaxy`, …) uses a workspace-hosted photo, not a third-party link — hotlinked logos break (404, CSP, rate-limits, the provider rotates the asset) and leak a request to an external host on every render.

1. Find a candidate logo URL, try in order:
   - `WebFetch` the service homepage; look for a `<link rel="icon">` or `<meta property="og:image">`
   - Search `<service> logo svg OR png site:official-domain`
   - Check the favicon at `https://<domain>/favicon.ico` or `https://<domain>/favicon.svg`
   - Last resort: `https://logo.clearbit.com/<domain>` (free, returns a square PNG)
2. Prefer **SVG** > **PNG square** > other. Avoid wide banner logos.
3. Confirm the chosen source URL with the user.
4. **Download** it into the workspace folder as `prismeai-workspaces/workspaces/<slug>/logo.<ext>` — keep the real extension (`.svg`, `.png`). Example: `curl -sSL "<url>" -o prismeai-workspaces/workspaces/<slug>/logo.svg`. This file is a build artifact: it is uploaded then deleted in Phase 6, and is never part of the pushed DSUL.
5. Leave `index.yml` `photo:` **empty** (`photo: ''`) for now. It is set to the real `uploads.prisme.ai` URL in Phase 6, once the workspace exists and the logo has been uploaded to it — never store the external source URL in `photo:`.

### Phase 4 — Scaffold the workspace files

**Goal**: create all non-tool files (config, security, imports, helpers) from the templates in `./templates/`.

Templates use `<<PLACEHOLDER>>` syntax. Replace these globally:

- `<<SERVICE_NAME>>` → Human-readable display name, may include spaces/accents (e.g. `Acme Corp`, `Google Docs`, `Sage X3`). Used as workspace `name:`, `app.name`, and inside descriptions/messages.
- `<<APP_SLUG>>` → **PascalCase** app slug used as `app.slug` in `.import.yml` (e.g. `AcmeCorp`, `GoogleDocs`, `SageX3`, `Hubspot`). No spaces, no hyphens, no suffix. **Immutable once published** — see Naming rules.
- `<<SERVICE_SLUG>>` → camelCase slug used in event names + secret keys + `slug:<...>` cross-workspace URLs (e.g. `acmeCorp`, `googleDocs`, `sageX3`). Derived from `<<APP_SLUG>>` by lowercasing the first letter.
- `<<WORKSPACE_ID>>` → Prisme.ai workspace ID (e.g. `_gwEr1h`) — only known after the workspace is created on the platform, use a placeholder then update
- `<<BASE_URL>>` → API base URL
- `<<LOGO_URL>>` → leave empty: substitute with `''` so `index.yml` ships `photo: ''`. The real `uploads.prisme.ai` URL is set in Phase 6 after the logo file is uploaded to the workspace — never substitute a hotlinked external URL here.
- `<<PROVIDER_APP_URL>>` → OAuth-app management page URL (Phase 4.5 only — captured in Phase 1)

**Steps**:

1. `mkdir -p prismeai-workspaces/workspaces/<slug>/{automations,imports,pages}`
2. Copy `templates/security.yml` verbatim.
3. Copy `templates/.import.yml` and substitute placeholders.
4. Copy `templates/index.yml` and fill in:
   - Description (2 sentences summarising the service)
   - `config.schema` — one field per credential + the standard `mcpEndpoint` / `mcpApiKey` (readOnly)
   - `config.value` — NEVER add `mcpApiKey` (see mcp-auto-install.md §"Don't put auto-generated config in config.value")
   - `config.value.mcpTools` — one entry per operation, with `inputSchema` pulled from swagger + always adding an `outputFormat` enum property
   - `secrets.schema` — one field per credential + `appSecret` (HMAC secret; leave its **value empty** — `generateKey.yml` auto-generates and persists it on the first key request)
5. Copy `templates/helpers/*.yml` into `automations/` and substitute placeholders. **Do NOT copy `triggerSync.yml`** even if it exists in the templates folder — it's incompatible with the dispatcher pattern (see "MCP tool discovery" + Common Traps).
6. **Adapt `buildAppAuth`** for the auth model (see the `<<ADAPT>>` comment inside):
   - Static token: keep simple, just read `config.token`
   - OAuth2: add a `fetch` to exchange credentials + session cache (look at an existing OAuth2 workspace's `buildAppAuth.yml` as reference)
7. **Adapt `executeApiCall`** for the auth header shape:
   - Bearer: `Authorization: Bearer {{accessToken}}` (default)
   - Custom: rename the header, adjust pruning of Content-Type for GET, etc.
8. **Adapt `getConfig`**: return **exactly** the shape that `mcp.yml` expects in the `creds` block. If buildAppAuth needs `clientId/clientSecret`, getConfig must return them; if it just needs `token`, only forward that.
9. Copy `templates/imports/Custom-Code.yml` into `imports/Custom Code.yml`. If the API is GraphQL, add a `getGraphqlQuery` function with the full registry of operations (look at an existing GraphQL workspace's `imports/Custom Code.yml` — the registry is a big JS object keyed by operation name).
10. Create `imports/MCP Core.yml` by mirroring `config.value.mcpTools` from `index.yml`. Automate this with a short Python script:
    ```python
    import yaml
    d = yaml.safe_load(open('index.yml'))
    out = {'appSlug': 'MCP Core', 'slug': 'MCP Core', 'config': {'apiKey': '', 'mcpTools': d['config']['value']['mcpTools']}}
    yaml.dump(out, open('imports/MCP Core.yml','w'), sort_keys=False)
    ```

11. **Tenant secret auto-wiring** — mandatory for any connector whose `secrets.schema` contains tenant-sensitive credentials (apiKey, token, password, OAuth client id/secret, etc.). Skip ONLY if the app has zero tenant secrets.

    **Why** — without this step, the tenant admin has to (a) create each secret in the Builder "Secrets" section AND (b) manually wire `{{secret.X}}` into the matching app-instance config field. Most admins don't know this 2-hop pattern. With auto-wiring, the admin only fills the **values** in the Secrets section; everything else is wired by `onInstall.yml` at install time.

    **Pattern** (validated end-to-end on `google-docs` for OAuth + `google-search` for static-token; same mechanism, different secret keys) — chains:
    ```
    app-instance config.<configKey> = "{{config.<secretKey>}}"       ← binding A (terminal set: config merge)
    workspace config.value.<secretKey> = "{{secret.<secretKey>}}"    ← binding B (PATCH /config via workspace JWT)
    Secrets section: <secretKey> = <real value>                      ← filled by admin
    → resolution chain: instance {{config.x}} → workspace config.value {{secret.x}} → secret store ✓
    ```

    **Concrete steps in `automations/onInstall.yml`**:
    1. After the `mcpEndpoint` set and before the `<<TENANT_SECRET_PROVISION_HOOK>>` marker, insert the block from `templates/fragments/onInstall-tenant-secret-provision.yml`. Repeat the per-secret sub-blocks once per credential listed in `secrets.schema`.
    2. In the terminal `set: config type: merge` (at the `<<TENANT_SECRET_BINDING_A>>` marker), add `<configKey>: '{{binding<SecretKeyPascal>}}'` for each credential.
    3. Each secret's `description:` must guide the admin to the source (e.g. URL to vendor console, scopes to enable, where to copy from). The descriptions are the **only** onboarding doc most admins will see.

    **Required Custom Code helpers** (already in the template `imports/Custom-Code.yml`): `makeConfigRef(key)` and `makeSecretRef(key)`. They build the literal binding strings in plain JS — a direct `{{config.x}}` / `{{secret.x}}` in a DSUL set/fetch body resolves to empty at set-time, so the literals MUST go through Custom Code to survive substitution.

    **The block is idempotent** — GET `/security/secrets` first, only PATCH the secrets that don't already exist (admin may have created some manually).

    **Reference implementations**:
    - **Static-token** (apiKey + optional cx + optional model): `prismeai-workspaces/workspaces/google-search/automations/onInstall.yml`
    - **OAuth user-delegated** (oauthClientId + oauthClientSecret): `prismeai-workspaces/workspaces/google-docs/automations/onInstall.yml`

    OAuth connectors (Phase 4.5) **no longer extend this block** in the central model — the OAuth client lives in the SERVICE workspace's own secrets (see Phase 4.5 step 6/7), not provisioned per-tenant. The PAT auto-wiring above still applies.

### Phase 4.5 — OAuth2 user-delegated, CENTRAL mode (conditional)

**Skip this phase entirely if Phase 1 determined the service uses static token / basic auth / client-credentials only.** Otherwise, the OAuth templates live in `./templates/oauth/` and extend (not replace) the non-OAuth scaffold.

#### What changed (vs the legacy per-tenant model)

The legacy model exposed `oauthClientId`/`oauthClientSecret` (and 6 URL fields) in the App's `config.schema` — every tenant had to register their own OAuth Application with the provider, copy 2 secrets into their app instance, and pin a per-tenant callback URL. `onInstall.yml` ran a 2-hop binding A/B to per-tenant Builder secrets.

**Now the OAuth credentials are CENTRAL** — they live in the secrets of the SERVICE workspace (the workspace this skill scaffolds), shared by every Prisme.ai user who calls `/apps/<<SERVICE_SLUG>>` without an `mcp-api-key` header. Concretely:

- ONE OAuth Application registered with the provider (callback URL = `<api>/workspaces/slug:<<SERVICE_SLUG>>/webhooks/oauthCallback`, unique).
- TWO secrets in the service workspace: `<<SERVICE_SLUG>>OauthClientId` + `<<SERVICE_SLUG>>OauthClientSecret`. Admin fills them once via Builder UI > Secrets.
- The published App is **PAT-only**: tenants who want their own auth install the App and set `token` (Personal Access Token). No OAuth knob in the App config schema.

#### Two runtime modes (carved into `mcp.yml`)

| Mode | Trigger | Identity | Auth resolution |
|---|---|---|---|
| **CENTRAL** (default) | no `mcp-api-key` header | `{{user.id}}` ambient (Prisme.ai JWT) | user-delegated OAuth; tokens in user-scope `delegated_token_central` |
| **TENANT** | `mcp-api-key` header | tenant workspaceId decoded from key | static PAT (`{{config.token}}`); no OAuth in tenant mode |

`connect`/`disconnect` MCP tools exist only in central mode; `tools/list` filters them out in tenant mode.

#### Steps

1. **Copy templates verbatim** — substitute `<<SERVICE_SLUG>>` (kebab-case), `<<SERVICE_NAME>>` (display), `<<BASE_URL>>` (API base URL), `<<PROVIDER_AUTHORIZE_URL>>`, `<<PROVIDER_TOKEN_URL>>`, `<<PROVIDER_REVOKE_URL>>`, `<<PROVIDER_HOSTNAME>>` (provider DNS), `<<PROVIDER_APP_URL>>` (where admin creates the OAuth app):

   ```
   templates/oauth/automations/
     ├─ initiateOAuth.yml          endpoint — PKCE + redirect to provider (central; no mcp-api-key required)
     ├─ oauthCallback.yml          endpoint — exchange code, store user.<svc>.delegated_token_central
     ├─ disconnectOAuth.yml        endpoint + internal — revoke + delete secrets (scope=central)
     ├─ refreshOAuthToken.yml      internal — refresh access_token (scope-parameterised)
     ├─ checkAuthStatus.yml        endpoint — { connected, expiresAt, scopes, scope }
     ├─ ensureAuthentication.yml   internal — central=OAuth user / tenant=PAT (2 priorities)
     ├─ connect.yml                App-mode public — returns connect_url to central initiateOAuth
     ├─ method-connect.yml         shared core (central-only — refuses in tenant mode)
     ├─ tool-connect.yml           MCP wrapper (propagates `scope`, `authMode`)
     ├─ disconnect.yml             App-mode public
     ├─ method-disconnect.yml      shared core
     └─ tool-disconnect.yml        MCP wrapper
   templates/oauth/pages/
     └─ connector-callback.yml     success page (legacy DSUL; can be migrated to a React SPA via /workspace-page-implement)
   ```

2. **REPLACE `automations/mcp.yml`** with `templates/oauth/fragments/mcp.yml`. The OAuth version dispatches CENTRAL vs TENANT at the top (single `set authMode: central` + `conditions: '{{headers["mcp-api-key"]}}' → set authMode: tenant`), then special-cases `connect`/`disconnect` (central-only), then delegates to `ensureAuthentication` for every other tool. **Do NOT use a DSUL ternary** (`a ? b : c` is rejected by the parser — `feedback_dsul_no_ternary`).

3. **REPLACE `automations/getConfig.yml`** with `templates/oauth/fragments/getConfig.yml`. In central mode this webhook is NOT called; in tenant mode it returns ONLY `accessToken` (the PAT) + `baseUrl`. No OAuth block — OAuth lives in the central workspace, not the tenant.

4. **Merge `templates/oauth/imports/Custom-Code-oauth-fragment.yml`** into `imports/Custom Code.yml` under `config.functions` — adds `generatePkce`, `generateState`, `buildAuthorizeUrl`. **Do NOT use `URLSearchParams` or `URL` in Custom Code** (sandbox is Node without web globals — see Gotchas). `makeConfigRef`/`makeSecretRef`/`cleanCredential` from the universal Custom Code template are still used (PAT auto-wiring keeps them relevant).

5. **`templates/oauth/fragments/index-config-schema.yml` is now empty (deprecated).** The App `config.schema` exposes ONLY `baseUrl` + `token` (PAT) + `mcpEndpoint`/`mcpApiKey` (readOnly). Nothing OAuth-related.

   ⚠️ **R18 (hard rule)** — Never expose `oauthClient*`, `authorizationUrl`, `tokenUrl`, `revocationUrl`, `scopes`, `refreshTokenTtl`, or `oauthCallbackUrl` in the App's `config.schema`. They are secrets of the SERVICE workspace, not tenant inputs. The published App is PAT-only.

6. **Merge `templates/oauth/fragments/index-config-value.yml`** into `index.yml` under `config.value`:

   ```yaml
   oauthClientId: '{{secret.<<SERVICE_SLUG>>OauthClientId}}'
   oauthClientSecret: '{{secret.<<SERVICE_SLUG>>OauthClientSecret}}'
   authorizationUrl: <<PROVIDER_AUTHORIZE_URL>>
   tokenUrl: <<PROVIDER_TOKEN_URL>>
   revocationUrl: <<PROVIDER_REVOKE_URL>>
   scopes: api
   refreshTokenTtl: 7200
   ```

   `{{secret.X}}` resolves at runtime when this `config.value` is consumed by the service workspace itself (NOT by tenant app instances — they don't see these keys).

7. **Merge `templates/oauth/fragments/index-secrets-schema.yml`** into `index.yml` under `secrets.schema` — declares the two `<<SERVICE_SLUG>>OauthClient*` keys with `ui:widget: password` and a description pointing at `<<PROVIDER_APP_URL>>`. Workspace admin fills the VALUES in the Builder Secrets section after scaffold.

8. **`onInstall.yml` does NOT provision OAuth secrets per-tenant anymore.** Step 6b of the legacy flow (provisioning + binding A/B for `<<SERVICE_SLUG>>OauthClient*`) is REMOVED. Keep only:
   - `mcpEndpoint`/`mcpApiKey` generation
   - The universal PAT auto-wiring block from `templates/fragments/onInstall-tenant-secret-provision.yml` for `<<SERVICE_SLUG>>Token`
   - Binding A merge for `token` only

   `templates/oauth/fragments/onInstall-provision.yml` is deprecated (now a stub doc).

9. **Prepend `templates/oauth/fragments/index-mcptools.yml`** to `config.value.mcpTools`: `disconnect` and `connect` MUST be listed first so `tools/list` surfaces them to the LLM in central mode. Regenerate `imports/MCP Core.yml` afterwards.

10. **Provider-specific tweaks** (only if the provider deviates from RFC 6749):
    - Extra body fields in `oauthCallback.yml` `/oauth/token` fetch (e.g. `audience` for Auth0, `tenant` for Azure).
    - Same for `refreshOAuthToken.yml` if refresh deviates.
    - Adjust `scopes` default in `config.value` if `api` doesn't fit.
    - **Providers without `expires_in` / refresh tokens** (e.g. Monday): the templates default `expires_in` to 10 years when absent — no code change needed.

11. **Register the OAuth app at the provider** (GitHub / GitLab / Slack / Monday / ...) with ONE callback URL (stable across central-workspace recreations):

    ```
    https://<platform-api>/v2/workspaces/slug:<<SERVICE_SLUG>>/webhooks/oauthCallback
    ```

    Example for `gitlab`: `https://api.sandbox.prisme.ai/v2/workspaces/slug:gitlab/webhooks/oauthCallback`.

    Admin fills `<<SERVICE_SLUG>>OauthClientId` + `<<SERVICE_SLUG>>OauthClientSecret` in the Builder Secrets section of the SERVICE workspace.

#### DSUL gotchas hit while validating the gitlab pilot

- **No ternary in `{% %}`** — `'{% {{header}} ? "a" : "b" %}'` ⇒ `InvalidExpressionSyntax`. Use `set default + conditions override`. See `feedback_dsul_no_ternary`.
- **No `+` operator on strings in `{% %}`** — `'{% "https://" + replace(URL(...).hostname, "api.", "") %}'` ⇒ `InvalidExpressionSyntax`. Compute parts inside `{% %}` and assemble via plain `{{}}` interpolation in a separate `value:`.
- **`global.studioUrl` may not exist** — derive the Studio origin from `global.apiUrl` instead: `'{% replace(URL({{global.apiUrl}}).hostname, "api.", "") %}'`, then `https://{{studioHostname}}/...`.
- **`push_workspace` chokes on `node_modules`** under `pages/<reactApp>/` — when a React SPA shares the workspace, temporarily `mv pages/<reactApp> /tmp` before bulk pushes, or push automations individually via the MCP.

**Deliverables of this phase:** 12 new `automations/*.yml`, 1 `pages/*.yml` (or a React SPA via `/workspace-page-implement`), `mcp.yml` + `getConfig.yml` replaced, `Custom Code.yml` extended, `index.yml` extended (config.value + secrets.schema), `onInstall.yml` unchanged from Phase 4 (no OAuth provisioning block).

### Backward-compat note for migrating an existing OAuth connector

When migrating an existing per-tenant OAuth connector (google-docs, salesforce, etc.) to the central model, the breaking change is visible to tenants who had set their own `oauthClientId`/`oauthClientSecret` on their app instance: those fields disappear from the schema. Communicate before flipping the switch. The pilot for the new model is `gitlab` (workspace `bTRQGpr` on sandbox). Use `/app-mcp-fleet-sync` to propagate once validated.

### Phase 5 — Generate entity-grouped MCP tools + per-op public automations

**Goal**: 1 MCP tool per **entity** (with an `action` enum), N public App-mode automations (1 per op for tenant ergonomics), 4 generic dispatchers handling everything.

**Why entity grouping is the default** (not 1-tool-per-op):

- **OpenAI `gpt-5-chat-latest` caps tool arrays at 128.** Any mid-size API (40+ ops) blows past this; large APIs (Tableau: 131 ops, Jira: 200+) are blocked entirely. Anthropic Claude is more permissive but still benefits.
- **LLM disambiguation is sharper** with 15-25 entity tools than with 100+ specific tools.
- **No code-size penalty** — all ops still routable through one generic dispatcher backed by the JS registry.
- **App-mode UX preserved** — tenants still call `<Service>.listProjects:` directly; the per-op public files are kept verbatim.

**Architecture (visualised)**:

```
MCP client                              Workspace
─────────                               ─────────
tools/call("workbooks", {action:        ┌─ mcp.yml (extracts toolName + arguments)
   "list", pageSize: 5})         ───▶  │
                                        ├─ routeToolCall.yml
                                        │    └─ Custom Code.resolveToolAction("workbooks","list")
                                        │       → operationName = "listWorkbooks"
                                        │
                                        ├─ Custom Code.getOperation("listWorkbooks")
                                        │    → { method: GET, path, ..., graphql: false }
                                        │
                                        └─ tool-restOp.yml  (or tool-graphqlOp.yml)
                                            └─ method-restOp.yml
                                                ├─ Custom Code.buildTableauRequest(...)
                                                └─ executeApiCall.yml → HTTPS to API
```

**Step-by-step**:

1. **Define `ENTITY_OPS`** — a Python dict mapping entity → action → operationName. Pick entity names from the API's resource taxonomy (`workbooks`, `users`, `projects`, etc.). Common action vocabulary: `list`, `get`, `create`, `update`, `delete`. Resource-specific actions are fine: `download`, `publish`, `run`, `addPermissions`, `getRecentlyViewed`, etc.

   Example:

   ```python
   ENTITY_OPS = {
     'projects': {'list':'listProjects','create':'createProject','update':'updateProject','delete':'deleteProject'},
     'workbooks': {'list':'listWorkbooks','get':'getWorkbook','download':'downloadWorkbook',
                   'update':'updateWorkbook','delete':'deleteWorkbook','publish':'publishWorkbook',
                   'queryPermissions':'queryWorkbookPermissions', ...},
     'metadata':  {'query':'metadataQuery','search':'searchAssets','workbookLineage':'getWorkbookLineage', ...},
     ...
   }
   ```

2. **Generate one `mcpTools` entry per entity**:
   - `name`: the entity (e.g. `workbooks`)
   - `description`: 1-line entity prefix + `Available actions:` block with one bullet per action: `**list** (listProjects): summary — params: filter, pageSize, ...`. Verbose but the LLM uses it to pick the right action.
   - `inputSchema`:
     - `required: [action]`
     - `properties`: `action` (enum of all actions) + the **union** of all path/query/body params from every underlying op + `outputFormat`
     - For path params, expose camelCase versions (`siteId`, not `site-id`) — the dispatcher restores hyphens for path substitution.
   - Apply the broader-default-than-intent enrichment heuristic at the **action description level**, not the tool description level (the action bullet says "params: filter, scope=assigned_to_me — pass `scope` for _your_ X").

3. **Mirror into `imports/MCP Core.yml`** — same mcpTools array. MCP Core reads this at install time and serves it on `tools/list`.

4. **Add `resolveToolAction(toolName, action)`** to `imports/Custom Code.yml`:

   ```js
   const ENTITY_OPS = {
     /* …big literal… */
   };
   const ent = ENTITY_OPS[toolName];
   if (!ent) return { error: 'Unknown entity: ' + toolName };
   if (!action)
     return {
       error:
         'Missing required argument `action` for ' +
         toolName +
         '. Available: ' +
         Object.keys(ent).sort().join(', '),
     };
   const op = ent[action];
   if (!op)
     return {
       error:
         'Unknown action `' +
         action +
         '` for ' +
         toolName +
         '. Available: ' +
         Object.keys(ent).sort().join(', '),
     };
   return { operationName: op };
   ```

5. **Update `routeToolCall.yml`** to:
   - Try `resolveToolAction(toolName, action)` first → entity dispatch
   - Fall back to direct `getOperation(toolName)` for backward-compat (so a direct call to `metadataQuery` still works)
   - Read `opMeta.graphql` to choose `tool-graphqlOp` vs `tool-restOp`
   - On unresolved entity AND unknown direct op, return the resolveToolAction error message — it lists valid actions, which is auto-recovery information for the LLM.

6. **Generate `<op>.yml` public files** as before — 1 per op for App-mode ergonomics. Each forwards to `method-restOp` or `method-graphqlOp` with the op's name and the user's args. **Every argument MUST have a `description:`** (sourced from the API docs / swagger param description) — the App-mode Builder renders the instruction form from these `arguments`, so an undescribed arg leaves the tenant guessing (a bare `id` with no hint is the classic complaint). Describe `id`/`iid`-style path params by their **specific resource** — e.g. "ID or URL-encoded path of the project" vs "ID or username of the user" vs "ID or path of the group", never a generic "id". Apply the same rule to every mcpTool `inputSchema` property.

7. **Do NOT generate `tool-<op>.yml` or `method-<op>.yml` per op.** Only the 4 generic dispatchers exist. They use `getOperation(operationName)` to look up method/path/params from the Custom Code registry built from the swagger.

8. **Do NOT create `triggerSync.yml`.** Incompatible with the dispatcher pattern (see Common Traps + memory entry `feedback_mcp_core_dispatcher_incompatible.md`).

**Generation tips**:

- One Python script reads `swagger.yml`, builds `ENTITY_OPS`, generates `index.yml mcpTools` + `imports/MCP Core.yml` + `imports/Custom Code.yml` updates + N public automations. See `templates/_generate_entity_tools.py.tmpl` for the canonical shape.
- Always run `Custom Code.run: pruneEmpty` on request bodies / GraphQL variables to strip nulls.
- For ops with **no arguments**, omit the `arguments:` key entirely (do NOT write `arguments: {}`).
- Keep the `outputFormat` enum (`structured | verbose | both`) on every entity tool.
- Auth-internal ops (`signIn`, `signOut`, refresh endpoints) **must NOT** appear in `ENTITY_OPS` — they're called by `buildAppAuth` and should never be exposed.

**Sanity checks before push**:

- `len(ENTITY_OPS)` ≤ 30 (aim for clarity)
- Every op in `swagger.yml` (minus auth-internal) appears in exactly one entity bucket
- No two entities share an action that resolves to the same operationName
- The `outputFormat` enum is present on every entity tool's inputSchema
- No `type: array` without `items:` in any entity tool's inputSchema

**Cache key versioning**: when you change `buildAppAuth`'s credential schema (e.g. PAT → JWT, or add a new scope), bump the `session.<service>` cache prefix to `session.<service>V2`. Otherwise tenants with a cached token (signed with the old credentials/scopes) keep using it until TTL expires (~3h), and your fix appears to "not work" until then.

### Phase 6 — Validate + push + smoke-check

**Goal**: clean workspace, validated, deployed to prod, smoke-tested end-to-end.

1. `validate_automation` on the full `automations/` folder. Must be 100% valid — expect warnings on webhook automations (no `arguments:`), those are fine.
2. **Pre-push sanity check** — script to catch issues that `validate_automation` misses:
   ```python
   import yaml
   d = yaml.safe_load(open('index.yml'))
   # 1. No type: array without items in mcpTools
   for t in d['config']['value']['mcpTools']:
       for pn, ps in (t['inputSchema'].get('properties') or {}).items():
           if isinstance(ps, dict) and ps.get('type') == 'array' and 'items' not in ps:
               print(f"MISSING items: {t['name']}.{pn}")
   # 2. No `#` in any code: | block of imports/Custom Code.yml (JS comments must use //)
   with open('imports/Custom Code.yml') as f:
       in_code = False
       for i, line in enumerate(f, 1):
           stripped = line.rstrip()
           if stripped.endswith('code: |'):
               in_code = True
               base_indent = len(line) - len(line.lstrip())
               continue
           if in_code:
               if stripped and not line.startswith(' '): in_code = False; continue
               cur_indent = len(line) - len(line.lstrip())
               if cur_indent <= base_indent and stripped: in_code = False; continue
               if line.lstrip().startswith('#'):
                   print(f"JS-invalid `#` comment at Custom Code.yml:{i}")
   # 3. No `type: array` in any Custom Code function parameters (use `type: object`).
   #    A single `type: array` rejects the whole CC module — every function in
   #    imports/Custom Code.yml starts returning "Function not found" at runtime.
   import re
   with open('imports/Custom Code.yml') as f:
       cc = yaml.safe_load(f)
   for fname, fdef in (cc.get('config', {}).get('functions') or {}).items():
       for pname, pspec in (fdef.get('parameters') or {}).items():
           if isinstance(pspec, dict) and pspec.get('type') == 'array':
               print(f"INVALID type:array in Custom Code function {fname}.{pname} — use type:object")
   ```
3. Human review: list `method-*`, `tool-*`, `<op>` counts to the user. Confirm before pushing.
4. **Make sure the target workspace exists.** Brand-new workspace? Call `create_workspace` FIRST with `{name, slug, description, labels}` to get a real ID, then sed the ID into `<<WORKSPACE_ID>>` in `.import.yml` + `index.yml`. **`labels` MUST include `app-mcp`** (the canonical fleet-discovery label — see `/app-mcp-fleet-sync`) alongside `production:app`. **`push_workspace` will not create a workspace from a slug — it requires the workspace to already exist.** Default environment: **`sandbox`** unless the user explicitly asks for `prod` (CLAUDE.md default, and a fresh prod connector should be sandbox-validated first anyway).
5. `push_workspace` with `workspaceId: <ID>` (NOT `workspaceName`, since a fresh workspace isn't in the env mapping yet) on `sandbox` with a short message (`initial`, `add-tools`, etc., max 15 chars, alphanumeric + `-_`). Expect ONE error in the response: `"Could not publish app — Missing photo"`. That's normal — the workspace + all files were imported; only the App-publish step needs the photo, which we set in step 6.
6. **Upload the workspace logo via the Prisme MCP, then set `photo`** — the workspace now exists, so host the logo on it instead of hotlinking an external URL.

   **Always use `mcp__prisme-ai-builder__upload_file` directly.** Do NOT propose curl + bearer token, do NOT ask the user to upload via Studio UI, do NOT use `! curl` inline-shell prefixes. The MCP tool handles auth, multipart and ACL itself — one call is enough:

   ```
   mcp__prisme-ai-builder__upload_file {
     workspaceId: <ID>,
     environment: <env>,         # sandbox | prod
     path: "<absolute path to logo.<ext>>",
     public: true
   }
   ```

   The response is an array of file objects — read `result[0].url` (looks like `https://api.<env>.prisme.ai/v2/files/<ID>/<fileId>.<name>`), inject it into `index.yml` `photo:`, re-`push_workspace` (this time the App-publish step succeeds), then delete the local `logo.<ext>` build artifact. **Never** fall back to hotlinking the original external source URL into `photo:`. (See memory entry `feedback_app_mcp_logo_upload_via_mcp.md` — older revisions of this skill recommended an `AskUserQuestion` curl/UI fallback, which is wrong.)

7. **Post-push smoke check — do not skip**, it catches deploy-time failures that validation can't:
   - **Custom Code reload**: right after push, search events for `Custom Code.error` on `fetchAPI` / `onParentAppPublish` — if any, the CC runtime didn't reload and functions will be "not found" at runtime. Recovery: `update_app_instance_config` with the **complete** functions map (the tool replaces, doesn't merge — see memory). Then re-push.
   - **generateKey round-trip**: `execute_automation generateKey` with a dummy `{body: {workspaceId: "test", getConfigUrl: "https://x.y/z"}}` and confirm the response is a proper signed key `^[A-Za-z0-9_-]+\.[a-f0-9]{64}$` — not an error object. Catches `#`-in-code-block-style Custom Code issues.
   - **tools/list**: simulate a JSON-RPC `tools/list` via `execute_automation mcp` with `{body: {jsonrpc: "2.0", id: 1, method: "tools/list", params: {}}}`. Confirm `result.tools` is an array of **entity-grouped** tools (length should match `len(ENTITY_OPS)`, not the raw op count) and that each tool's `inputSchema.properties` contains an `action` enum with the declared actions (not just `{}`). MCP Core's internal formatter has historically stripped `inputSchema` properties — verify end-to-end what the LLM will actually see.
   - **Transport contract coverage**: when building or updating the matching `*-consumer` workspace, include the DSUL `fetch` transport-contract test from `/app-mcp-build-consumer`: `initialize` returns HTTP 200 with a JSON-RPC result, `notifications/initialized` returns HTTP 202 with an empty body, and `tools/list` returns HTTP 200 with `result.tools`. This belongs in consumer coverage because `execute_automation` bypasses webhook HTTP serialization.
   - **resolveToolAction round-trip**: pick one entity + one action and `execute_automation routeToolCall` with `{toolName: "<entity>", toolArgs: {arguments: {action: "<action>"}}}`. Confirm it dispatches to the right operationName and returns a non-error result. Catches a malformed `ENTITY_OPS` registry.
   - **Hallucinated action-verb sub-paths**: grep methods for suspect endpoint suffixes and flag each for manual verification against the official API docs:
     ```bash
     grep -nE "path: '[^']*/(close|reopen|approve|unapprove|merge|cancel|retry|revoke|publish|archive|lock)'" automations/method-*.yml
     ```
     Common truth: most APIs use state-event-in-body (e.g. `PUT /issues/{iid}` + `{state_event: close}`) instead of a dedicated sub-resource. Fix any hit before smoke-testing.
   - **OAuth secret auto-wiring present (OAuth connectors only)**: confirm `onInstall.yml` contains the per-tenant secret-provisioning block and the CC helpers exist — `grep -c secretsProvisioned automations/onInstall.yml` ≥ 1, and both `makeConfigRef` and `makeSecretRef` present in `imports/Custom Code.yml`. Zero hits = you copied a pre-Phase-4.5 reference (`google-mail`/`google-sheets`); port the block from `google-docs` / `templates/oauth/fragments/onInstall-provision.yml` before shipping. End-to-end check after a real tenant install: `getConfig` returns a resolved `oauth.clientId` once the admin fills the secret VALUES in the Builder "Secrets" section (no manual config-binding step).
8. Instruct the user how to **activate**:
   - Configure secrets (`<slug>Token` or `<slug>ClientId`/`<slug>ClientSecret`) on the central workspace. `appSecret` is **optional** — it auto-generates on the first key request; set it manually only to use a known value or to pre-empt the concurrent-first-install race.
   - Install the app in a tenant workspace; configure credentials. `mcpEndpoint` + `mcpApiKey` populate automatically via `onInstall`.
   - Call `<ServiceName>.<op>:` from a tenant automation, or point an MCP client at `mcpEndpoint` using `mcp-api-key`.
9. Optionally create `pages/_doc.yml` — look at an existing workspace's `pages/_doc.yml` as reference (TabsView with "Usage as App" / "Usage as MCP").

---

## MCP tool discovery — `imports/MCP Core.yml` is the source of truth

The `/mcp` endpoint does **not** read `config.value.mcpTools` from `index.yml` directly at runtime. It reads from the **MCP Core app instance's config**, which is populated **at install time** when Prisme.ai imports `imports/MCP Core.yml`. That's why we mirror the entity-grouped mcpTools array into both `index.yml` (for documentation/readability) and `imports/MCP Core.yml` (for runtime).

**Do NOT use `MCP Core.syncMcpTools` / a `triggerSync` automation.** The sync routine scans `tool-*.yml` automations on disk; with the dispatcher pattern, only 2 such files exist (`tool-restOp`, `tool-graphqlOp`), so calling sync would **overwrite** the real entity-grouped mcpTools array with two dispatcher entries. This is a one-way data loss — you'd have to re-push the workspace to recover.

**When you change the mcpTools list** (add/rename/drop ops, change descriptions), the workflow is:

1. Edit `index.yml` mcpTools (or regenerate via the entity-grouping script)
2. Mirror into `imports/MCP Core.yml`
3. `push_workspace` — Prisme.ai re-imports `imports/MCP Core.yml` and replaces the MCP Core app instance's config
4. Smoke-test `tools/list` to confirm the new array

There's no event-driven mechanism in the dispatcher pattern. Push = sync. That's the contract.

---

## MCP endpoint security checklist

Apply this to every connector before pushing — it is part of the deliverable,
not optional polish:

- **Mark every credential-bearing input `secret: true`.** In `config.schema`:
  tokens, API keys, passwords, client secrets, OAuth client IDs/secrets,
  refresh tokens, and tenant identifiers used as credentials. In any
  `configure*` MCP tool's `inputSchema.properties` (mirrored in BOTH
  `index.yml` and `imports/MCP Core.yml`): the same. `secret: true`
  auto-redacts the value from `runtime.automations.executed` events.
- **Classify non-secret identifiers intentionally.** Resource IDs (project,
  board, site, drive, mailbox, structure IDs…), `baseUrl` and `mcpEndpoint`
  stay unredacted — state in the field `description:` that this is deliberate.
- **Never echo a credential in an error message.** No token, secret, password
  or `Authorization` header value in any `error:` / `message:` / `text:`
  string — including `decoded.error` from `verifyAndDecodeKey`: return a
  generic `Invalid or expired MCP API key` instead.
- **Never return a raw provider response body.** `handleApiError.yml` extracts
  bounded fields (`error`, `error_description`, `message`,
  `errors[0].message`) only; the `default:` branch returns a placeholder,
  never `{% json(response.body) %}` — the body may carry credential echoes or
  sensitive provider metadata.
- **Auth docs must match the implementation.** `pages/_doc.yml` and App-mode
  instructions describe only auth paths/priorities actually wired in the code —
  no stale `buildMcpAuth` / header-auth / "priority order" text for flows that
  do not exist.

---

## Common traps

All of these were already hit on existing workspaces — don't rediscover them:

- **`config.value.mcpApiKey`** — if set, it defeats `onInstall`'s "key already generated" guard (the template string is truthy even when secret is empty). Solution: do NOT add `mcpApiKey` to `config.value`.
- **Trailing `/v2` on `global.apiUrl`** — `{{global.apiUrl}}` already includes `/v2`. Do not append it in `onInstall`.
- **`set: config type: merge` is TERMINAL in `onInstall`** — persisting config re-triggers `workspaces.apps.configured`, which ends the current run. Every instruction placed AFTER the config merge (emits, fetches, more sets) is silently skipped — no error, just nothing. Keep the config merge as the very LAST instruction; emit `completed` and run all side-effects (e.g. the OAuth secret-provisioning block) BEFORE it. Symptom if violated: `onInstall.completed` (and anything after the merge) never fires even though `mcpApiKey` got set. Validated on google-docs.
- **`{{secret.X}}` resolves ONLY through a workspace's `config.value`, never directly in an automation, AND never in an app-instance config** — reading `{{secret.foo}}` inside an automation `set`/`condition`/`output` yields empty/falsy; the same literal in an installed app-instance config also stays unresolved (and `getConfig`'s `matches "{{"` guard wipes it). It resolves only as `config.value.foo: '{{secret.foo}}'`, read back via `{{config.foo}}`. An app instance chains to it: instance `config.x: '{{config.foo}}'` → workspace `config.value.foo: '{{secret.foo}}'` → secret store (the two-hop binding). An app's published `config.value` defaults do NOT propagate to tenant instances.
- **Writing a literal `{{config.x}}`/`{{secret.x}}` from an automation needs the Custom Code trick** — a plain `set`/fetch body resolves the template to empty at set-time, and `{% "{{" + … %}` concat is rejected by the parser. BUT a Custom Code function that returns the string in plain JS (`return '{' + '{secret.' + key + '}' + '}'`) works: substituting the CC-returned value is single-pass, so the literal is stored intact and resolves at read time. This is what lets `onInstall` auto-wire per-tenant OAuth bindings (`makeConfigRef`/`makeSecretRef`). Validated on google-docs.
- **`auth: workspace: true` JWT can PATCH the workspace's own `/security/secrets` and `/config`** — no extra `security.yml` grant; the workspace JWT carries the permission for its own workspace. Pattern: `- auth: {workspace: true, output: wsJwt}` then `fetch … Authorization: Bearer {{wsJwt.jwt}}`. Combined with the CC-literal trick, this lets onInstall provision secret placeholders + write both config bindings on install. Events emitted by an installed app are namespaced `AppName.<<SERVICE_SLUG>>.event` — search with a `*onInstall*` wildcard, not the bare event name, when debugging in a tenant.
- **`break: { scope: all }` in `try/catch`** — propagates as `$error = {"break":{}}`. Use nested `conditions` with `default:` instead.
- **MCP Core's internal break on invalid API key** — we bypass MCP Core for `tools/call`. MCP Core is kept only for `tools/list` / `initialize`.
- **MCP notifications must be empty at the HTTP wire boundary** — `notifications/initialized` and any JSON-RPC message without an `id` must produce `202 Accepted` with no body. Do not return `{}`, `null`, `""`, or a JSON-RPC success object. Validate this in the consumer workspace with a narrow DSUL `fetch` transport-contract test, not only `execute_automation`. If the DSUL output is intentionally empty but the HTTP response body is `{}`, fix the platform runtime serializer instead of changing the connector contract.
- **`integer` type** — DSUL only accepts `number`. Convert every `type: integer` in the swagger to `number` in DSUL.
- **NEVER use a ternary (`cond ? a : b`) inside a DSUL `{% %}` or `{{ }}` expression** — it raises `InvalidExpressionSyntax: invalid syntax` at runtime (NOT caught by `validate_automation`). The #1 offender is a helper `output:` boolean like `setupOk: '{% {{addedTool.id}} ? true : false %}'`. To compute a value conditionally, pre-`set` the variable then flip it with a `conditions:` block:
  ```yaml
  - set: { name: ok, value: false }
  - conditions:
      '{{x}}':
        - set: { name: ok, value: true }
  output:
    ok: '{{ok}}'   # NOT '{% {{x}} ? true : false %}'
  ```
  Ternaries ARE valid inside Custom Code `code: |` blocks and inline page `<script>` (real JS) — the ban applies only to DSUL `{% %}`/`{{ }}` expressions. (This bug shipped latent in `google-mail-consumer`'s `_runMcpToolWithAgent` + `_createTestDoc` helpers — every Agent Factory MCP test 500'd before asserting; the `*-consumer` helpers are the usual place it bites.)
- **`arguments: {}` or empty comment body** — YAML-parse-invalid schema. If the automation has no arguments, omit the key.
- **Hyphenated keys in expressions** — use `{{obj["key-name"]}}`, never `{{obj.key-name}}` (parsed as subtraction).
- **Dispatcher default branch** — only needed when you have a generic fallback (GraphQL dispatcher). Without it, set `result: null` and `!{{result}}` becomes an "Unknown tool" error — that's fine.
- **`type: array` in `mcpTools[].inputSchema` requires `items:`** — OpenAI/Azure function-calling strictly rejects array properties without `items` (error: `Invalid schema for function 'X': array schema missing items`). This only surfaces at LLM call time, NOT in `validate_automation`. Generated swagger specs often emit bare `type: array`; when you copy properties into mcpTools, add an `items:` schema (prefer `{type: string, enum: [...]}` for constrained values, else `{type: string}` as a safe default). Also apply to nested arrays under `items: {type: array, items: ...}`. Quick sanity check in Phase 4: `python3 -c "import yaml; d=yaml.safe_load(open('index.yml')); [print(f'MISSING items: {t[\"name\"]}.{pn}') for t in d['config']['value']['mcpTools'] for pn,ps in (t['inputSchema'].get('properties') or {}).items() if isinstance(ps,dict) and ps.get('type')=='array' and 'items' not in ps]"`
- **`json("")` crashes on HTTP 204 No Content** — DELETE endpoints (and some action endpoints) on virtually all REST APIs respond HTTP 204 with an empty body. `executeApiCall` propagates `response.body` (empty string) into `apiResult.data`, `method-*` copies it to `structuredData`, and `formatToolOutput` calls `{% json({{structuredData}}) %}` which fails with `InvalidExpressionSyntax: Invalid JSON syntax (json: "")`. The resulting HTTP 500 masks a successful API call — every DELETE looks broken even though the backend accepted it. The fix in `templates/helpers/formatToolOutput.yml` guards the `json()` call on `{{structuredData}}` truthy and falls back to a minimal `{operation, itemType, identifier, empty: true}` payload. Same pattern needed for any downstream helper that calls `json()` on data that might be empty — never call `json("")`. Same concern for `json(<plainString>)` in `handleApiError` when an API returns `{message: "401 Unauthorized"}` as a plain string — use bare `'{{value}}'` substitution, not `'{% json({{value}}) %}'`.
- **Swagger sub-agent hallucinates action-verb sub-resources** — when asked to generate a closeIssue/reopenIssue/approveX/mergeX/cancelX/retryX/revokeX/publishX operation, an LLM often invents paths like `/issues/{iid}/close` or `/merge_requests/{iid}/approve` because they "look natural." These sub-paths often don't exist — the real API typically uses a state-event pattern (`PUT /issues/{iid}` with `{state_event: close}` for GitLab) or a top-level action (`POST /resource/X/actions` for others). Symptom: the tool returns HTTP 404 on a happy-path call. Detection: grep methods for path segments that are action verbs (`/close`, `/reopen`, `/approve`, `/unapprove`, `/merge`, `/cancel`, `/retry`, `/revoke`, `/publish`, `/archive`, `/lock`) and validate each against the official API reference before Phase 6 ships. Always include this grep as a Phase 6 smoke-check step.
- **Swagger path-param names must match the real API exactly** — LLM sub-agents often simplify distinct path params to shared short names (GitLab's `merge_request_iid` / `issue_iid` / `note_id` collapsing to `iid`/`note_id` inconsistently). This works internally (tool → method forward the same name), but downstream LLM consumers typing the real API name into tool calls get 404 because the arg doesn't bind. Keep the exact names from the official API reference — `merge_request_iid`, `issue_iid`, `note_id`, `pipeline_id`, `job_id`, `hook_id`, etc. — in both the swagger path and the generated tool arguments. Phase 5 script should preserve swagger-sourced names verbatim; don't re-map.
- **OpenAI / Anthropic 128-tool hard cap** — `gpt-5-chat-latest` (and most production OpenAI models) reject `tools` arrays longer than 128 with `Invalid 'tools': array too long`. Anthropic Claude is more permissive but still loses precision past ~50 tools. Any API with >40 ops MUST use the entity-grouping pattern (Phase 5 default) — DO NOT generate 1 mcpTool per op. Symptom if missed: `tools/list` works in Prisme.ai but the downstream LLM call fails. Reference workspace: `tableau` (131 ops → 20 entity tools). Detection during Phase 5: count `len(swagger.paths)` — if >40, skip directly to entity grouping.
- **`MCP Core.syncMcpTools` overwrites the entity-grouped mcpTools array** — the sync routine scans `tool-*.yml` files on disk and rebuilds mcpTools from them. With the dispatcher pattern, only `tool-restOp.yml` and `tool-graphqlOp.yml` exist, so a single sync call replaces your 20 entity tools with 2 dispatcher tools — silent data loss recoverable only by re-pushing. **Never create `triggerSync.yml`. Never call `MCP Core.syncMcpTools` from any automation.** Push = sync. The mcpTools array is set once at install time via `imports/MCP Core.yml` and updated by re-importing the same file. (See memory entry `feedback_mcp_core_dispatcher_incompatible.md`.)
- **Cache key must be bumped when credentials/scopes change** — `buildAppAuth` caches the access token in `session.<service>[cacheKey]` for 1-3h. If you migrate credentials (PAT → JWT, OAuth2 → Connected App) or add a scope, sessions with a cached token signed under the OLD shape keep using it until TTL expires — your fix appears to "not work". Bump the cache prefix `session.<service>` → `session.<service>V2` when the credential schema or scopes change. Same applies inside Custom Code if you cache anything keyed on credentials. **The robust auto-recovery (canonical: `data-galaxy`)** — don't rely on a manual version bump: (1) key the cache on `tokenFingerprint(credential)` (Custom Code helper) so changing the credential auto-invalidates a stale token; (2) give `buildAppAuth` a `forceRefresh` arg that skips the cache + re-exchanges, returning `refreshed: true` on a fresh exchange; (3) in `executeApiCall`, on HTTP 401, call `buildAppAuth(forceRefresh: true)` and retry the request once when `freshAuth.refreshed`. Transparently recovers from a stale cached token in App mode; guarded no-op for static-token connectors (never `refreshed`) and MCP-central calls (no central credential). Symptom it cures: a connector that works in a fresh session but returns persistent `401 "token expired or invalid"` for an existing session after the credential was re-saved — the cached access token was exchanged from the old credential and kept until TTL.
- **JS comments in `code: |` blocks of Custom Code must use `//`, never `#`** — the Custom Code sandbox loads all `config.functions[].code` as a single JS module. A single `#` (from copy-pasted Python-style comments or untransformed template docs) is a JS syntax error that breaks the ENTIRE module, including unrelated functions like `generateSignedKey`. Symptom: `onInstall` completes but writes an error object as `mcpApiKey` instead of a signed key. The OAuth template fragment (`templates/oauth/imports/Custom-Code-oauth-fragment.yml`) historically had `#` comments inside `buildAuthorizeUrl.code: |` — verify after merging. Defense: `generateKey` should check `signedKey.error` and return HTTP 500 when Custom Code fails, and `onInstall` should validate `keyResponse.body.mcpApiKey` against `^[A-Za-z0-9_-]+\.[a-f0-9]{64}$` before merging and for its "already set" guard.
- **`type: array` in Custom Code function `parameters` is INVALID and silently breaks the whole module** — the Custom Code parameter type system only supports `string | number | object | boolean`. Declaring `type: array` causes the CC loader to reject the entire `imports/Custom Code.yml` module, so EVERY function (including unrelated ones like `generateSignedKey`) starts returning `ObjectNotFoundError: Object not found, path: /run/<funcName>` at runtime. Symptom on a fresh app+mcp: `onInstall` fails with `Key generation failed: Function generateSignedKey not found` even though the function is plainly visible in `get_app_instance_config`. Root-cause hunt is misleading because the broken parameter is usually on a _sibling_ function (we hit this on GitLab's `filterOAuthTools` having `tools: {type: array}` + `remove: {type: array}` — killed the whole module). **Fix: use `type: object` for list-shaped parameters.** JS code can iterate it as an array via `Array.isArray(x) ? x.filter(...) : ...` without any runtime change. Detection: the Phase 6 sanity-check script greps `imports/Custom Code.yml` for `type: array` under `config.functions.*.parameters.*` and flags every hit. (See memory entry `feedback_custom_code_no_array_type.md`.)

### OAuth-specific traps (only relevant if Phase 4.5 was run)

- **LLM loops on `connect` instead of data tools after successful OAuth** — the LLM reads a permissive connect description as "the way to sign in" and calls it on every request, even when the user is already authenticated. Symptom: agent keeps returning connect cards after a successful OAuth flow, never calling listX/getY. **Three mitigations, apply ALL THREE** (MCP clients cache the tool registry per session so filtering alone doesn't help existing sessions): (1) **Balanced descriptions** on `connect` — one bolded positive recommendation ("**Prefer calling data tools directly — they auto-handle authentication…**") plus a short usage condition. Anti-pattern: heavy `[RARE, USER-INITIATED ONLY]` + caps-locked "DO NOT" language over-filtered the LLM into not calling ANY tool, hallucinating OAuth setup guidance instead. See `templates/oauth/fragments/index-mcptools.yml`. (2) State-aware tools/list filter in `mcp.yml`: hide `connect` when `user.<<SERVICE_SLUG>>.oauth.authMethod == "delegated" && user.<<SERVICE_SLUG>>.oauth.tenantId == decoded.workspaceId`. The `filterOAuthTools` Custom Code function accepts `remove: [<names>]`; pass `[connect]` or `[connect, disconnect]`. (3) Server-side short-circuit in `method-connect.yml`: before computing the connect URL, check the same `user.<<SERVICE_SLUG>>.oauth` state — if already connected AND secret exists, return `{operation: action, message: "You are already connected…", structuredData: {alreadyConnected: true, …}}` and `break`. This is the ONLY defense that works for sessions with the OLD cached tool registry — teaches the LLM in-context that connect is a no-op.
- **Copying `google-mail`/`google-sheets` silently drops the per-tenant secret auto-wiring (Phase 4.5 step 6b)** — those two Google connectors predate it: their `onInstall.yml` has NO secret-provisioning block and their `imports/Custom Code.yml` lacks `makeConfigRef`/`makeSecretRef`. When you scaffold an OAuth connector by copying a reference workspace (the encouraged shortcut), copy **`google-docs`** instead, or port the provisioning from `templates/oauth/fragments/onInstall-provision.yml` + the two CC helpers. Detection: `grep -l secretsProvisioned automations/onInstall.yml` and `grep -l makeSecretRef "imports/Custom Code.yml"` must BOTH hit. Symptom if missed: the tenant admin has to hand-wire two config bindings (binding A on the instance + binding B in `config.value`) — which most won't — so `getConfig` returns an empty `oauth.clientId` and every data tool returns `needsConnect` forever, even after a successful browser OAuth. (Hit on `google-calendar` 2026-05-25: scaffolded by copying `google-mail`, shipped without the block, fixed by porting `google-docs`'s.)
- **`break: { scope: all }` does NOT exist in DSUL** — observed to propagate unexpectedly to the caller, so `mcp.yml` silently stops after `ensureAuthentication` and never reaches `routeToolCall`. The only valid scopes are `automation` (exits the current automation) and `repeat` (exits a loop). The OAuth template for `ensureAuthentication.yml` is written **flag-based** (`resolved: false` guard on each priority) with no `break` at all — keep it that way.
- **URLSearchParams / URL are not defined in Custom Code sandbox.** Use `encodeURIComponent` + manual `&`-joined string. See the `buildAuthorizeUrl` template.
- **Unresolved `{{secret.xxx}}` stays as a literal truthy string.** `getConfig.yml` must guard with `matches "{{"` and wipe the value — otherwise the PAT priority in `ensureAuthentication` wrongly short-circuits the OAuth branch. The OAuth template's `getConfig.yml` already has this guard.
- **Missing OBJECT fields stay as literal truthy strings too.** If `getConfig` doesn't emit an `oauth` field when OAuth isn't configured, `{{tenantConfig.oauth.clientId}}` in `ensureAuthentication` resolves to the literal template string (truthy) and wrongly triggers the `needsConnect` branch — even when the tenant only has a PAT. The OAuth `getConfig.yml` template always emits `oauth: {clientId: '', clientSecret: '', …}` as a default and only populates real values when both guarded credentials are present. Never return `oauth: undefined`.
- **DSUL `'{{a}} && {{b}}'` is unsafe when both sides may be empty.** After substitution the string becomes `' && '` which some DSUL expressions treat as a truthy non-empty string. Use **nested conditions** (`conditions: {{a}}:` → `conditions: {{b}}:`) instead of a single `&&` string when either side is a potentially-empty config value.
- **Providers without `expires_in` / `refresh_token` (Monday, similar).** The `oauthCallback.yml` template defaults `expires_in` to 315360000 (10 years) when absent — otherwise `expires_in * 1000` yields NaN, `expiresAt` is an invalid date, and every subsequent call looks "expired". Priority 1 in `ensureAuthentication.yml` fetches the stored access_token secret FIRST (no expiry check) and only attempts refresh if the secret is missing. If refresh also fails, it `delete`s `user.<slug>.oauth` so Priority 2 (tenant PAT) can take over. These three behaviours together make OAuth + PAT coexistence robust even for providers whose tokens never expire.
- **`session.tenantConfig[{{tenantId}}]` cache hides fixes.** The 10-min cache in `mcp.yml` means a freshly-pushed `getConfig` fix won't take effect on the next call if the user's session already has stale data. When debugging OAuth auth, either wait out the TTL, use a fresh session, or verify the bug reproduces with a never-cached session first. `user.*` variables persist across Prisme.ai logout/login for authenticated users, so a simple deco/reco does NOT reset them.
- **User-scope variables are per-workspace.** `user.connectorsVersion` set in this workspace is NOT visible to other workspaces (e.g. an upstream agent orchestrator in a different workspace). Cross-workspace cache invalidation needs a different mechanism — the bump is nonetheless useful for same-workspace consumers.
- **`window.close()` from inline page scripts can be blocked** on the legacy `*.pages.prisme.ai` host. The `connector-callback.yml` template doesn't auto-close — it posts a message and asks the user to close manually.
- **The `connector-callback` page must be labeled `public`, NOT `accessControl: public`** — the `security.yml` rule grants anonymous page read via `conditions: {labels: {$in: [public]}}`, so the page DSUL needs `labels:\n  - public`. `accessControl:` is not a recognized Page field and is silently ignored, so the page stays workspace-members-only. Symptom: the workspace OWNER sees the success page fine, but any other user (a colleague completing OAuth) hits the `pages` service **401**. Applies to EVERY OAuth connector. Fixed in `templates/oauth/pages/connector-callback.yml`; the older fleet connectors (`google-*`, `gitlab`, `gitlab-oauth`, `figma`, `monday`) still ship `accessControl: public` and need the relabel + re-push (hit 2026-05-25 on google-calendar).
- **Provider rotates refresh tokens.** Some providers issue a new refresh token on each refresh. `refreshOAuthToken.yml` writes the new one if present.
- **Webhook URLs accept `slug:<workspaceSlug>` too.** Prisme.ai's webhook URL format `/workspaces/{idOrSlugRef}/webhooks/{automationSlug}` accepts either the raw ID (`Ha9hyWW`) or `slug:<workspace-slug>` (e.g. `slug:monday`). **Always prefer the slug form** for any URL that points at the central workspace: it's readable, stable across workspace recreations, and survives a re-import to a fresh workspace ID. Hardcode `slug:<SERVICE_SLUG>` in every template that targets the central workspace — `initiateOAuth`, `oauthCallback`, `method-connect`, `mcp.yml` connectUrl, and the `generateKey` + `mcpEndpoint` URLs in `onInstall.yml`. The ONLY URL in `onInstall.yml` that must keep `{{global.workspaceId}}` is the `getConfigUrl`, because that one points back at the **tenant** workspace (where the app is being installed). Never hardcode a raw central-workspace ID like `_gwEr1h` in any template.

See `mcp-auto-install.md` (in this skill folder) for the full reasoning behind the auto-install flow and all its gotchas. For OAuth, see `./prismeai-workspaces/workspaces/gitlab-debug-oauth/README.md` (bare flow) and `google-docs` — its `onInstall.yml` + `makeConfigRef`/`makeSecretRef` CC helpers are the canonical reference for the per-tenant secret auto-wiring (copy THAT one, not `google-mail`/`google-sheets`).

---

## Reference workspaces

When in doubt, list `prismeai-workspaces/workspaces/` and read the closest-matching existing workspace. Match by API shape, not by name:

| Target API shape                                                     | Look for a reference workspace with                                                                                                                                                                                                                                                           |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| REST-RPC, static token                                               | Flat operation set, `Authorization: Bearer` header                                                                                                                                                                                                                                            |
| Pure GraphQL, static token                                           | Single `/graphql` endpoint, one `Authorization` header                                                                                                                                                                                                                                        |
| Hybrid REST + GraphQL, OAuth2 client-credentials                     | JWT exchange with session cache, generic GraphQL dispatcher                                                                                                                                                                                                                                   |
| OAuth2 with token refresh (service-to-service)                       | Session-cached refresh flow                                                                                                                                                                                                                                                                   |
| **OAuth2 authorization-code (user-delegated)**                       | **`google-docs`** — canonical reference for Phase 4.5 **with per-tenant secret auto-wiring** (its `onInstall.yml` provisions placeholder secrets + writes both config bindings; validated E2E). Also `gitlab-debug-oauth` for the bare OAuth flow + gotchas. ⚠️ Do NOT copy `google-mail`/`google-sheets` for an OAuth connector — they predate the auto-wiring (no `onInstall` provisioning block, no `makeConfigRef`/`makeSecretRef`).                                                                                                                                                                           |
| Basic auth, rich REST                                                | `Basic base64(email:token)` header                                                                                                                                                                                                                                                            |
| **Large API (>40 ops), entity-grouped MCP tools, JWT Connected App** | **`tableau`** — the canonical reference for the entity-grouping pattern (Phase 5 default). 131 ops compressed to 20 entity tools via `ENTITY_OPS` + `resolveToolAction` + `routeToolCall` + 4 generic dispatchers. Custom Code header `X-Tableau-Auth`, JWT signed in Custom Code with HS256. |

Run `Read` on the matching workspace before starting phase 4, especially on the helpers and `imports/Custom Code.yml`. The skill templates here are deliberately minimal — the existing workspaces contain all the nuanced cases.

---

## Output format to the user

At the end of each phase, summarise in ≤5 bullets what was done and what's next. Never dump entire files — reference them by path.

When the whole skill finishes, produce a final summary:

- Paths of every created file (grouped by purpose)
- Number of tools / public instructions / helpers
- Deploy command that was run
- Next actions for the user (secrets to configure, how to install, how to test)
