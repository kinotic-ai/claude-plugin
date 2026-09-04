---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: how the UI resolves which
  server URL to connect to (same-origin vs an explicit server override, and the dev proxy
  that makes them equivalent), Kinotic.connect, browser session-cookie authentication,
  BasicCredentialsResolver and BearerCredentialsResolver, machine identities via
  environment credentials, OIDC login, creating application users, and invoking published
  services and entity repositories from the UI. Use when building a Kinotic app frontend
  or SPA, working out what host or URL the UI should point at, wiring up login or
  authentication, or connecting any client to a Kinotic server.
---

# Frontends and Clients

Every client — browser SPA, Node client, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth (`/api/auth/*`) and MCP (`/mcp`). Frontends
live in `packages/ui` of the project scaffold and may use any framework that builds to
static assets. Docs: <https://kinotic.ai/apps/security/authentication>.

A UI under `packages/ui` **is** built and published by the deployment and served at its own
site — see "The UI build contract" below, which every UI must honor or it 404s on its own
assets.

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

**Same-origin — configure nothing.** The SPA is served from the gateway itself, or a dev
server proxies to it. `window.location` is already the right answer, so a bare
`Kinotic.connect()` is correct. A Vite dev proxy has to forward both the REST and the
WebSocket paths, and `/v1` needs `ws: true` — that is the broker path the STOMP client
opens:

```typescript
// vite.config.ts
proxy: {
    '/api': { target: 'http://localhost:58503', changeOrigin: true },
    '/v1':  { target: 'http://localhost:58503', changeOrigin: true, ws: true }
}
```

**Cross-origin — pass `server` explicitly.** The SPA runs on its own host (a framework
dev server with no proxy, static hosting, any origin that is not the gateway). Left
alone, the client would try to open a socket against the page's own origin. Build the
override from your bundler's build vars and hand it to `connect()`:

```typescript
import { Kinotic } from '@kinotic-ai/core'
import type { ServerInfo } from '@kinotic-ai/core'

// Empty when no host is configured, so core falls back to the page's location and the
// same build still works served same-origin.
function serverOverrides(): Partial<ServerInfo> {
    const host = import.meta.env.VITE_KINOTIC_HOST
    if (!host) return {}
    return {
        host,
        port: import.meta.env.VITE_KINOTIC_PORT ? parseInt(import.meta.env.VITE_KINOTIC_PORT) : 58503,
        useSSL: import.meta.env.VITE_KINOTIC_USE_SSL === 'true' || window.location.protocol === 'https:'
    }
}

await Kinotic.connect({ server: serverOverrides() })
```

The REST calls must reach the same server, so build their URLs from the same override —
returning a bare path when none is set keeps the dev proxy and a same-origin deployment
working — and send `credentials: 'include'` so the session cookie travels:

```typescript
function apiUrl(path: string): string {
    const { host, port, useSSL } = serverOverrides()
    return host ? `${useSSL ? 'https' : 'http'}://${host}:${port}${path}` : path
}

await fetch(apiUrl('/api/auth/me'), { credentials: 'include' })
```

This is the pattern the platform's own SPAs use (`serverOverrides()` / `apiUrl()` in
`@kinotic-ai/frontend-common`). A published Kinotic UI is served from its own site host,
not the gateway, so it is **cross-origin** — and the deployment hands the build the address
to use, so the override comes from `KINOTIC_UI_SERVER_URL` rather than a hand-set var. The
next section is how.

Session-cookie auth (Recipe 1) is origin-scoped, so a cross-origin SPA that cannot get
the cookie sent needs a bearer token instead (Recipe 3). Same-origin, or a dev proxy that
makes it look same-origin, is what keeps the cookie flow simple.

## The UI build contract

Every UI package under `packages/ui` that declares a `build` script is built during the
deployment with `bun run build` and published to its own site. The build is handed three
variables, and **the build must honor them** — the platform does not rewrite the output:

| Variable | Value | What the build must do |
|---|---|---|
| `KINOTIC_UI_BASE_PATH` | `/<commit sha>/` | Set the bundler's base/public path. Assets are published under the commit and cached for a year |
| `KINOTIC_UI_COMMIT` | the commit sha | Embed it, so a tab can tell when the site has moved on |
| `KINOTIC_UI_SERVER_URL` | the platform's public API address | The address the UI connects to Kinotic on from the browser |

**Ignoring `KINOTIC_UI_BASE_PATH` is the failure to look for first.** A default Vite build
emits `<script src="/assets/index-*.js">` while the publish uploads that file under
`/<commit>/assets/…`, so the page loads and every asset 404s. A Vite config that satisfies
all three:

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
    // Published assets live under /<commit>/; without this the built index asks for
    // /assets/... and 404s against the site.
    base: process.env.KINOTIC_UI_BASE_PATH ?? '/',
    define: {
        __KINOTIC_UI_COMMIT__: JSON.stringify(process.env.KINOTIC_UI_COMMIT ?? 'dev'),
        __KINOTIC_UI_SERVER_URL__: JSON.stringify(process.env.KINOTIC_UI_SERVER_URL ?? '')
    }
})
```

`define` is what carries the other two into client code — Vite only exposes `VITE_`-prefixed
variables through `import.meta.env`, so a bare `import.meta.env.KINOTIC_UI_COMMIT` is
`undefined`. Declare the injected constants for TypeScript
(`declare const __KINOTIC_UI_COMMIT__: string`).

The server address then feeds the same override the cross-origin case above needs, with the
defaults applying locally where the variable is unset:

```typescript
declare const __KINOTIC_UI_SERVER_URL__: string

const configured = __KINOTIC_UI_SERVER_URL__ ? new URL(__KINOTIC_UI_SERVER_URL__) : null
await Kinotic.connect(configured
    ? { server: { host: configured.hostname,
                  port: configured.port ? Number(configured.port) : null,
                  useSSL: configured.protocol === 'https:' } }
    : {})
```

A build that leaves no `dist/index.html` **fails the deployment**, naming the UI — so a UI
whose build writes elsewhere never publishes.

Each site also serves the commit it is running as `version.json` next to `index.html`, and a
publish keeps the previous commit's assets so open tabs keep working. That is what
`checkUiVersion` reads:

```typescript
import { checkUiVersion } from '@kinotic-ai/core'

const { stale } = await checkUiVersion(__KINOTIC_UI_COMMIT__)
if (stale) {
    // the site now serves a newer commit — offer a reload
}
```

It never rejects: a site whose `version.json` cannot be read leaves the tab not stale, so a
network blip never prompts a reload.

A package under `packages/ui` **without** a `build` script is treated as a library and is
never published — that is how shared component packages live there.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on. Only a **Node** process sending credentials must first call
`ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which swaps in `ws` because Node's
own WebSocket ignores a headers option. It is a no-op under Bun and unnecessary in the
browser — do not add it to a Bun entry point.

## Recipe 1 — Browser SPA (session cookie)

Same-origin only (see above). The browser logs in through the REST/OIDC flow first, which
sets a session cookie; the WebSocket upgrade is then authenticated by the cookie and the
host comes from the page's own origin — no configuration in JS at all:

```typescript
import { Kinotic } from '@kinotic-ai/core'

await Kinotic.connect()
```

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
