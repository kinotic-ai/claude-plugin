# Expected Kinotic project scaffold

The repository provisioned by Kinotic OS is rendered from the Kinotic project template
(`kinotic-ai/kinotic-tpl-isomorphic-ts`, a Bun workspace mono repo). **The cloned
repository is the source of truth** — if it differs from this document, trust the clone,
report the difference to the user, and do not "fix" the clone to match this document.

## Layout

```
<repo root>/
├── README.md
├── package.json              # Bun workspace root
├── tsconfig.json             # `bun run generate` reads compilerOptions from here
├── tsconfig.base.json
├── bunup.config.ts           # bunup workspace definition (packages are registered here)
├── .config/
│   ├── kinotic.config.ts     # Kinotic project configuration (see below)
│   └── c3/                   # generated schemas (written by `bun run generate`; committed)
├── .kinotic/                 # local incremental-generation cache (gitignored — never commit)
├── migrations/               # V<N>__<description>.sql files (absent until first used)
└── packages/
    ├── domain/               # entity model + generated repositories
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── index.ts          # package entry — export entities/repositories here
    │   ├── model/            # @Entity classes go here
    │   └── repositories/     # `bun run generate` writes repository classes here
    ├── microservices/
    │   └── main/             # THE deployed entry point (packages/microservices/main/src/main.ts)
    └── ui/                   # frontend packages (built, but not hosted by the platform)
```

`packages/domain/model`, `packages/domain/repositories`, and `packages/ui` ship as empty
directories holding a `.gitkeep`.

## Root `package.json`

```json
{
    "name": "<project slug>",
    "private": true,
    "scripts": {
        "build": "bunup",
        "dev": "bunup --watch",
        "generate": "kinotic generate",
        "type-check": "bun run --filter '*' type-check"
    },
    "devDependencies": {
        "@kinotic-ai/kinotic-cli": "<pinned version>",
        "@kinotic-ai/management-api": "<pinned version>",
        "@types/bun": "<pinned version>",
        "bunup": "<pinned version>",
        "typescript": "<pinned version>"
    },
    "catalog": {
        "@kinotic-ai/core": "<pinned version>",
        "@kinotic-ai/management-api": "<pinned version>",
        "@kinotic-ai/persistence": "<pinned version>"
    },
    "workspaces": ["packages/*", "packages/microservices/*", "packages/ui/*"],
    "type": "module"
}
```

Workspace packages reference the catalog (`"@kinotic-ai/core": "catalog:"`) so the
Kinotic SDK version is pinned once at the root. The versions are not written by the
template — Kinotic OS injects the ones the server ships with when it renders the repo, so
a fresh project always matches its server.

The current SDK line is `@kinotic-ai/core` and `@kinotic-ai/persistence` on
`5.0.0-beta.x`, `@kinotic-ai/management-api` on `5.0.0-beta.x`, and the CLI on `5.2.x`.
After `bun install`, check what actually resolved (`bun pm ls | grep @kinotic-ai`);
anything on the 4.x line, or any `@kinotic-ai/os-api` (the package was renamed to
`@kinotic-ai/management-api`), means the pins are stale — report that to the user rather
than working around it.

The Kinotic CLI ships as a project dependency with a `generate` script wired in
`package.json` — repository classes are generated with `bun run generate`, entirely
locally. Server synchronization happens inside the deployment (`kinotic sync` runs in the
build VM on every push); the globally-installed CLI (`kinotic login`, `kinotic sync`) is a
human-operator tool that Claude does not use.

## `.config/kinotic.config.ts`

A TypeScript module default-exporting a `KinoticProjectConfig`:

```typescript
import type { KinoticProjectConfig } from '@kinotic-ai/management-api'

const config: KinoticProjectConfig = {
  organizationId: "<organization id>",
  applicationId: "<application id>",
  entitiesPaths: [
    {
      path: "packages/domain/model",
      repositoryPath: "packages/domain/repositories",
      mirrorFolderStructure: true
    }
  ],
  fileExtensionForImports: ".js",
  validate: false
}

export default config
```

- `organizationId` and `applicationId` must match the Application created through the
  MCP tools (Step 1 of the workflow). If they differ, synchronization will target the
  wrong application — surface this to the user before pushing.
- `entitiesPaths[].path` is where `@Entity` classes are discovered;
  `repositoryPath` is where repository classes are generated. Trust the values in the
  cloned config over the ones shown here.
- Optional fields the template leaves out: `name`, `description`, `generatedPath` (the
  fallback repository output for plain-string `entitiesPaths` entries).

## The microservice entry point

`packages/microservices/main/src/main.ts` is **the file the platform runs**. The template
ships it already wired: the zone prefix from the project config, a zero-arg
`await Kinotic.connect()`, an OpenTelemetry bootstrap gated on `OTEL_EXPORTER_OTLP_ENDPOINT`,
and a SIGTERM handler that flushes spans.

```typescript
import { Kinotic } from '@kinotic-ai/core'
import { appZone } from '@kinotic-ai/management-api'
import config from '../../../../.config/kinotic.config'

// The zone prefix must be set before any @Publish class is instantiated.
Kinotic.zonePrefix = appZone(config.organizationId, config.applicationId)

// Instantiate @Publish services here. Before or after connect() both work.
await Kinotic.connect()
```

Add services to this process. A second package under `packages/microservices/` is built
and type-checked but never started — see the `deploying` skill.

Its `package.json` depends on the domain package plus the catalog entries, and pins the
OpenTelemetry packages its tracing bootstrap uses:

```json
{
    "name": "@<project slug>/main",
    "version": "0.1.0",
    "type": "module",
    "scripts": { "type-check": "tsc --noEmit" },
    "dependencies": {
        "@<project slug>/domain": "workspace:*",
        "@kinotic-ai/core": "catalog:",
        "@kinotic-ai/management-api": "catalog:",
        "@opentelemetry/exporter-trace-otlp-grpc": "<pinned version>",
        "@opentelemetry/resources": "<pinned version>",
        "@opentelemetry/sdk-node": "<pinned version>",
        "@opentelemetry/semantic-conventions": "<pinned version>"
    }
}
```

## The domain package

`packages/domain/index.ts` is the barrel other packages import through; the template
ships it as `export {}`. An entity or repository is only importable once it is exported
here:

```typescript
export * from './model/Person.js'
export * from './repositories/PersonRepository.js'
```

Its `package.json` `exports` map lists the source conditions (`development`, `bun`) ahead
of the `dist` fallbacks, so inside the repository the package resolves to its TypeScript
source and type-checks and runs without a prior build. Consumers installing it from a
registry get the built `dist/` output instead. `bunup.config.ts` keeps that map in sync
and must be edited by hand if a new workspace package is added.

## Adding a UI package

Each UI is its own workspace package under `packages/ui/` — the root workspace globs
cover exactly that depth. Register it in `bunup.config.ts` and run `bun install` so the
workspace links it. Any framework that builds to static assets works; the platform does
not host the result (see the `deploying` skill), which also makes the UI cross-origin
with the gateway — how it resolves the server URL is in the `frontend` skill.

```jsonc
// tsconfig.json — three levels up to the base config
{
    "extends": "../../../tsconfig.base.json",
    "files": [],
    "include": ["**/*"],
    "exclude": ["node_modules", "dist", "bin"]
}
```

Production single-file executables are built from a package directory with
`bun build --compile src/main.ts --outfile bin/<name>` — use `bun build` directly, not
bunup's `compile` option, which drops an import binding when bundling `@kinotic-ai/core`
and produces a binary that crashes on startup.

## Verification checklist

1. `package.json` at the root declares the workspace globs shown above and the
   `@kinotic-ai/*` catalog entries.
2. `.config/kinotic.config.ts` exists, default-exports a config, and its
   `organizationId`/`applicationId` match the created Application.
3. The directories named by `entitiesPaths` exist.
4. `bun install` succeeds and resolves `@kinotic-ai/*` on the current 5.x line, with no
   `@kinotic-ai/os-api` anywhere.
5. `bun run type-check` succeeds.
