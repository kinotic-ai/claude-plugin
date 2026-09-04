# Kinotic Claude Code Plugin

Build [Kinotic](https://kinotic.ai) applications with Claude. The plugin connects
Claude to Kinotic OS through the platform's MCP server and teaches Claude the Kinotic
programming model, so a user can go from nothing to a working application: sign up,
create an Application and Project (Kinotic OS provisions the GitHub repository), and
develop entities, services, and frontends.

## Installation

```
/plugin marketplace add kinotic-ai/claude-plugin
/plugin install kinotic@kinotic
```

For local development, add the marketplace from a checkout path instead:

```
/plugin marketplace add /path/to/claude-plugin
/plugin install kinotic@kinotic
```

## Configuration

The plugin connects to Kinotic OS Cloud (`https://api.kinotic.ai/mcp`). The URL is a
literal on purpose: the Claude desktop app interpolates neither `${user_config.*}`
plugin settings nor `${VAR:-default}` env syntax — both reach its connector dialog as
the raw placeholder and fail validation (and `${user_config.*}` also has open Claude
Code bugs: anthropics/claude-code#51573, #51538).

To work against a different Kinotic OS (staging, self-hosted, local), add that
endpoint as its own MCP server alongside the plugin:

- **Claude Code** — `claude mcp add --transport http kinotic-os-test <your endpoint URL>`
- **Claude desktop app** — Settings → Connectors → Add custom connector (`https` only)

The skills resolve the Kinotic OS tools by title from the tool listing, so they work
the same through a manually added server. Two server-side settings gate a non-cloud
server: it must list the Claude client-metadata URL in
`kinotic.domain.oauth.allowedClientIds` (there is no dynamic client registration),
and when its OAuth surface is reached at a different host than the browser uses
(localhost behind a tunnel), it must set `kinotic.domain.oauth.issuerBaseUrl` —
missing either makes `/mcp` OAuth fail as if the server never appeared.

After installing, run `/mcp`, select `kinotic-os`, and authenticate. The browser opens
the Kinotic OS OAuth flow — sign up from the login page if you have no account, then
approve the consent screen. Claude Code stores and refreshes the token automatically.

## What's inside

| Component | Purpose |
|---|---|
| `kinotic-os` MCP server | Remote streamable-HTTP MCP endpoint exposing Kinotic OS platform tools (application/project creation, project lookup, deployment status) secured by OAuth 2.1 |
| `create-app` skill | End-to-end onboarding: sign up or sign in to Kinotic OS, create the Application and its first Project, handle GitHub linking and repo provisioning, clone, verify the scaffold, first entity and first push |
| `entities-and-persistence` skill | Entity classes and decorators, `bun run generate`, repository API, named queries, migrations, multi-tenancy |
| `services` skill | Publishing services with `@Publish`, zones and addressing, the organization scope a service host needs, service proxies, streaming |
| `frontend` skill | How the UI resolves the server URL, the UI build contract every published UI must honor, connecting browser and Node clients, authentication recipes, application users and OIDC, calling services from the UI |
| `deploying` skill | Push-to-deploy: what the platform actually runs, the deployment job, checking status with `Project Service Find Deployment`, diagnosing failures, machine identities, what can be run locally |
| `/kinotic:new-app` command | Deterministic entry point that runs the create-app workflow |

## How a Kinotic app fits together

A push to the project repository's default branch is the whole pipeline. The deployment
runs a sandboxed build VM that checks the commit out, installs, and runs `kinotic sync`
(which regenerates from the entity sources, pushes and publishes the entity definitions and
named queries, and applies migrations), builds and publishes every UI, then leaves every
microservice of the commit with a running VM. A commit that does not compile fails the build
and never reaches the running services.

Three consequences shape every skill in this plugin:

- **Artifacts are discovered from the tree.** Every package under `packages/microservices`
  gets its own runtime workload, and every one under `packages/ui` with a `build` script is
  published as a site — its `dist` is uploaded under the site root as it is, and a build
  leaving no `dist/index.html` fails the deployment.
- **Hosting a `@Publish` service requires organization scope**, which the portal's
  Machines page does not mint. Running your services means pushing.
- **A published entity definition is additive-only.** The push publishes new entities and
  creates their storage, but from then on only new fields apply in place — a rename or
  type change needs a migration, or an un-publish that drops the index and its data.


Kinotic OS mints MCP tool names as opaque base-36 hashes (≤ 25 chars of `[0-9a-z]`),
so permission names look like `mcp__plugin_kinotic_kinotic-os__5nwldv2gqljeygvmd1ywjl2z7`
— grant with a wildcard (`mcp__plugin_kinotic_kinotic-os__*`) rather than per-tool
literals, and select tools by their human-readable `title` in the tool listing.

## End-to-end test checklist (maintainers)

Requires a running Kinotic OS (`kinotic-server`), a test GitHub org with the Kinotic
GitHub App installable, and a browser.

1. Add the local server alongside the plugin:
   `claude mcp add --transport http kinotic-os-test http://localhost:58503/mcp`
   (a localhost server needs `kinotic.domain.oauth.issuerBaseUrl` + an
   `allowedClientIds` entry for the OAuth flow to complete).
2. `/mcp` → `kinotic-os` → complete the OAuth flow, including one pass as a brand-new
   signup.
3. Ask Claude to create an app (or run `/kinotic:new-app TestApp`):
   - Application and Project created; correct ids recorded.
   - With GitHub unlinked: the "GitHub is not linked" error is surfaced with dashboard
     instructions, and the same call succeeds after linking.
   - Repo provisioned from the template; `repoConnectionStatus` is `CONNECTED`.
   - Force a baseline failure (e.g. temporarily revoke contents permission) and verify
     the `INITIALIZATION_FAILED` → `retryRepoInitialization` path.
4. Clone the repo and compare against
   `skills/create-app/references/project-scaffold.md` — fix that reference, not the
   clone, if the template has drifted.
5. `bun install`, `bun run generate` (with a first entity), export it from
   `packages/domain/index.ts`, `bun run type-check`, commit, push. Then poll
   `Project Service Find Deployment` until `status.type` is `RUNNING` on the pushed
   `commitSha`, confirm the entity definitions arrived published, and verify a `save()`
   round-trips without any manual step.
6. Push a commit that does not compile and verify the deployment reports `FAILED` with a
   reason while the previous commit keeps running.
7. Spot-check skill triggering: "define a kinotic entity for orders" should load
   entities-and-persistence, "publish a service" the services skill, "why isn't my
   service running" the deploying skill, and none should fire on unrelated prompts.
