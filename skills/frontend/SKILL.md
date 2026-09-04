---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: how the UI resolves which
  server URL to connect to (same-origin vs an explicit server override, and the dev proxy
  that makes them equivalent), Kinotic.connect, the three VITE_KINOTIC_* build variables
  the deployment sets, browser sign-in through the gateway's login routes and the session
  cookie it establishes, BasicCredentialsResolver and BearerCredentialsResolver for Bun
  and Node clients, machine identities via environment credentials, OIDC login, creating
  application users, and invoking published services and entity repositories from the UI.
  Use when building a Kinotic app frontend or SPA, working out what host or URL the UI
  should point at, wiring up login or authentication, or connecting any client to a
  Kinotic server.
---

# Frontends and Clients

Every client — browser SPA, Node client, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth (`/api/auth/*`) and MCP (`/mcp`). Frontends
live in `packages/ui` of the project scaffold and may use any framework that builds to
static assets. Docs: <https://kinotic.ai/apps/security/authentication>.

A UI under `packages/ui` **is** built and published by the deployment and served at its own
site — see "The UI build contract" below for what the build is handed and what it must
produce.

Authentication happens at the **WebSocket upgrade**, and every authentication carries a
scope. Application users always authenticate at APPLICATION scope — both
`organizationId` and `applicationId` are supplied (directly or via JWT claims). Scope
isolation means a user of one application can never reach another application, even
with the same identity provider.

An application-scope connection may **call** services in its own
`app.<org>.<app>` zone and use entity repositories (`app-api`), which is everything a
frontend needs. It cannot **host** a `@Publish` service — that requires organization
scope, and is what the deployment's runtime does (see the `services` and `deploying`
skills).

`Kinotic.connect(options?)` takes an optional plain `ConnectOptions` literal; every
field is optional. Unset server fields resolve in order: explicit value →
`KINOTIC_SERVER_HOST` / `KINOTIC_SERVER_PORT` / `KINOTIC_SERVER_USE_SSL` → the browser's
own location → `https://api.kinotic.ai`. Credentials resolve through a resolver chain:
the environment variables, then the browser session.

## How the UI knows where the server is

Answer this before writing any connect code — it decides whether the frontend needs
configuration at all, and it is the question the resolution ladder above does not settle
on its own.

**The env-var rung does not exist in a browser.** Core reads it as
`typeof process !== 'undefined' ? process.env : undefined`, and bundlers do not provide
`process` in browser builds (Vite exposes `import.meta.env` instead). So putting
`KINOTIC_SERVER_HOST` in a `.env` does nothing for an SPA — the ladder skips straight
from an explicit `server` option to `window.location`. That leaves exactly two shapes:

**Cross-origin — pass `server` explicitly.** This is the platform's path. A published UI
is served from its own site host, not the gateway, and the deployment sets
`VITE_KINOTIC_HOST`, `VITE_KINOTIC_PORT` and `VITE_KINOTIC_USE_SSL` for the build; `vite dev`
takes the same three from the app's `.env` (next section). Left alone, the client would open
a socket against the page's own origin, so `connect()` is handed a `server` built from those
variables, and the REST calls are built from the same values and sent with
`credentials: 'include'` so the session cookie travels. Recipe 1's `serverOptions()` /
`apiUrl()` are that code, and the platform's own consoles connect the same way. Nothing is
hand-set, and the code does not change between local and published.

**Same-origin — configure nothing.** The alternative, for a UI served from the gateway's own
webroot or from a dev server that proxies to it. `window.location` is already the right
answer, so a bare `Kinotic.connect()` and bare REST paths
(`fetch('/api/auth/me', { credentials: 'include' })`) are correct, and the three variables
stay unset. A Vite dev proxy has to forward both the REST and the WebSocket paths, and `/v1`
needs `ws: true` — that is the broker path the STOMP client opens:

```typescript
// vite.config.ts
proxy: {
    '/api': { target: 'http://localhost:58503', changeOrigin: true },
    '/v1':  { target: 'http://localhost:58503', changeOrigin: true, ws: true }
}
```

Cross-origin does not push the SPA off the session cookie. The cookie is `SameSite=Lax`,
which is sent whenever the API and the site share a site — as `api.kinotic.ai` and
`apps.kinotic.ai` do in production. A published UI therefore uses the cookie flow
(Recipe 1), cross-origin or not.

## The UI build contract

Every UI package under `packages/ui` that declares a `build` script is built during the
deployment with `bun run build` and published to its own site. The build is handed exactly
three variables — the same three the platform's own consoles use — and Vite exposes them to
the page on its own:

| Variable | Value | What the UI does with it |
|---|---|---|
| `VITE_KINOTIC_HOST` | e.g. `api.kinotic.ai` | The host the UI connects to Kinotic on from the browser |
| `VITE_KINOTIC_PORT` | e.g. `443` | Its port |
| `VITE_KINOTIC_USE_SSL` | `true` or `false` | Whether to connect over TLS |

A Vite project needs **no `vite.config.ts` changes** to be published — no `base`, no
`define`. A UI on another build tool must pass the three through to the page itself. The
deployment hands the build no base path and no commit, and the publish uploads `dist` under
the site's root as it is.

Declare the UI's ambient typing once, so `import.meta.env.VITE_KINOTIC_*` type-checks
(Vite's `vite/client` types merge with it):

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
    readonly VITE_KINOTIC_HOST?: string
    readonly VITE_KINOTIC_PORT?: string
    readonly VITE_KINOTIC_USE_SSL?: string
}
```

Files under `assets/` carry Vite's content hash in their name and are cached for a year;
everything else, `index.html` included, is never cached. A file outside `assets/` that
changes between publishes is therefore safe.

Each site publishes `version.json` (`{ "commitSha": "<commit>" }`) next to `index.html`.
The platform reads it to decide when a site is ready. A publish deletes the previous
commit's files.

A build that leaves no `dist/index.html` **fails the deployment**, naming the UI — so a UI
whose build writes elsewhere never publishes.

A package under `packages/ui` **without** a `build` script is treated as a library and is
never published — that is how shared component packages live there.

For `vite dev` on a developer's machine the same three variables go in the app's `.env`,
pointing at the local server:

```
VITE_KINOTIC_HOST=localhost
VITE_KINOTIC_PORT=58503
VITE_KINOTIC_USE_SSL=false
```

The code does not change between local and published. The gateway sets the session cookie
`Secure` (it is a `__Host-` cookie), so signing in over plain `http://localhost` relies on
the browser treating localhost as a secure context: Chrome and Firefox do; Safari does not
store the cookie, and the upgrade after a `204` login arrives unauthenticated.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on. Only a **Node** process sending credentials must first call
`ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which swaps in `ws` because Node's
own WebSocket ignores a headers option. It is a no-op under Bun and unnecessary in the
browser — do not add it to a Bun entry point.

## Recipe 1 — Browser SPA (session cookie)

A browser **cannot** set WebSocket upgrade headers. `BasicCredentialsResolver` puts the
credentials in upgrade headers, which only the Node and Bun sockets accept, so in a browser
it produces an unauthenticated upgrade and `Max number of reconnection attempts reached`.
In a browser the session cookie is the credential, and the gateway's login route
establishes it.

- The app-scope password login is `POST /api/auth/app/:orgId/:appId/login` with
  `{ email, password }`. It answers `204` and sets the cookie, or `4xx` with
  `{ "error": "…" }`. The user must exist in **that application's** user base (portal,
  Application → Members); the route does not authenticate organization members.
- `GET /api/auth/me` answers `204` when the browser holds a live session and `401`
  otherwise. `POST /api/auth/logout` ends the session.
- Every request carries `credentials: 'include'`, and `Kinotic.connect` takes only the
  server: the STOMP upgrade rides the cookie.

```ts
// src/kinotic/session.ts
//
// Sign-in for a Kinotic app UI. The deploy builds the UI with three variables naming the
// platform it was published against, and Vite exposes them to the page on its own:
//   VITE_KINOTIC_HOST     e.g. api.kinotic.ai
//   VITE_KINOTIC_PORT     e.g. 443
//   VITE_KINOTIC_USE_SSL  "true" | "false"
// For `vite dev` on a laptop, put the same three in the app's .env, pointing at the local
// server (localhost, 58503, false).
//
// A browser cannot set WebSocket upgrade headers, so credentials never go to Kinotic.connect
// here. The gateway's login route establishes the session cookie, and the STOMP upgrade rides
// that cookie; connect() takes only the server.

import { Kinotic, type ServerInfo } from '@kinotic-ai/core'

const ORGANIZATION_ID = 'my-organization'
const APPLICATION_ID = 'my-application'

/** The platform this UI was built against. */
export function serverOptions(): ServerInfo {
    const host = import.meta.env.VITE_KINOTIC_HOST
    if (!host) {
        throw new Error('VITE_KINOTIC_HOST is not set: the build was not handed the platform address')
    }
    const useSSL = import.meta.env.VITE_KINOTIC_USE_SSL === 'true'
    const port = parseInt(import.meta.env.VITE_KINOTIC_PORT || (useSSL ? '443' : '80'))
    return { host, port, useSSL }
}

/** Absolute URL of a gateway REST path on that platform. */
export function apiUrl(path: string): string {
    const { host, port, useSSL } = serverOptions()
    return `${useSSL ? 'https' : 'http'}://${host}:${port}${path.startsWith('/') ? path : `/${path}`}`
}

/**
 * Signs in with an application user's email and password, then opens the connection.
 * Rejects with the gateway's message ("Invalid credentials" when it gives none).
 */
export async function login(email: string, password: string): Promise<void> {
    const res = await fetch(apiUrl(`/api/auth/app/${ORGANIZATION_ID}/${APPLICATION_ID}/login`), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',            // stores the Set-Cookie from the API's origin
        body: JSON.stringify({ email, password }),
    })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Invalid credentials'))
    }
    await connect()
}

/**
 * Reconnects a returning tab: true when the browser still holds a live session and the
 * connection is open, false when the user must sign in. Rejects when the session is live
 * but the connection cannot be opened, so a caller can tell the two apart.
 */
export async function resume(): Promise<boolean> {
    const res = await fetch(apiUrl('/api/auth/me'), { credentials: 'include' })
    if (!res.ok) {
        return false
    }
    await connect()
    return true
}

/** Closes the connection and ends the browser session. */
export async function logout(): Promise<void> {
    await Kinotic.disconnect(true)
    await fetch(apiUrl('/api/auth/logout'), { method: 'POST', credentials: 'include' })
}

async function connect(): Promise<void> {
    await Kinotic.connect({
        server: serverOptions(),
        // bounded so a session the gateway no longer honors fails fast instead of retrying
        // the upgrade forever with no feedback on the login button
        maxConnectionAttempts: 3,
    })
}

async function errorMessage(res: Response, fallback: string): Promise<string> {
    try {
        const body = await res.json() as { error?: string }
        return body.error ?? fallback
    } catch {
        return fallback
    }
}
```

One condition applies outside production. The cookie is `SameSite=Lax`, sent when the API
and the site share a site — which production does. An environment whose API sits on a
tunnel origin (ngrok) while the site is under `apps-<environment>.kinotic.ai` needs
`kinotic.apiGateway.sessionCookieSameSite: NONE` in the server's profile, which sends the
cookie from any origin the CORS pattern admits.

## Recipe 2 — Machine identity, zero ceremony (environment credentials)

The default credential resolver chain reads the environment, so a script or local
process normally passes nothing:

```bash
KINOTIC_CLIENT_ID=<machine identity id> KINOTIC_CLIENT_SECRET=<secret> \
bun run src/main.ts
```

```typescript
import { Kinotic } from '@kinotic-ai/core'

await Kinotic.connect()
```

Create the machine in the portal: **Application → Machines → create machine**. The
secret is disclosed exactly once (only a hash is stored), so it can be rotated but never
re-read. Machines created there are **application scope** — they call services and use
repositories, and cannot publish a service. Optionally `KINOTIC_ORGANIZATION_ID` /
`KINOTIC_APPLICATION_ID` pin the scope explicitly; `KINOTIC_TOKEN` resolves to bearer
auth instead. A `clientId` containing `@` is a user email; anything else resolves only
to a machine identity. When nothing resolves, connect fails with
`No Kinotic credentials found; consulted: <resolvers>`.

Do not borrow a project deployment's machine secret for this — rotating it cuts the
running deployment off.

## Recipe 3 — explicit credentials in code

For **Bun and Node clients** — scripts, tests, machines. Both resolvers below put the
credentials in WebSocket upgrade headers, which a browser cannot send; a browser signs in
through Recipe 1.

Email/password (an application user):

```typescript
import { BasicCredentialsResolver, Kinotic } from '@kinotic-ai/core'

await Kinotic.connect({
    credentials: new BasicCredentialsResolver('user@example.com', 'password', 'my-organization', 'my-application')
})
```

The arguments are `(clientId, clientSecret, organizationId?, applicationId?)` — scope
is structural: neither id → SYSTEM, organization only → ORGANIZATION, both →
APPLICATION. (In Node, call `ensureNodeWebSocket()` first; on Bun, don't.)

Bearer token (OIDC):

```typescript
import { BearerCredentialsResolver, Kinotic } from '@kinotic-ai/core'

await Kinotic.connect({ credentials: new BearerCredentialsResolver(async () => await getToken()) })
```

The supplier runs before every (re)connect, so short-lived tokens refresh
automatically. A Kinotic-minted JWT already carries the organization/application
claims — no extra scope headers.

## Application users and OIDC login

Application-scope users are **pre-created** by an administrator — an OIDC identity alone
does not grant access. The organization admin manages them on the portal's **Members**
page, scoped to the application: local users get an email and password, OIDC users get
an email only and the subject links automatically on first login.

For OIDC:

1. An admin enables OIDC configurations (Google, Microsoft, a corporate IdP) on the
   application. To see what is enabled, call the MCP tool titled
   `Application Service Get Oidc Configurations` with `{"applicationId": "..."}`
   (Kinotic MCP tool names are opaque hashes — always resolve by title from the tool
   listing). An empty result means no provider is wired up yet — tell the user rather
   than writing login code against a provider that will reject it.
2. The frontend runs the OAuth flow; the REST login establishes the session cookie;
   then connect as in Recipe 1.

## Calling services from the frontend

Once connected, the frontend calls published services exactly like any client — the
hand-written proxy pattern from the services skill (note the `~` between zone and
qualified name):

```typescript
const proxy = Kinotic.serviceProxy('app.my-organization.my-application~com.example.OrderService')
const orders = await proxy.invoke('findOpenOrders', [customerId])
proxy.invokeStream('watchOrder', [orderId]).subscribe(update => render(update))
```

A proxy call reaches a service only while that service is running — which, for an
application's own services, means the project has been deployed. If calls fail to
resolve, check the deployment before debugging the address (`deploying` skill).

Generated entity repositories (entities-and-persistence skill) also work in the
browser after `Kinotic.use(PersistencePlugin)` — data access needs no bespoke backend
endpoints.

## Further reading

- Authentication: <https://kinotic.ai/apps/security/authentication>
- Service proxies: <https://kinotic.ai/apps/services/service-proxies>
- Platform auth model: <https://kinotic.ai/platform/system-security>
