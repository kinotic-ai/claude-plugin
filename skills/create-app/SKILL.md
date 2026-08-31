---
name: create-app
description: >
  Set up a new Kinotic application end to end: authenticate with Kinotic OS through the
  kinotic-os MCP server (signing up if needed), create the Application and Project with
  the Kinotic MCP tools, wait for the GitHub repository to be provisioned from the
  template, connect to GitHub, clone the repository, and verify the scaffold. Use when
  the user wants to create, start, bootstrap, or set up a new Kinotic app or project,
  sign up for Kinotic OS, or connect Claude to Kinotic OS.
---

# Creating a Kinotic Application

Follow these steps in order. Exact MCP tool contracts (argument and result JSON shapes,
error strings) are in `references/mcp-tools.md`. The expected repository layout is in
`references/project-scaffold.md`.

An application holds one or more projects; the workflow below creates the first one,
scaffolded as a Bun-workspace mono repo. The project's GitHub repository is created by
Kinotic OS — never create the repository yourself.

**MCP tool names are opaque hashes.** Resolve every Kinotic OS tool from the tool
listing by its human-readable `title` (e.g. `Application Service Create Application If
Not Exist`) — never call a tool by a guessed or remembered name, and never parse the
hash. `references/mcp-tools.md` documents each tool by title.

## Step 0 — Sign up or sign in to Kinotic OS

Check whether the kinotic-os MCP tools are available — look in the tool listing for
titles starting with `Application Service` / `Project Service` (e.g.
`Application Service Create Application If Not Exist`). If they are present, the user
is already connected — skip to Step 1.

If the tools are missing, the user has not authenticated the `kinotic-os` MCP server
yet. Ask whether they already have a Kinotic OS account, then walk them through the
matching path — the whole flow happens in their browser, so narrate what they will see:

1. Tell the user to run `/mcp`, select `kinotic-os`, and authenticate. A browser opens
   on the Kinotic OS **portal** login page — the one with a "New to Kinotic?
   **Create an organization**" link. The plugin targets Kinotic OS Cloud
   (`https://api.kinotic.ai/mcp`); for a different Kinotic OS see "Other servers"
   below.
2. **Existing account** — sign in, then on the consent screen ("Authorize …", with a
   "Verified as <host>" line) click **Approve**. **Deny** is a real decision that
   returns a denial to Claude Code — the tools will not appear. If the page shows
   "Authorization request unavailable", the link is stale: re-run `/mcp` to mint a
   fresh one.
3. **No account yet** — click **Create an organization** on the login page:
   - Sign up with GitHub — that is the only sign-up path (Kinotic projects are backed
     by GitHub repositories, so the organization starts from the user's GitHub
     identity). Users invited to an existing organization accept the emailed
     invitation link instead of creating a new organization.
   - Pick an organization name (and optional description) on the registration page and
     click **Create organization**. This creates the Kinotic organization that owns
     every application and project.
   - The page then shows **"Your organization is ready"** with a **Continue to
     GitHub** button — this installs the Kinotic GitHub App, which authorizes
     repository access. The user chooses the GitHub account or org to install into;
     that is where project repositories will be created. GitHub may interrupt with a
     sudo email code or a 2FA prompt — normal, not a Kinotic error.
   - Signup ends on the Applications page. It does **not** return to the OAuth consent
     screen — the authorization request is not resumed. Tell the user to go back to
     Claude Code and run `/mcp` → `kinotic-os` → authenticate again; now signed in,
     they land directly on the consent screen — click **Approve**.
4. Back in Claude Code, confirm the kinotic-os tools are now available before moving on.

If tools still do not appear right after a server restart, wait and retry: the service
directory publishes tools shortly after startup, not instantly.

The user can review or revoke this connection later under **Account → Connected apps**
in the portal.

**Other servers.** A staging, self-hosted, or local Kinotic OS is added as its own MCP
server: `claude mcp add --transport http kinotic-os-test <url>` — the tools work the
same from either. Two server-side settings gate this: the server must list Claude
Code's client-metadata URL in `kinotic.domain.oauth.allowedClientIds` (there is no
dynamic client registration), and a server whose OAuth surface is reached at a
different host than the browser uses (e.g. localhost behind a tunnel) must set
`kinotic.domain.oauth.issuerBaseUrl`. Missing either looks like "the server never
appeared" after OAuth.

## Step 1 — Create the Application

1. Ask the user for an application name and one-line description if not already known.
   Any human-readable name works — the server slugifies it into the application id. It
   is rejected only when blank, all punctuation, or slugifying to the platform-reserved
   id `system-api`.
2. Call the tool titled `Application Service Create Application If Not Exist` with
   `{"name": ..., "description": ...}`. The call is idempotent — if the application
   already exists it is returned unchanged.
3. From the result, record `id` (the server-minted slug of the name, e.g.
   `Inventory App` → `inventory-app`) and `organizationId`. Both are needed in Step 2.

## Step 2 — Create the Project (provisions the GitHub repository)

Call the tool titled `Project Service Create Project If Not Exist`. The single argument
key is exactly `project`:

```json
{
  "project": {
    "applicationId": "<application id from Step 1>",
    "organizationId": "<organizationId from Step 1>",
    "name": "<project name — reuse the application name unless the user wants otherwise>",
    "description": "<project description>",
    "repoPrivate": true,
    "sourceOfTruth": "TYPESCRIPT"
  }
}
```

`organizationId` is required — the server rejects a project without it. Creation
provisions a GitHub repository from the Kinotic template through the organization's
GitHub App installation and commits a rendered baseline. The call is idempotent:
re-running it returns the existing project unchanged.

Handle the outcomes:

- **Error containing "GitHub is not linked for this organization"** — the organization
  has no GitHub App installation. Fresh signups normally completed this during Step 0
  ("Continue to GitHub"), so this error usually means that install was skipped or
  abandoned. Direct the user to the Kinotic OS portal → sidebar **Organization →
  Organization settings** → **Integrations** → the **GitHub** card → **Link GitHub**.
  They choose the GitHub account or org to install the Kinotic GitHub App into — that
  is where project repositories are created; on success the card reads "Linked as
  <account>". An organization holds one installation at a time (moving to a different
  account requires **Unlink** first). Then re-run the exact same tool call.
- **GitHub App installs can fail visibly.** The install returns to a callback page; on
  failure it shows "Couldn't finish linking GitHub" plus a reason. Common ones:
  *state is missing, expired, or already used* (the install state is single-use and
  expires in 10 minutes — start the link again from Organization settings; never
  refresh or re-open the callback URL); *the GitHub account that authorized this
  install does not have access to the requested installation* (they authorized GitHub
  as a different user than the owner/admin of the target account — retry signed in as
  the right GitHub user); *already linked to installation N* (Unlink first).
- **Result with `repoConnectionStatus: "CONNECTED"`** — proceed to Step 3.
- **Result with `repoConnectionStatus: "INITIALIZATION_FAILED"`** — the repository was
  created but the baseline commit failed. Call the tool titled
  `Project Service Retry Repo Initialization` with `{"projectId": "<project id>"}`.
  Never call it in any other status — it fails unless the project is awaiting a retry.
  If the retry also fails, report the error to the user and stop; do not delete or
  recreate anything. Deleting a project cascades to its deployment and machine
  identities, so it is never a recovery step.

Record `repoFullName` (`owner/repo`) from the result — the repo is named after the
slugified project name, not the project id, so never derive it. If you are ever unsure
whether a project already exists for a repository, call the tool titled
`Project Service Find by GitHub Repo` with `{"repoFullName": "owner/repo"}`.

## Step 3 — Connect to GitHub and clone

The repository lives under the GitHub account or organization where the Kinotic GitHub
App is installed, and is private by default, so the user's own GitHub credentials must
have access to it.

1. Check GitHub access with whatever is available: `gh auth status` if the `gh` CLI is
   installed, or a connected GitHub MCP server. If neither is authenticated, ask the
   user to connect one (e.g. run `gh auth login`).
2. Verify the repository is reachable: `gh repo view <repoFullName>` (or the MCP
   equivalent). A 404 on a repository that was just created means the user's GitHub
   account lacks access — they need to accept an invitation or be granted access in
   the GitHub organization settings.
3. Clone it: `gh repo clone <repoFullName>` (or `git clone`).

## Step 4 — Verify the scaffold

Compare the cloned repository against `references/project-scaffold.md`. In short: a Bun
workspace with `packages/domain`, `packages/microservices/main`, `packages/ui`, and a
`.config/kinotic.config.ts` whose `organizationId` and `applicationId` match Step 1.

Run `bun install`, then `bun run type-check`. Check which `@kinotic-ai/*` versions
actually resolved (`bun pm ls | grep @kinotic-ai`) — the current SDK line is `5.x`
(`@kinotic-ai/core` 5.0.0-beta and up, `@kinotic-ai/management-api` alongside it). A
resolution on the old `4.x` line, or any `@kinotic-ai/os-api` (renamed to
`@kinotic-ai/management-api`), means the template's catalog pins are stale — report that
to the user rather than working around it. Report any mismatch between the clone and the
expected shape to the user instead of silently changing files — the repository contents
are the source of truth.

## Step 5 — First entity, first push

1. Define a first entity under the path listed in `.config/kinotic.config.ts`
   `entitiesPaths` (see the entities-and-persistence skill).
2. Run `bun run generate` from the project root to generate the typed repository
   classes. The script wraps the Kinotic CLI vendored as a project dependency and runs
   locally — no server connection or login. If the script is missing from
   `package.json`, tell the user instead of improvising.
3. Export the entity and its repository from `packages/domain/index.ts` — nothing else
   in the workspace can import them until you do — then run `bun run type-check`.
4. Commit and push everything generate produced: entity sources, the generated
   repository classes, and `.config/c3` (a committed generation artifact — keep it in
   git so schema changes show up in diffs; never gitignore it). Never run
   `kinotic login` or `kinotic sync` yourself.
5. **The push is the deployment.** A push to the default branch builds the project,
   synchronizes the entity definitions, and starts (or reloads) the runtime that runs
   `packages/microservices/main/src/main.ts`. Verify it landed with the tool titled
   `Project Service Find Deployment` (`{"projectId": "<project id>"}`) — poll until
   `status.type` is `RUNNING` with the pushed `commitSha`, and report a `FAILED`
   status's `message` to the user. Full workflow and failure handling: the `deploying`
   skill.
6. **Tell the user to publish the entity.** A synchronized entity definition has no
   backing storage until someone publishes it in the portal (Application → Project →
   **Entity Definitions** → publish). Until then every repository call fails, even with
   the deployment `RUNNING`. There is no MCP tool for this — wait for the user to
   confirm. Details and the additive-only rule that follows publication: the
   `entities-and-persistence` skill.

From here, hand off to the other kinotic skills: entity modeling and persistence →
`entities-and-persistence`; business logic and APIs → `services`; UI and client
connections → `frontend`; running, releasing, and diagnosing → `deploying`.
