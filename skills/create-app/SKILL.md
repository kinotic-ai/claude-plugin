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

Kinotic currently supports one project per application, scaffolded as a single
Bun-workspace mono repo. The project's GitHub repository is created by Kinotic OS —
never create the repository yourself.

## Step 0 — Sign up or sign in to Kinotic OS

Check whether the kinotic-os MCP tools are available, e.g. the tool named
`os-api.org.kinotic.os.api.services.ApplicationService.createApplicationIfNotExist`.
If they are, the user is already connected — skip to Step 1.

If the tools are missing, the user has not authenticated the `kinotic-os` MCP server
yet. Ask whether they already have a Kinotic OS account, then walk them through the
matching path — the whole flow happens in their browser, so narrate what they will see:

1. Tell the user to run `/mcp`, select `kinotic-os`, and authenticate. A browser opens
   on the Kinotic OS login page. The plugin connects to Kinotic OS Cloud
   (`https://api.kinotic.ai/mcp`); a self-hosted or local server is added manually
   instead: `claude mcp add --transport http kinotic-os http://localhost:58503/mcp`.
2. **Existing account** — sign in, approve the consent screen, done.
3. **No account yet** — click **Create an organization** on the login page:
   - Sign up with GitHub, or with email and password (the email path sends a
     verification email — complete it before continuing).
   - Pick an organization name on the registration page. This creates the Kinotic
     organization that will own every application and project.
   - Invited to an existing organization instead? Accept the emailed invitation link
     rather than creating a new organization.
   - The browser returns to the OAuth consent screen — approving it completes the
     connection.
4. Back in Claude Code, confirm the kinotic-os tools are now available before moving on.

For a brand-new organization, tell the user up front that one more browser step is
coming: before the first project can be provisioned, the organization must link GitHub
(Step 2 fails with "GitHub is not linked" until then). Mentioning it now avoids the
surprise later.

If tools still do not appear right after a server restart, wait and retry: the service
directory publishes tools shortly after startup, not instantly.

## Step 1 — Create the Application

1. Ask the user for an application name and one-line description if not already known.
   The name must start with a letter and contain only letters, numbers, periods,
   underscores, or dashes.
2. Call `os-api.org.kinotic.os.api.services.ApplicationService.createApplicationIfNotExist`
   with `{"name": ..., "description": ...}`. The call is idempotent — if the application
   already exists it is returned unchanged.
3. From the result, record `id` (the server-minted slug of the name, e.g.
   `Inventory App` → `inventory-app`) and `organizationId`. Both are needed in Step 2.

## Step 2 — Create the Project (provisions the GitHub repository)

Call `os-api.org.kinotic.os.api.services.ProjectService.createProjectIfNotExist`. The
single argument key is exactly `project`:

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

Creation provisions a GitHub repository from the Kinotic template through the
organization's GitHub App installation and commits a rendered baseline. The call is
idempotent: re-running it returns the existing project unchanged.

Handle the outcomes:

- **Error containing "GitHub is not linked for this organization"** — the user's Kinotic
  organization has not installed the Kinotic GitHub App. This is the expected state for
  an organization created moments ago in Step 0. Direct the user to the Kinotic OS
  dashboard in their browser (the same site where they approved the OAuth consent),
  open the organization settings, and link GitHub — they choose the GitHub account or
  org to install the Kinotic GitHub App into, and that is where project repositories
  will be created. Then re-run the exact same tool call.
- **Result with `repoConnectionStatus: "CONNECTED"`** — proceed to Step 3.
- **Result with `repoConnectionStatus: "INITIALIZATION_FAILED"`** — the repository was
  created but the baseline commit failed. Call
  `os-api.org.kinotic.os.api.services.ProjectService.retryRepoInitialization` with
  `{"projectId": "<project id>"}`. Never call it in any other status — it fails unless
  the project is awaiting a retry. If the retry also fails, report the error to the user
  and stop; do not delete or recreate anything.

Record `repoFullName` (`owner/repo`) from the result. If you are ever unsure whether a
project already exists for a repository, call
`os-api.org.kinotic.os.api.services.ProjectService.findByRepoFullName` with
`{"repoFullName": "owner/repo"}`.

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
workspace with `packages/domain`, `packages/microservices`, `packages/ui`, and a
`.config/kinotic.config.ts` whose `organizationId` and `applicationId` match Step 1.

Run `bun install`, then `bun run type-check`. Report any mismatch between the clone and
the expected shape to the user instead of silently changing files — the repository
contents are the source of truth.

## Step 5 — First entity and handoff

1. Define a first entity under the path listed in `.config/kinotic.config.ts`
   `entitiesPaths` (see the entities-and-persistence skill).
2. Run `bun run generate` from the project root to generate the typed repository
   classes. The script wraps the Kinotic CLI vendored as a project dependency and runs
   locally — no server connection or login. If the script is missing from
   `package.json`, tell the user instead of improvising.
3. Commit and push. Kinotic OS synchronizes the project from its connected GitHub
   repository — never install the Kinotic CLI globally or run `kinotic login` /
   `kinotic sync` yourself.

From here, hand off to the other kinotic skills: entity modeling and persistence →
`entities-and-persistence`; business logic and APIs → `services`; UI and client
connections → `frontend`.
