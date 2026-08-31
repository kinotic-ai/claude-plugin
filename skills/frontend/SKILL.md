---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: Kinotic.connect,
  browser session-cookie authentication, BasicCredentialsResolver and
  BearerCredentialsResolver, machine identities via environment credentials, OIDC login,
  creating application users, and invoking published services and entity repositories
  from the UI. Use when building a Kinotic app frontend or SPA, wiring up login or
  authentication, or connecting any client to a Kinotic server.
---

# Frontends and Clients

Every client — browser SPA, Node client, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth (`/api/auth/*`) and MCP (`/mcp`). Frontends
live in `packages/ui` of the project scaffold and may use any framework that builds to
static assets. Docs: <https://kinotic.ai/apps/security/authentication>.

**The platform does not host frontends today.** A `packages/ui` package is built and
type-checked by the deployment but never served — the user runs it with their framework's
dev server or hosts the static build wherever they like. Say so plainly rather than
implying a deploy will publish it.

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
the environment variables, then the browser session. A browser therefore needs no
configuration at all, and a fully unconfigured client reaches Kinotic OS Cloud.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on. Only a **Node** process sending credentials must first call
`ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which swaps in `ws` because Node's
own WebSocket ignores a headers option. It is a no-op under Bun and unnecessary in the
browser — do not add it to a Bun entry point.

## Recipe 1 — Browser SPA (session cookie)

The browser logs in through the REST/OIDC flow first, which sets a session cookie; the
WebSocket upgrade is then authenticated by the cookie and the host comes from the
page's own origin — no configuration in JS at all:

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
