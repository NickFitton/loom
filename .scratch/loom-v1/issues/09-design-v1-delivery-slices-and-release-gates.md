# Design v1 delivery slices and release gates

Type: grilling
Status: resolved
Blocked by: 03, 05, 06, 08

## Question

Given the settled product, UX, data, Pi, Git, security, and orchestration decisions, what is the smallest implementation sequence that proves the complete Loom loop, and which automated, integration, end-to-end, packaging, and release gates make v1 safe to ship?

## Answer

Loom v1 will first prove one complete, manually started safe loop: a person uses a verified Account on an enrolled Device, connects a local Project, creates a Weave and Ticket, starts a contained Agent session in the Ticket's Agent worktree, reviews its result, and approves an Apply-to-primary-checkout Action. This is the first product boundary; it must work before scheduling, queue controls, or broader management features are treated as v1 progress.

The delivery sequence is:

1. **Foundation.** Build the Electron execution plane and NestJS/Postgres control plane with the Account, Device, Project, Weave, Ticket, Action, and Agent-session records, audit history, authenticated Device lease, and the Navigator + Detail UI shell.
2. **Manual complete loop.** Connect and validate a Project, create its Git worktree, run Pi from a narrow Electron-main host, retain the Agent session context, show human Actions, and apply an accepted result only through the explicit Apply-to-primary-checkout Action.
3. **Containment and recovery.** Run Agent sessions in the selected per-session Linux VM boundary. Before any automatic implementation start, a locally built signed validation artifact must pass the VM adversarial containment gates. Complete the versioned/idempotent journal, lease-expiry pause, and restart recovery for a retained Agent session and Action.
4. **Automation.** Add Device capacity, first-ready ordering and reordering, scheduling enablement, and conservative reconciliation. Automation may start only when the containment and recovery gates above pass.
5. **Private preview.** Test the full loop on real supported macOS hardware with a developer-run build. This is for the owner only. Loom will not yet distribute, notarize, auto-update, or support a public macOS application release.

The private-preview gate requires: unit tests for state transitions and idempotency; integration tests for control-plane authorization, lease, audit, and Action behavior; Electron end-to-end tests for the manual complete loop and recovery; Git fixture tests for unavailable, dirty, relinked, and conflicting repositories; and the documented containment adversarial tests on every supported macOS and CPU combination. A failed containment, lease, recovery, or Apply Action gate blocks automatic implementation scheduling and preview use of that capability.
