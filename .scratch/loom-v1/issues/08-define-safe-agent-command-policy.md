# Define the safe agent command policy

Type: grilling
Status: resolved
Blocked by: 01, 04, 07

## Question

Which Pi agent capabilities and command classes are allowed automatically within an Agent worktree, which create a human Action, how are approvals recorded, and what OS/container/VM containment plus Electron and shell controls make the worktree boundary real rather than merely advisory?

## Answer

Loom makes Agent containment a hard product boundary, not a best-effort policy. In v1 it supports macOS only. Pi's working directory, prompts, and its own tool policy do not enforce this boundary: Loom must select and validate a macOS containment mechanism before it starts unattended implementation Agent sessions. A separate research ticket will select that mechanism and define the escape tests; until those tests pass, unattended implementation execution is unavailable.

### Capability profiles and scheduling

- A Ticket has a control-plane-owned Capability profile. It sets the baseline resources and operation classes available to its Agent session.
- An Agent may propose a profile or an additional capability, but a Ticket created by an Agent is unschedulable until a person reviews and accepts the proposed profile and capabilities. A person-created Ticket may use an already accepted profile and schedule normally.
- Loom shows a Ticket's profile, scoped capabilities, requested escalations, and reasons in its detail view. The Project's primary checkout is never implicit access.
- Any profile may receive a separately approved Project-read capability. It exposes a read-only, commit-pinned source snapshot with credentials and Git metadata excluded; it is not the primary checkout or an Agent worktree.

### Profile authority

- An implementation profile may write only in its dedicated Agent worktree. It may use a controlled shell inside Agent containment for normal development work, including Git inspection, formatting, builds, tests, linting, and dependency tooling. Loom resolves executables from fixed locations, provides a minimal environment, applies time/output/resource limits, and records each invocation. It does not rely on command-string matching as its security boundary.
- A research profile uses mediated, destination-allowlisted web research by default and has no Project access unless it receives the separate Project-read capability. Its canonical findings and citations are control-plane records; its local scratch material is not Project data.
- A prototype profile may write in an isolated prototype workspace but has no Project access unless it receives the separate Project-read capability. It may create HTML and related prototype assets there.
- Implementation network access is deny-by-default. It can receive only approved package registries and exact documented origins/versions attached to the Ticket. Other network access remains denied.

### Actions, artifacts, and audit

- Any authority beyond the session's accepted profile and capabilities requires one immutable, inspectable human Action. An approval has exact targets, arguments, expected effects, expiry, and audit records; it cannot become a reusable allow-all grant.
- A Prototype artifact is untrusted Ticket data. The control plane may retain it so the Account's Devices can surface it for later review or a later Action. It applies strict authorization, type/size limits, integrity checks, encryption at rest and in transit, and untrusted-content handling.
- A Project's Prototype artifact policy sets the permitted number of retained artifacts per prototype Ticket. A prototype session requests a bounded Store Prototype Artifacts Action that lists the exact immutable files, including type, size, and checksum. On approval, Electron uploads those files; the Agent has no general control-plane upload capability.
- Loom displays retained HTML only in a separate sandboxed preview with no `window.loom` bridge, Node access, filesystem access, or network access by default. The artifact is data, never trusted Loom UI.
