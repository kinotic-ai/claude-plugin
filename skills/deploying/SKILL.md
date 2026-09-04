---
name: deploying
description: >
  How a Kinotic project runs: pushing to the default branch builds and deploys it, the
  deployment job's three steps, checking deployment status and the deployed commit with
  the Project Service Find Deployment MCP tool, diagnosing a FAILED deployment, the
  machine identities the deployment provisions, and what can and cannot be run locally.
  Use when deploying, running, restarting, or releasing a Kinotic app, checking whether
  a service is live, or working out why a push did not take effect.
---

# Running and Deploying a Kinotic Project

Pushing to the project repository's **default branch** is the deployment. There is no CI
to configure, no pipeline file, and nothing to run from a developer machine.
Docs: <https://kinotic.ai/apps/deployment/push-to-deploy>.

Only default-branch pushes deploy. Pushes to other branches, branch deletions, and tag
operations are ignored.

## What the platform actually runs

| Artifact | Status |
|---|---|
| Entities and their generated repositories | Synchronized and published on every push |
| Every directory under `packages/microservices` with a `package.json` | Gets its own long-lived runtime workload |
| Every directory under `packages/ui` whose `package.json` declares a `build` script | Built with `bun run build` and published to its own site |
| A `packages/ui` package with no `build` script | Treated as a library — never published |
| Batch/scheduled jobs | Not implemented |

**Artifacts are discovered by where they sit and what their `package.json` says**, so
adding one is adding a package — there is no registry to update:

- A microservice is a directory directly under `packages/microservices` holding a
  `package.json`. Its entry is that file's `main`, or `src/main.ts` when it declares none.
  Each gets its own workload, so a second microservice really does run.
- A UI is a directory directly under `packages/ui` whose `package.json` declares a `build`
  script. It must honor the UI build contract (`frontend` skill) or it publishes assets
  nothing can load.
- An artifact's identity is the **unscoped part of its `package.json` name** (`@acme/orders`
  is `orders`), which must be lowercase letters, digits, and interior dashes, and unique
  among artifacts of its kind. The directory name never matters. A missing or invalid name,
  or two artifacts of one kind sharing a name, fails the deployment naming the package.

## The deployment job

Each qualifying push runs one tracked job with three steps:

1. **Resolve deployment target** — the first deployment picks a node with capacity and
   creates the project's checkout directory on it; later deployments reuse the same node
   and directory.
2. **Sync project source** — a short-lived sandboxed VM fetches the pushed commit into
   the checkout (incremental, so installs stay warm), runs `bun install`, then
   `kinotic sync`, which regenerates from the entity sources, pushes the entity
   definitions and named queries, publishes every entity it introduces (which creates
   the backing storage), and applies pending migrations from `./migrations`. **This step
   is the build gate**: a commit that does not compile never reaches the running
   services. Only after it fully succeeds does it write the reload sentinel.
   The UIs are built in this step too, each with the three build variables, and a UI build
   that writes no `dist/index.html` fails the deployment naming it.
3. **Ensure runtime workloads** — every microservice of the commit ends with a workload
   serving it, one per microservice: a new microservice gets a VM on its first appearance,
   and a running one is kept, its supervisor restarting the process onto the new commit
   with escalating backoff if it crashes immediately.

Consequences worth stating to a user:

- A failed build leaves the **previously deployed commit running**. Nothing is torn down.
- Deployments serialize per project, **latest-wins**: pushes arriving during a deployment
  collapse to the newest commit; intermediate commits are skipped, not queued.
- Redelivering the same push is harmless — syncing a commit twice converges.
- The runtime restarts your services as whole processes. There is no hot reload of a
  single class.
- An artifact that disappears from a commit is marked **orphaned**, not deleted: its
  workload keeps running and its site keeps serving. One that comes back is adopted rather
  than re-provisioned. Removing it for real is a console action.
- **The build gate covers compilation, not the server's verdict on a definition.** A
  commit that does not compile fails the job; an entity or named-query the server then
  rejects is logged by the sync step and does not fail it. So a `RUNNING` deployment
  proves the code built, not that every definition landed — check the sync step's output
  on the job page when an entity behaves as if it were never synchronized.

## Checking a deployment

Call the MCP tool titled `Project Service Find Deployment` with
`{"projectId": "<project id>"}` (Kinotic OS tool names are opaque hashes — always resolve
by title from the tool listing). It returns `null` when the project has never deployed.

```json
{
  "id": "inventory-app-main",
  "nodeId": "node-1",
  "commitSha": "9f2c1ab…",
  "runtimeWorkloadId": "wl-…",
  "syncMachineIdentityId": "…",
  "runtimeMachineIdentityId": "…",
  "lastJobRunId": "…",
  "status": { "type": "RUNNING", "message": null }
}
```

| `status.type` | Meaning | What to tell the user |
|---|---|---|
| `DEPLOYING` | A job is running for the latest qualifying push | Wait and re-check; the job page shows live steps |
| `RUNNING` | `commitSha` built and the runtime is serving it | Compare `commitSha` to the pushed commit |
| `FAILED` | The last deployment failed | `status.message` carries the reason; the previous commit is still live |

Poll rather than assume: after pushing, `findDeployment` moves `DEPLOYING` →
`RUNNING`/`FAILED`. A `commitSha` that still names the previous commit while the status
is `RUNNING` means the new push has not been picked up yet — check that it landed on the
default branch.

## When a deployment fails

`status.message` is the failure reason recorded from the job. The most common cause is
the build gate: the pushed commit does not compile, or `kinotic sync` rejected an entity
change. Reproduce it locally first — `bun install && bun run generate && bun run type-check`
runs the same generation the sync step does — then fix and push again.

Deployment logs are **not** reachable through MCP: `LogService` streams workload logs but
is not exposed as a tool. Send the user to the portal for anything the status message
does not answer:

- **Project → Deployment page** (`/application/<applicationId>/project/<projectId>/deployment`)
  — status, deployed commit, the last job's steps live, and the project's machines.
- **Jobs page** (`/jobs`, then the run) — the same job among all organization jobs.
- The failed sync VM is kept (not discarded) so its logs stay inspectable from the
  workload logs view; it is cleaned up on the next successful deployment.

Never try to "fix" a deployment by deleting and recreating the project — that removes
the project's machine identities and deployment record along with it.

## Machine identities

The deployment provisions the project two machine identities, one per workload, listed
on the Deployment page. Both are **organization scope**, because publishing services into
the application's zone and synchronizing entity definitions are things the organization
does on its own behalf — an application-scope identity cannot do either.

Kinotic stores only a hash of a machine's secret, so a secret is disclosed exactly once:

- The **sync** VM gets a freshly issued secret on every deployment; the previous one stops
  working at that moment.
- The **runtime** VM's secret is issued once with the VM and lives for that VM's life,
  since later deployments restart the process inside the running VM.

Do not rotate a project's runtime machine secret to borrow it for local work — the
running deployment is using it, and rotating cuts it off at its next reconnect. Deleting
a project machine is recoverable: the next deployment provisions a replacement.

## Running locally

What is possible locally follows directly from connection scope (see the `services`
skill for the full table):

- **Calling services and using entity repositories** — works. Create a machine identity
  on the portal's **Application → Machines** page (application scope), and run with
  `KINOTIC_CLIENT_ID` / `KINOTIC_CLIENT_SECRET` set, or connect as an application user
  with `BasicCredentialsResolver(email, password, organizationId, applicationId)`. This
  is how a local frontend or a one-off script talks to the deployed application.
- **Hosting a `@Publish` service** — needs organization scope, which the Machines page
  does not mint (it provisions application-scope machines). Push to run your services.

So the working loop is: edit → `bun run generate` (when entities changed) → export from
`packages/domain/index.ts` → `bun run type-check` → commit → push → `findDeployment`
until `RUNNING`. Verify behavior by calling the deployed service through a proxy from a
local script or the frontend, not by trying to host it locally.

A fully local stack (`deployment/docker-compose/` in the platform repository) is a
platform-operator setup, not part of an application project's workflow.
