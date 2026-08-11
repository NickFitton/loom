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

## Not yet specified

- The implementation sequence and acceptance gates once the execution architecture, data model, user flow, and security boundaries are decided.
- Packaging, distribution, hosted deployment, observability, recovery, and support expectations appropriate for releasing Loom v1.
- Whether the initial release needs a migration/import path for early local development data.

## Out of scope

- Multi-user collaboration, roles, and sharing a Project or Weave across Accounts.
- Accessing a Project from an Account's other Devices.
- Loom-managed model billing or backend custody of provider credentials.
