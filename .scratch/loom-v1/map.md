# Loom v1 weave map

## Destination

Deliver Loom v1: a single-user Electron desktop platform that lets a person connect a device-local Git repository, begin a destination-led weave, and safely drive Pi agents through planning and implementation until human Actions are required.

## Notes

- This weave explicitly carries execution through to a working v1, rather than ending at a planning handoff.
- Use the Wayfinder, domain-modeling, grilling, prototype, research, and TDD skills as applicable. Consult `CONTEXT.md` before domain work.
- The NestJS/Postgres control plane stores account, device, project, weave, ticket, action, and session records but never accesses project files.
- Pi and Git execute only in Electron's device-local execution plane. Provider credentials remain in each device's secure credential store.
- A Project belongs to one Device in v1. It has no collaboration or cross-device availability.
- Agents work only in ticket-specific worktrees; pushes and any operation outside the Project require an explicit human Action.

## Decisions so far

- [Define the Weave, Ticket, Action, and agent-session state model](issues/02-define-weave-ticket-action-state-model.md) — Separate, audited Weave, Ticket, Action, and session state machines keep local context durable and every human next action explicit.
- [Research Pi integration for durable local agent sessions](issues/01-research-pi-integration.md) — Use pinned local Pi SDK sessions and exact JSONL session pointers; Pi needs external containment to enforce Loom's worktree boundary.
- [Research Electron security boundaries for local agent execution](issues/07-research-electron-security-boundaries.md) — Sandboxed renderer and a narrow validated main/preload capability bridge; Electron does not sandbox Pi or Git.
- [Prototype the action tree and action modal](issues/03-prototype-the-action-tree-and-modal.md) — Navigator + Detail: collapsible source list with roll-up panels at every level (All projects / Project / Weave / Ticket); action cards inline in the detail panel; agent progress as a timestamped activity log, not a chat transcript.
- [Define the device-local Project and Git lifecycle](issues/04-define-local-project-and-git-lifecycle.md) — Projects are explicit Device-local Git-clone connections; origins associate core repositories without deduplicating clones, and Agent worktrees are session-scoped, recoverable, and applied only through a human Action.
- [Define the account, device, and authentication contract](issues/05-define-account-device-authentication-contract.md) — Browser-based verified-email sign-in enrols secure Devices; a short Device lease bounds offline authority, while revocation deletes local authority and credentials.
- [Define the orchestration, scheduling, and recovery protocol](issues/06-define-orchestration-and-recovery-protocol.md) — Weaves automatically queue ready Tickets against Device capacity; versioned idempotent commands and conservative recovery keep cross-device control safe.
- [Define the safe agent command policy](issues/08-define-safe-agent-command-policy.md) — macOS-only enforced Agent containment, visible control-plane Capability profiles, scoped resource grants, and immutable Actions keep Pi work within its intended authority.
- [Research and validate macOS Agent containment](issues/10-research-macos-agent-containment.md) — Validate a per-session Linux VM with only capability-scoped shares and mediated services; block unattended implementation until adversarial escape gates pass.
- [Design v1 delivery slices and release gates](issues/09-design-v1-delivery-slices-and-release-gates.md) — First prove a manually started safe loop, then containment/recovery, then scheduling; validate it as a private developer preview, not a macOS app release.
- [Define the initial implementation architecture and developer workflow](issues/11-define-initial-implementation-architecture.md) — A pinned workspace separates Control plane, Electron Execution plane, contracts, domain rules, UI, and test fixtures; typed adapters and layered tests preserve Loom's safety boundaries.

## Not yet specified

- The concrete containment-validation harness and its macOS/CPU test matrix.
- Hosting, observability, and support expectations for the private preview.
- Whether the initial release needs a migration/import path for early local development data.

## Out of scope

- Multi-user collaboration, roles, and sharing a Project or Weave across Accounts.
- Accessing a Project from an Account's other Devices.
- Loom-managed model billing or backend custody of provider credentials.
- Public macOS application distribution, notarization, auto-update, and general-user support; the first preview is a developer-run build for its owner only.
