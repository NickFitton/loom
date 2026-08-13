# Initial implementation architecture

Loom uses a pnpm/Turborepo TypeScript workspace: the NestJS/Postgres Control plane and Electron Execution plane are separate applications; contracts, pure domain rules, renderer-only UI, and test fixtures are separate packages. Electron main owns Device-local Project, Git, Pi, credential, persistence, and containment adapters; its sandboxed renderer uses only a typed preload bridge. This preserves Loom's Control-plane/Execution-plane, Device, Project, Agent-session, and Agent containment boundaries while allowing each boundary to have focused tests.

The workspace uses Electron Forge with Vite despite its experimental status, because it provides the preferred desktop development loop. Direct dependencies use secure non-breaking ranges and a committed exact lockfile, except Turborepo 2.10.9, Electron 43.4.0, and NestJS 11.1.29, which are pinned. New minor or major dependency updates require a Snyk-clean change.

