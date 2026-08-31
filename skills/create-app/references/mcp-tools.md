# Kinotic OS MCP tool contracts

The kinotic-os server is a stateless streamable-HTTP MCP endpoint at `POST <server url>`
secured by OAuth 2.1. Behavior common to every tool:

- **Tool names are opaque hashes — never hardcode or invent one.** A tool's `name` is
  the base-36 XXH3-128 of `<zone>~<package>.<ServiceName>/<functionName>` (≤ 25 chars of
  `[0-9a-z]`), e.g.
  `management-api~org.kinotic.management.api.services.ProjectService/findById` hashes
  to something like `5nwldv2gqljeygvmd1ywjl2z7`. Resolve tools from `tools/list` by
  their human-readable `title` (the titles used below), then call the listed `name`.
  A stale or invented name fails as a JSON-RPC protocol error (`-32602`,
  `Unknown tool: <name>`) — not as a tool result.
- **Results are raw JSON in a single text content block.** The tool result contains one
  `text` item whose text is the JSON serialization of the service's return value — parse
  it. An empty list result is the literal text `[]`, and a `null` result is the literal
  text `null`.
- **Unknown argument keys fail the call** with `isError: true`
  (`Received argument '<key>' which matches no parameter of <function>`). Send exactly
  the argument names documented below.
- **Service errors arrive as `isError: true`** with the exception message as text
  content. Match on the message substrings documented below.
- **Every tool carries an `annotations` block** with all four hints written explicitly
  (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`). Treat
  `destructiveHint: true` (`… Save`, `… Delete By Id`, and their `Sync` twins) as
  never-call-speculatively. `openWorldHint` is `false` on everything except
  `Project Service Retry Repo Initialization`, which reaches GitHub.
- Authenticated users act as their Kinotic **organization**, so the tools below operate
  on that organization's applications and projects.

## The full tool surface

Both services carry a type-level `@McpTool`, so **every** function on them is a tool,
including the ones inherited from the CRUD hierarchy. Beyond the tools documented below
that is:

| Inherited from | Tools |
|---|---|
| `CrudService` | `… Save`, `… Save Sync`, `… Find By Id`, `… Count`, `… Delete By Id`, `… Delete By Id Sync`, `… Find All`, `… Search`, `… Sync Index` |
| `IdentifiableCrudService` | `… Create`, `… Create Sync` |
| `ApplicationScopedCrudService` (ProjectService only) | `Project Service Count For Application`, `Project Service Find All For Application` |

`Project Service Find All For Application` with `{"applicationId": ..., "pageable": ...}`
is the way to list an application's projects. Stick to the documented tools for the
onboarding workflow, and never call a destructive tool unless the user explicitly asks
for that exact operation:

- `Application Service Delete By Id` is guarded server-side
  (`Cannot delete an application with projects in it.`).
- `Project Service Delete By Id` has no such guard, and deleting a project **cascades**:
  its deployment record and both of its machine identities go with it, cutting off any
  workload still running. The checkout and runtime VM on the node are not reachable from
  the management plane; deleting the runtime machine is what strands them.

## ApplicationService

Qualified name: `management-api~org.kinotic.management.api.services.ApplicationService`

### `Application Service Create Application If Not Exist`

Creates an application, or returns the existing one whose id matches the slugified name.
Idempotent.

Arguments:

```json
{ "name": "Inventory App", "description": "Tracks warehouse inventory" }
```

Any human-readable `name` works — the server slugifies it into the id (lowercase
letters, digits, interior dashes). It fails only when the name is blank, slugifies to
nothing (all punctuation), or slugifies to the platform-reserved id `system-api`.

Result (Application):

```json
{
  "id": "inventory-app",
  "organizationId": "acme",
  "name": "Inventory App",
  "description": "Tracks warehouse inventory",
  "oidcConfigurationIds": null,
  "tenantPerUser": false,
  "updated": 1753747200000
}
```

`id` is the server-minted slug of `name`. `organizationId` is derived from the
authenticated user — pass both into project creation.

### `Application Service Get Oidc Configurations`

Read-only. Returns the **enabled** OIDC configurations registered on an application
(used by the frontend skill when wiring login). Returns `[]` when the application has
none. Errors with `Application not found: <applicationId>` when the id does not name an
application in the caller's organization.

Arguments:

```json
{ "applicationId": "inventory-app" }
```

## ProjectService

Qualified name: `management-api~org.kinotic.management.api.services.ProjectService`

### `Project Service Create Project If Not Exist`

Creates a project and provisions its GitHub repository from the Kinotic template through
the organization's GitHub App installation. Returns the existing project unchanged if
one with the same id exists. Idempotent.

The single argument key is exactly `project`:

```json
{
  "project": {
    "applicationId": "inventory-app",
    "organizationId": "acme",
    "name": "Inventory App",
    "description": "Tracks warehouse inventory",
    "repoPrivate": true,
    "sourceOfTruth": "TYPESCRIPT"
  }
}
```

- `applicationId` and `name` are required; the project id is derived as
  `<applicationId>-<slugified name>` when not set (e.g. `inventory-app-inventory-app`).
- `organizationId` is **required**, not just constrained: omitting it fails with
  `Organization id must be set on Project`, and a mismatch fails with
  `Cannot save Project with organizationId '<x>' while authenticated as organization '<y>'`.
- The GitHub repository is named after the slugified project **name** (truncated to
  100 chars), created under the account holding the GitHub App installation — record
  `repoFullName` from the result rather than deriving it.
- `sourceOfTruth` accepts only `"TYPESCRIPT"` today (it is not validated server-side).
- `repoPrivate` controls the GitHub repository visibility at creation.

Result (Project) — fields beyond the input:

```json
{
  "id": "inventory-app-inventory-app",
  "repoFullName": "acme-gh-org/inventory-app",
  "repoId": 123456789,
  "repoDefaultBranch": "main",
  "repoConnectionStatus": "CONNECTED",
  "updated": 1753747200000
}
```

`repoConnectionStatus` values:

| Status | Meaning | Action |
|---|---|---|
| `CONNECTED` | Repo provisioned and baseline committed | Proceed |
| `INITIALIZATION_FAILED` | Repo exists but the baseline commit failed | Call `Project Service Retry Repo Initialization` |
| `DISCONNECTED` | GitHub revoked the platform's access to the repo | User must re-link in the portal |

Errors:

- `"GitHub is not linked for this organization. Link GitHub before creating a project."`
  — the organization has no GitHub App installation. The user links GitHub in the
  Kinotic OS portal (Organization settings → Integrations → GitHub → Link GitHub), then
  the same call is re-run.

### `Project Service Retry Repo Initialization`

Re-runs repository initialization for a project left in `INITIALIZATION_FAILED`.
Succeeds with the project marked `CONNECTED` once the baseline is committed; the write
is read-your-write consistent, so the returned project reflects the outcome.

Arguments:

```json
{ "projectId": "inventory-app-inventory-app" }
```

Errors:

- `"projectId must not be blank"`
- `"Project for id <projectId> does not exist"`
- `"Project <projectId> is not awaiting initialization retry (status <status>)"` — the
  project is not in `INITIALIZATION_FAILED`; never call this tool speculatively.
- `"Project repoId must be set to reinitialize"` (likewise `repoFullName` /
  `repoDefaultBranch`) — provisioning never got far enough for a retry; re-run project
  creation instead.
- `"GitHub is not linked for this organization. …"` — the installation disappeared
  between create and retry; back to GitHub linking.

### `Project Service Find Deployment`

Read-only. Returns the project's deployment record — where its code is running and which
commit is live — or `null` when the project has never been deployed. This is how a push
is verified; see the `deploying` skill for the whole workflow.

Arguments:

```json
{ "projectId": "inventory-app-inventory-app" }
```

Result (ProjectDeployment), `id` always equal to the project id:

```json
{
  "id": "inventory-app-inventory-app",
  "organizationId": "acme",
  "applicationId": "inventory-app",
  "nodeId": "node-1",
  "hostDir": "<node workload data dir>/projects/inventory-app-inventory-app",
  "runtimeWorkloadId": "wl-…",
  "syncMachineIdentityId": "…",
  "runtimeMachineIdentityId": "…",
  "commitSha": "9f2c1ab…",
  "lastJobRunId": "…",
  "status": { "type": "RUNNING", "message": null },
  "created": 1753747200000,
  "updated": 1753747200000
}
```

`status.type` is `DEPLOYING`, `RUNNING`, or `FAILED`; `status.message` carries the last
job's failure reason when there is one. Errors with `"projectId must not be blank"`.

### `Project Service Find by GitHub Repo`

Read-only. Looks up projects in the caller's organization whose backing GitHub repo has
the given `owner/repo` full name (function `findByRepoFullName`). Returns `[]` when none
match.

Arguments:

```json
{ "repoFullName": "acme-gh-org/inventory-app" }
```
