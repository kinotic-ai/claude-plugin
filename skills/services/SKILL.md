---
name: services
description: >
  How to write and call Kinotic services: publish TypeScript classes with @Publish,
  version and zone addressing (app.<org>.<app>, app-api, management-api), consuming
  services through hand-written service proxies with invoke and invokeStream, streaming
  results as RxJS Observables, and the organization scope a service host must connect
  with. Use when adding business logic, microservices, APIs, service-to-service calls,
  or real-time streams in a Kinotic application.
---

# Kinotic Services

Every remote call in a Kinotic app — service to service, frontend to backend — goes
over one STOMP-over-WebSocket connection through the RPC gateway, addressed by zone +
qualified name. There is no REST layer for application services.
Docs: <https://kinotic.ai/apps/services/overview>.

## Publishing a service

A plain class becomes remotely callable with `@Publish` — without it, the class is
never registered and proxies cannot reach it:

```typescript
import { Publish, Version } from '@kinotic-ai/core'

@Publish('com.example')
@Version('1.0.0')
export class GreetingService {
    async greet(name: string): Promise<string> {
        return `Hello ${name}`
    }
}
```

`@Publish(namespace?, name?, advertise?)` — `name` defaults to the class name;
`advertise: true` additionally lists the service in the Service Directory (publishing
alone makes it callable but unlisted).

**Hosting a service requires ORGANIZATION scope.** This is the rule that silently
breaks a service that otherwise looks correct. The gateway derives what a connection
may do from its participant scope, and only an organization participant may *subscribe*
— which is what hosting a `@Publish` class is:

| Connection scope | May send to | May subscribe in (host services) |
|---|---|---|
| ORGANIZATION (org id, no app id) | `management-api`, `app-api`, `app.<org>` and below | `app.<org>` and below |
| APPLICATION (org id + app id) | `app-api`, `app.<org>.<app>` | **nothing** |

So an application-scope identity — an app end-user, or a machine created on an
application's Machines page — can call services and use entity repositories, but can
never host one. The deployment provisions its runtime workload an organization-scope
machine identity for exactly this reason, which is why **pushing is how you run your
services** (see the `deploying` skill).

## The entry point

**Set `zonePrefix` before instantiating any `@Publish` class.** The zone is baked into
the registration when the class is instantiated, so a prefix set afterwards leaves the
service listening at the wrong address, silently. Instantiation order relative to
`connect()` is free: registrations queue until the connection is up and re-subscribe on
every reconnect — including across an explicit `disconnect()` / `connect()` cycle.
Outbound calls are the opposite — invoking a proxy or sending while disconnected throws
`You must call connect() before sending any data`; messages are never queued.

```typescript
import { Kinotic } from '@kinotic-ai/core'
import { appZone } from '@kinotic-ai/management-api'
import config from '../../../../.config/kinotic.config'

Kinotic.zonePrefix = appZone(config.organizationId, config.applicationId)
await Kinotic.connect()
// ... instantiate @Publish services here (before or after connect both work)
```

The application id comes from the project's own config, never from the connection: the
runtime connects at organization scope, so nothing in its credentials names the
application.

The scaffold ships this entry point at `packages/microservices/main/src/main.ts`, and
that path is what the deployment runs. Add services by instantiating them there or by
importing modules it pulls in — a second package under `packages/microservices/` is
built and type-checked but **not started** by the platform.

`Kinotic.connect(options?)` takes an optional plain `ConnectOptions` object literal —
every field is optional. Unset fields resolve in order: explicit value →
`KINOTIC_SERVER_HOST` / `KINOTIC_SERVER_PORT` / `KINOTIC_SERVER_USE_SSL` → the browser's
own location → `https://api.kinotic.ai`. Credentials resolve from
`KINOTIC_CLIENT_ID` + `KINOTIC_CLIENT_SECRET` (a machine identity's connection
credentials — no OAuth grant, no token to manage) or `KINOTIC_TOKEN` for bearer auth,
then the browser session. An explicit override nests the server fields:
`Kinotic.connect({ server: { host: 'localhost', port: 58503 } })`. When no credentials
resolve, connect fails with `No Kinotic credentials found; consulted: <resolver names>`.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on — nothing extra is needed. Only a **Node** process sending
credentials must first call `ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which
swaps in `ws` because Node's own WebSocket ignores a headers option.

A class-level `@Zone('billing')` nests a sub-zone (`app.<org>.<app>.billing`); a
project-wide default comes from `Kinotic.defaultZone`, which the entry point assigns
itself (conventionally from the `kinotic.zone` field in `package.json`:
`Kinotic.defaultZone = pkg.kinotic?.zone ?? null` — it is not auto-loaded).

Other service decorators (from `@kinotic-ai/core`):

- `@Version('1.0.0')` — semantic version pinning; callers can address `#<version>`.
- `@Scope` — on a getter/method supplying an instance id, to route calls to one
  specific instance of a service (e.g. per-node or per-device).
- `@ScopeOptional` — on a method of a scoped service that any instance may answer.
  Each instance then also listens on the shared unscoped address, where only the
  annotated methods may be invoked; an unscoped call to any other method is rejected.
- `@Context` — marks a method whose **final** parameter receives the request context
  (never sent by the caller). The context is `{}` unless the process registers a
  `ContextInterceptor`, which builds it from the raw event — the caller's participant
  arrives as the JSON `sender` header, not as a pre-populated field:

  ```typescript
  Kinotic.serviceRegistry.registerContextInterceptor({
      intercept(event, context) {
          return { ...context, sender: JSON.parse(event.headers.get('sender') ?? '{}') }
      }
  })
  ```

A published method may return a generic type (`Page<Order>`): the platform publishes a
monomorphized concrete schema for it (`OrderPage`). A doubly-generic return
(`Page<Map<string, unknown>>`) fails publication — publish a named DTO instead.

Details: <https://kinotic.ai/apps/services/publishing-services>.

## Consuming a service — the proxy pattern

Proxies are hand-written: define an interface, delegate each method through
`serviceProxy`:

```typescript
import type { IKinotic, IServiceProxy } from '@kinotic-ai/core'
import type { Observable } from 'rxjs'

export interface INotificationService {
    sendAlert(message: string): Promise<void>
    watchAlerts(severity: string): Observable<Alert>
}

export class NotificationService implements INotificationService {

    private readonly serviceProxy: IServiceProxy

    constructor(kinotic: IKinotic) {
        this.serviceProxy = kinotic.serviceProxy('app.acme-org.orders-app~com.example.NotificationService')
    }

    sendAlert(message: string): Promise<void> {
        return this.serviceProxy.invoke('sendAlert', [message])
    }

    watchAlerts(severity: string): Observable<Alert> {
        return this.serviceProxy.invokeStream('watchAlerts', [severity])
    }
}
```

The address string is load-bearing: `<zone>~<namespace>.<ClassName>` — the `~` (tilde)
delimits the zone from the qualified name; a dotted address collapses the zone into the
resource name and routes nowhere. The zones:

| Zone | Contains |
|---|---|
| `app.<orgId>.<appId>` | The application's own services |
| `app-api` | Platform data plane for applications (entity repositories live here) |
| `management-api` | Platform management services (organization scope) |
| `system-api` | Platform-internal services (system participants only) |

An address without a `~` carries no zone at all and is reachable only by system
participants — never what an application wants. The gateway validates the zone on every
send against the authenticated participant — a wrong zone routes nowhere and can never
cross into another application.
Addressing spec: <https://kinotic.ai/platform/reference/cri-format>.

## Streaming

A published method that returns an RxJS `Observable` becomes a streaming endpoint;
consumers call it with `invokeStream(...)` and receive an `Observable`. Values flow
until the consumer unsubscribes or the producer completes. Use for live feeds,
progress, and tailing. Docs: <https://kinotic.ai/apps/services/streaming>.

## Rules of thumb

- One connected `Kinotic` singleton per process; set `Kinotic.zonePrefix` before any
  `@Publish` class is instantiated — instantiation before or after `connect()` both
  work.
- Service classes hold business logic; entities stay dumb data (persistence is the
  entities-and-persistence skill). A service reaches data through the generated
  repositories, which work in any connected process.
- Method arguments and returns must be serializable data; use `Promise` for
  request-response and `Observable` for streams.
