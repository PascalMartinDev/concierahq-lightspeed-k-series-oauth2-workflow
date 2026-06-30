# ConcieraHQ Lightspeed K-Series OAuth2 Authorization Code Workflow

> **Lightspeed powers the transaction. ConcieraHQ powers the relationship.**

> This is **workflow 1 of 3** in the Lightspeed K-Series integration. See the
> [project README](./README.md) for the big picture, and the
> [Access Token Refresh](./TokenRefreshWorkflow.md) and
> [Create Customer in POS](./PosAddCustomer.md) workflows for what happens once this handshake
> hands tokens off to the tenant account.

This document describes the **OAuth 2.0 Authorization Code flow** that connects a hospitality
venue's **Lightspeed Hospitality K-Series** account to **ConcieraHQ**, so that customer profiles,
orders, items and other performance metrics can be securely synced into ConcieraHQ's
customer-intelligence platform.

Lightspeed K-Series is a **confidential client**: the authorization code is exchanged for tokens
using the client secret (HTTP Basic auth), so there is **no PKCE `code_verifier`**. Instead, a
single-use **opaque nonce** is carried in the `state` parameter — it provides CSRF/replay
protection and lets the callback recover its routing context server-side, so tenant identifiers
never travel in the URL.

> ℹ️ **Naming note:** the DynamoDB table is called `PKCE_Table` for historical reasons. It now
> stores **nonce → routing-context** records, not PKCE verifiers.

The integration is implemented as a **serverless, multi-tenant architecture on AWS**, with strict
separation between the tenant-facing application account and the service account that brokers the
OAuth handshake and holds the client credentials.

<p align="center">
  <img src="./assets/diagrams/ConcieraHQ-KSeries-Integration-Architecture.png" alt="ConcieraHQ Lightspeed K-Series OAuth2 architecture" width="100%">
</p>

---

## Table of Contents

- [Overview](#overview)
- [Architecture at a glance](#architecture-at-a-glance)
- [The OAuth2 flow](#the-oauth2-flow)
  - [Phase 1 — Authorization initiation](#phase-1--authorization-initiation)
  - [Phase 2 — Callback &amp; token exchange](#phase-2--callback--token-exchange)
  - [Phase 3 — Token storage &amp; scheduled refresh](#phase-3--token-storage--scheduled-refresh)
  - [Failure handling](#failure-handling)
- [Sequence diagram](#sequence-diagram)
- [Data contracts](#data-contracts)
- [AWS components](#aws-components)
- [User experience](#user-experience)
- [Security model](#security-model)
- [Configuration](#configuration)
- [Support](#support)

---

## Overview

To connect a venue safely, ConcieraHQ never asks for its Lightspeed password. The venue authorises
access from inside their own Lightspeed back office, and ConcieraHQ only ever holds short-lived,
revocable tokens scoped to the permissions the venue approved. This authorization handshake is
brokered centrally and, on success, hands those tokens **cross-account** to the venue's own AWS
account — where the [refresh](./TokenRefreshWorkflow.md) and [POS-sync](./PosAddCustomer.md)
workflows take over. See the [project README](./README.md#overview) for the wider value proposition.

---

## Architecture at a glance

This handshake spans the hardened **service account**, the venue's **own tenant AWS account**, and
the external **Lightspeed K-Series** authorization server (a Keycloak-style OpenID Connect provider
at `auth.lsk-{stage}.app/realms/k-series`):

| Boundary | Role in this workflow |
| --- | --- |
| **ConcieraHQ Service Account — Lightspeed K-Series** | The OAuth broker: API endpoints, initiation/callback Lambdas, nonce/routing state, and client-secret storage. |
| **ConcieraHQ Application — Tenant Account** *(one per venue)* | Starts the connection from the Admin Portal and receives the tokens; afterwards owns long-term token storage and the venue's own refresh machinery. |
| **Lightspeed K-Series (Tenant Back office)** | The external authorization & token server where the venue grants access. |

The split keeps the OAuth **client secret** and the live handshake inside the hardened service
account, while the resulting per-tenant tokens are written **cross-account** into the venue's own
account that consumes them. `stage` is either **`demo`** (trial accounts) or **`prod`**
(production). The full project architecture is in the [README](./README.md#architecture-at-a-glance).

---

## The OAuth2 flow

### Phase 1 — Authorization initiation

1. The venue logs into the **ConcieraHQ Admin Portal**, opens **Settings → Integrations**, and
   clicks **Connect** on the Lightspeed Hospitality K-Series card.


2. The portal sends a `POST` to the service-account endpoint **`{SERVICE-ACCOUNT-URL}/initiation`**
   with the tenant routing context:

   ```json
   {
     "initiation_payload": {
       "tenantId": "string",
       "awsAccountId": "string",
       "region": "string"
     }
   }
   ```

3. The **`handle-oauth-initation`** Lambda (`OAuthInitiator`):
   - fetches the Lightspeed **`client_id`** from **Secrets Manager**;
   - mints a random **`nonce`** (`uuid4`);
   - writes a routing record to the **`PKCE_Table`** keyed `PKCE#{nonce}`, with a **600 s TTL**
     that bounds the flow window;
   - encodes `{ "nonce": ... }` as a **base64url** `state` string (no padding);
   - builds the Lightspeed authorize URL (comma-separated scopes become space-delimited).
4. It returns `200 application/json` with the URL the portal redirects the venue to:

   ```json
   {
     "authUrl": "https://auth.lsk-{stage}.app/realms/k-series/protocol/openid-connect/auth?response_type=code&client_id=...&redirect_uri=...&scope=...&state=..."
   }
   ```

> 🔒 The routing context (`tenantId`, `awsAccountId`, `region`) is **never** placed in the URL —
> only the opaque nonce travels in `state`. The full auth URL is intentionally **not logged**, to
> keep `state` out of CloudWatch.

### Phase 2 — Callback & token exchange

5. The venue signs in to Lightspeed and approves the requested permissions. Lightspeed redirects
   back to the registered **Redirect URI** at **`{SERVICE-ACCOUNT-URL}/lsk`** with `code` and `state`
   query parameters.
6. The **`handle-oauth-callback`** Lambda (`StateManager`) decodes `state`, extracts the nonce, and
   looks up `PKCE#{nonce}` in `PKCE_Table` to recover `tenant_id`, `aws_account_id` and `region`.
   - The record's **TTL is validated explicitly** — DynamoDB TTL deletion is best-effort and can
     lag for hours, so an expired nonce is rejected rather than trusted.
7. The **`Authorisation`** service exchanges the code for tokens against Lightspeed's token
   endpoint, authenticating as a confidential client:
   - `POST https://auth.lsk-{stage}.app/realms/k-series/protocol/openid-connect/token`
   - `Authorization: Basic base64(client_id:client_secret)` (secret pulled from Secrets Manager)
   - body: `grant_type=authorization_code`, `code`, `redirect_uri`
   - bounded by a **3 s connect / 10 s read** timeout so a hung endpoint can't stall the Lambda.
8. On success Lightspeed returns an **access token** and **refresh token**. Over AWS's secure
   internal network and **cross-account access**, these are written directly into the
   **Access &amp; Refresh Token storage** (DynamoDB) in the **tenant account**, and the refresh
   **scheduler is armed**.
9. Only after the token exchange and the cross-account hand-off succeed does the callback
   **consume (delete) the nonce record**, closing the replay window immediately. A failed flow
   leaves the nonce reusable for retry.

### Phase 3 — Token storage & scheduled refresh

10. Once the tokens land in the tenant account, that account's own **EventBridge Scheduler** arms
    a Step Functions workflow that rotates the access token roughly **every 23 minutes**, so the
    connection stays live with no further action from the venue.

This refresh lifecycle runs entirely inside the venue's own account and is documented separately in
the **[Access Token Refresh Workflow](./TokenRefreshWorkflow.md)**. From there, the
**[Create Customer in POS Workflow](./PosAddCustomer.md)** shows how the kept-valid token is used to
write data into Lightspeed.

### Failure handling

If any step of the workflow fails, the **tenant is notified**. User-facing errors stay generic
(e.g. *"Invalid state parameter"*, *"An unexpected error occurred"*) while specific diagnostics
(missing fields, misconfigured environment variables, upstream failures) are sent to **CloudWatch
only** via the error `detail`, so internal schema is never leaked to the browser.

---

## Sequence diagram

```mermaid
sequenceDiagram
    autonumber
    actor Venue as Venue (Admin Portal)
    participant GWInit as API GW<br/>/initiation
    participant Init as λ handle-oauth-initation
    participant Secrets as Secrets Manager
    participant Nonce as PKCE_Table<br/>(nonce → routing ctx)
    participant LS as Lightspeed K-Series
    participant GWcb as API GW<br/>/lsk (Redirect URI)
    participant CB as λ handle-oauth-callback
    participant Tokens as Token Storage<br/>(tenant account)
    participant Sched as EventBridge Scheduler
    participant Refresh as Step Functions<br/>Token Refresh

    Note over Venue,Nonce: Phase 1 — Authorization initiation
    Venue->>GWInit: POST initiation_payload<br/>(tenantId, awsAccountId, region)
    GWInit->>Init: invoke
    Init->>Secrets: read client_id
    Init->>Nonce: store PKCE#{nonce} + routing ctx (TTL 600s)
    Init-->>Venue: 200 { authUrl } → redirect to /auth

    Note over Venue,Tokens: Phase 2 — Callback & token exchange
    Venue->>LS: Sign in & approve permissions
    LS-->>GWcb: Redirect URI ?code & state
    GWcb->>CB: invoke
    CB->>Nonce: decode state → lookup PKCE#{nonce}, validate TTL
    CB->>Secrets: read client_id:client_secret
    CB->>LS: POST /token (Basic auth, authorization_code)
    LS-->>CB: access_token + refresh_token
    CB->>Tokens: cross-account write tokens + arm scheduler
    CB->>Nonce: delete nonce (single-use, anti-replay)

    Note over Sched,Tokens: Phase 3 — Scheduled refresh (~23 min)
    Sched->>Refresh: trigger on schedule
    Refresh->>Tokens: read refresh token (Access/Update)
    Refresh->>LS: POST /token (refresh grant)
    LS-->>Refresh: new access token
    Refresh->>Tokens: write rotated tokens
```

---

## Data contracts

**Initiation request** → `POST /initiation`

```json
{ "initiation_payload": { "tenantId": "string", "awsAccountId": "string", "region": "string" } }
```

**Initiation response** → `200 application/json`

```json
{ "authUrl": "https://auth.lsk-{stage}.app/realms/k-series/protocol/openid-connect/auth?response_type=code&client_id=...&redirect_uri=...&scope=...&state=..." }
```

**`state` parameter** — base64url-encoded JSON, no padding

```json
{ "nonce": "uuid4" }
```

**`PKCE_Table` record** (nonce → routing context)

```text
pk             = "PKCE#{nonce}"     # nonce is a uuid4
tenant_id      = "..."              # routing context (snake_case)
aws_account_id = "..."
region         = "..."
ttl            = now + 600s         # bounds the flow window
```

**Token response** from Lightspeed `/token` → `200 OK`

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "..."
}
```

---

## AWS components

| Component | AWS service | Account | Role in the flow |
| --- | --- | --- | --- |
| ConcieraHQ Admin Portal | Amplify | Tenant | Venue-facing UI that starts the connection |
| `{SERVICE-ACCOUNT-URL}/initiation` | API Gateway | Service | Entry point for the authorization request |
| API Authoriser | Lambda | Service | Authorises inbound initiation requests |
| `handle-oauth-initation` | Lambda | Service | Mints nonce, stores routing ctx, builds auth URL |
| `PKCE_Table` | DynamoDB | Service | Nonce → routing-context store (name is historical) |
| Secure Storage Client ID/Secret | Secrets Manager | Service | Holds the OAuth client credentials |
| `{SERVICE-ACCOUNT-URL}/lsk` | API Gateway | Service | OAuth **Redirect URI** callback endpoint |
| `handle-oauth-callback` | Lambda | Service | Validates state, exchanges code for tokens |
| Tenant Access Permissions | IAM | Service | Cross-account access into the tenant token store |
| Access &amp; Refresh Token storage | DynamoDB | Tenant | Persists per-tenant access/refresh tokens |
| Scheduler | EventBridge | Tenant | Triggers periodic refresh (~23 min) |
| AccessToken Refresh | Step Functions | Tenant | Orchestrates the refresh workflow |
| Token Refresh | Lambda | Tenant | Rotates tokens against Lightspeed |

---

## User experience

The whole handshake is two clicks for the venue.

**1. Integrations page** — the venue opens **Settings → Integrations** and clicks **Connect** on the
Lightspeed Hospitality K-Series card.

<p align="center">
  <img src="./assets/diagrams/ConcieraHQ-Admin-Integrations-Lightspeed-Connect.png" alt="ConcieraHQ Integrations page with Lightspeed K-Series Connect button" width="100%">
</p>

**2. Connect screen** — ConcieraHQ explains what will happen, then **Authorise Lightspeed** kicks off
Phase 1. The venue is redirected to Lightspeed's secure login, approves the requested permissions,
and is redirected straight back — connected automatically.

<p align="center">
  <img src="./assets/diagrams/ConcieraHQ-Admin-Integration-Lightspeed-Connect-To-K-Series.png" alt="Connect ConcieraHQ to Lightspeed K-Series screen" width="100%">
</p>

**3. Confirmation screen** — Upon either success or failure the tenant client will see a confirmation message. 

<p align="center">
  <img src="./assets/diagrams/ConcieraHQ-Admin-Integration-Lightspeed-Connection-Successful.png" alt="Successfully connected ConcieraHQ to Lightspeed K-Series screen" width="100%">
</p>



---

## Security model

- **Confidential client** — the code is exchanged using the client secret over HTTP Basic auth;
  there is no PKCE verifier to intercept.
- **Opaque single-use nonce** — the routing context stays server-side and never leaks via the URL,
  browser history or referer; consuming the nonce on success gives CSRF/replay protection.
- **Client secret isolation** — `client_id`/`client_secret` live only in **Secrets Manager** in the
  single dedicated Lightspeed service account, never in code. Tenant accounts that later refresh the
  token read the secret via **cross-account access** at call time and never persist their own copy.
- **Account separation** — the live OAuth handshake (one shared service account) is isolated from
  long-term token storage and consumption (each venue's own tenant account); tokens move via secure
  cross-account access.
- **Bounded flow window** — nonce records carry a 600 s TTL, validated explicitly on callback
  because DynamoDB TTL deletion is best-effort and can lag.
- **No state in logs** — the full auth URL is never logged, keeping `state` out of CloudWatch.
- **Generic user errors** — browsers see generic messages; specific diagnostics go to CloudWatch
  via the error `detail` only.
- **Bounded upstream calls** — token exchange uses a 3 s connect / 10 s read timeout so a hung
  Lightspeed endpoint can't stall the Lambda.
- **Token rotation** — access tokens are refreshed on a schedule (~23 min), limiting the lifetime
  of any single token.

---

## Configuration

Environment variables consumed by the service-account Lambdas in this workflow:

| Key | Description |
| --- | --- |
| `PKCE_TABLE` | DynamoDB table storing nonce → routing-context records |
| `REDIRECT_URI` | OAuth redirect URI registered with Lightspeed (`https://{SERVICE-ACCOUNT-URL}/lsk`) |
| `LSK_SCOPES` | Comma-separated OAuth scopes, e.g. `financial-api,orders-api,items,offline-access` |
| `LSK_APP_STAGE` | Lightspeed environment slug — `demo` or `prod` — embedded in the auth/token host |

The `LSK_APP_STAGE` → host mapping is shared across all three workflows and is documented in the
[README](./README.md#environments). The Lightspeed **`client_id` / `client_secret`** are **not**
environment variables — they are read at runtime from **Secrets Manager**, with `LSK_APP_STAGE`
selecting the demo vs prod credential set.

---

## Support

ConcieraHQ is developed by **Konnectit.io**.
For support, email **dev@konnectit.io**.
