---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: Kinotic.connect,
  browser session-cookie authentication, BasicCredentialsResolver and
  BearerCredentialsResolver for Node clients, machine identities via environment
  credentials, OIDC login, and invoking published services from the UI. Use when
  building a Kinotic app frontend or SPA, wiring up login or authentication, or
  connecting any client to a Kinotic server.
---

# Frontends and Clients

Every client — browser SPA, Node service, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth (`/api/auth/*`) and MCP (`/mcp`). Frontends
live in `packages/ui` of the project scaffold and may use any framework that builds to
static assets. Docs: <https://kinotic.ai/apps/security/authentication>.

Authentication happens at the **WebSocket upgrade**, and every authentication carries a
scope. Application users always authenticate at APPLICATION scope — both
`organizationId` and `applicationId` are supplied (directly or via JWT claims). Scope
isolation means a user of one application can never reach another application, even
with the same identity provider.

`Kinotic.connect(options?)` takes an optional plain `ConnectOptions` literal; every
field is optional. Unset fields resolve from the environment: server from
`KINOTIC_SERVER_HOST` / `KINOTIC_SERVER_PORT` / `KINOTIC_SERVER_USE_SSL` (in a browser,
from the page's own origin), credentials from a resolver chain. Node clients that send
credential headers must call `ensureNodeWebSocket()` from `@kinotic-ai/core/node`
before connecting.

## Recipe 1 — Browser SPA (session cookie)

The browser logs in through the REST/OIDC flow first, which sets a session cookie; the
WebSocket upgrade is then authenticated by the cookie and the host comes from the
page's own origin — no configuration in JS at all:

```typescript
import { Kinotic } from '@kinotic-ai/core'

await Kinotic.connect()
```

## Recipe 2 — Node client or machine, zero ceremony (environment credentials)

The default credential resolver chain reads the environment, so a microservice or
script normally passes nothing:

```bash
KINOTIC_SERVER_HOST=localhost KINOTIC_SERVER_PORT=58503 \
KINOTIC_CLIENT_ID=<machine identity id> KINOTIC_CLIENT_SECRET=<secret> \
bun run src/main.ts
```

```typescript
import { Kinotic } from '@kinotic-ai/core'
import { ensureNodeWebSocket } from '@kinotic-ai/core/node'

ensureNodeWebSocket()
await Kinotic.connect()
```

`KINOTIC_CLIENT_ID` + `KINOTIC_CLIENT_SECRET` are a machine identity's connection
credentials — sent on the WebSocket upgrade, no OAuth grant or token lifecycle
(optionally `KINOTIC_ORGANIZATION_ID` / `KINOTIC_APPLICATION_ID` pin the scope). A
`KINOTIC_TOKEN` env var resolves to bearer auth instead. A `clientId` containing `@`
is a user email; anything else resolves only to a machine identity. When nothing
resolves, connect fails with `No Kinotic credentials found; consulted: <resolvers>`.

## Recipe 3 — explicit credentials in code

Email/password (user identity):

```typescript
import { BasicCredentialsResolver, Kinotic } from '@kinotic-ai/core'
import { ensureNodeWebSocket } from '@kinotic-ai/core/node'

ensureNodeWebSocket()
await Kinotic.connect({
    credentials: new BasicCredentialsResolver('user@example.com', 'password', 'my-organization', 'my-application')
})
```

The arguments are `(clientId, clientSecret, organizationId?, applicationId?)` — scope
is structural: neither id → SYSTEM, organization only → ORGANIZATION, both →
APPLICATION.

Bearer token (OIDC):

```typescript
import { BearerCredentialsResolver, Kinotic } from '@kinotic-ai/core'

await Kinotic.connect({ credentials: new BearerCredentialsResolver(async () => await getToken()) })
```

The supplier runs before every (re)connect, so short-lived tokens refresh
automatically. A Kinotic-minted JWT already carries the organization/application
claims — no extra scope headers.

## OIDC login for your application's users

1. An admin enables OIDC configurations (Google, Microsoft, a corporate IdP) on the
   application. To see what is enabled, call the MCP tool titled
   `Application Service Get Oidc Configurations` with `{"applicationId": "..."}`
   (Kinotic MCP tool names are opaque hashes — always resolve by title from the tool
   listing).
2. The frontend runs the OAuth flow; the REST login establishes the session cookie;
   then connect as in Recipe 1.
3. Application-scope users are **pre-created** by an administrator — an OIDC identity
   alone does not grant access. OIDC users are created with an email only; the subject
   links automatically on first login.

## Calling services from the frontend

Once connected, the frontend calls published services exactly like any client — the
hand-written proxy pattern from the services skill (note the `~` between zone and
qualified name):

```typescript
const proxy = Kinotic.serviceProxy('app.my-organization.my-application~com.example.OrderService')
const orders = await proxy.invoke('findOpenOrders', [customerId])
proxy.invokeStream('watchOrder', [orderId]).subscribe(update => render(update))
```

Generated entity repositories (entities-and-persistence skill) also work in the
browser after `Kinotic.use(PersistencePlugin)` — data access needs no bespoke backend
endpoints.

## Further reading

- Authentication: <https://kinotic.ai/apps/security/authentication>
- Service proxies: <https://kinotic.ai/apps/services/service-proxies>
- Platform auth model: <https://kinotic.ai/platform/system-security>
