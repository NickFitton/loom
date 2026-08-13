# Define the initial implementation architecture and developer workflow

Type: grilling
Status: resolved
Blocked by: none

## Question

What repository layout, runtime boundaries, local-development workflow, and test harnesses let Loom build the first manual complete loop without weakening the established control-plane/execution-plane, Device, Project, Agent-session, and containment boundaries?

## Answer

Loom will start as a TypeScript pnpm workspace managed by Turborepo `2.10.9`. It has `apps/desktop` for the Electron Execution plane, `apps/control-plane` for the NestJS/Postgres Control plane, and separate packages for `contracts`, `domain`, `ui`, and test fixtures. `packages/ui` contains renderer-safe React, shadcn, styling, icon, and accessibility components only. It has no Electron, filesystem, credential, Pi, Git, or product-state access. Each app owns its own composition and data fetching.

`packages/contracts` owns Zod wire schemas and derives the TypeScript types used by Electron and NestJS. The Device/Control-plane protocol is versioned JSON over HTTPS, with separate command and report endpoints carrying idempotency keys, Ticket revisions, Agent-session IDs, and ordered local sequence numbers. `packages/domain` is framework-free and owns pure state transitions, revisions, and invariants; it imports neither the ORM nor runtime adapters.

The Control plane uses NestJS `11.1.29`, Drizzle ORM, SQL migrations, and Postgres. Its persistence adapters remain inside `apps/control-plane`. Docker Compose runs local Postgres only; developers run NestJS and Electron natively. Electron `43.4.0` uses Electron Forge with Vite, accepting the plugin's experimental status for its renderer HMR loop. pnpm is configured with `node-linker=hoisted` for Forge packaging. Electron main is the sole owner of an explicit execution-host interface and adapters for Git, secure credentials, Pi, VM containment, Device-local persistence, and Control-plane synchronization. The renderer calls typed, validated preload capabilities only.

Electron main stores its ordered local journal, Pi session pointers, command acknowledgements, and non-secret execution metadata in a small SQLite database behind the execution-host persistence adapter, using Node `24.17.0`'s built-in `node:sqlite`. Secrets remain in secure credential storage. Development and automated tests may use dedicated development/test identities plus fake Pi and containment adapters. Production implementation starts are refused unless the Account is verified, the Device lease is valid, secure credential storage is available, and the validated per-Agent-session Linux VM adapter is present.

The standard workflow provides `pnpm dev` to start Postgres, the Control plane, and Electron through Turbo, together with focused `dev:desktop`, `dev:api`, `test`, `test:integration`, and `test:e2e` commands. Tests are layered: Vitest unit tests for domain and contract behavior; NestJS integration tests against disposable real Postgres; Electron-main integration tests using temporary real Git repositories plus fake Pi/VM adapters; and a small Playwright Electron end-to-end suite for Navigator-to-Action behavior. Pull requests run formatting, linting, type checks, unit tests, Postgres integration tests, and desktop main-process/Git-fixture tests. Packaged Electron E2E and VM-containment tests run on macOS runners as nightly or release-candidate gates.

Direct dependencies use secure non-breaking ranges and the committed `pnpm-lock.yaml` fixes their exact resolved versions. The exception is the explicitly pinned Turborepo `2.10.9`, Electron `43.4.0`, and NestJS `11.1.29`. New minor or major updates require a Snyk-clean dependency update. This architecture is recorded in [ADR 0001](../../../docs/adr/0001-initial-implementation-architecture.md).
