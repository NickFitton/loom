# Define the device-local Project and Git lifecycle

Type: grilling
Status: resolved
Blocked by: none

## Question

How does Loom discover and validate a local Git repository; detect moved, missing, dirty, or changed repositories; create and clean ticket worktrees; and present recovery options while keeping each Project bound to one Device?

## Answer

Loom treats a Project as one Device-owned connection to a particular local Git clone. A normalized `origin` (when configured) identifies the clone's core repository association, but it never makes Projects unique: a person may connect several clones with the same origin on one Device, or connect a clone on another Device as a separate Project. The Project, its paths, Agent worktrees, and Agent-session context never transfer automatically between Devices.

### Connecting, validation, and identity

- A person explicitly selects an existing, non-bare Git working tree; Loom does not initialize Git for an ordinary directory in v1.
- Electron main resolves the selected path with Git and accepts it only when it is the repository's actual top-level working directory. It records the canonical local path, the current Git status and `HEAD`, and a normalized `origin` when present.
- On first connection, Loom writes a generated `loom.repositoryId` into the local repository's shared Git config. The value is not tracked by Git and moves with the local repository's `.git` directory. It identifies this exact local clone; `origin` identifies the associated core repository.
- Adding a new Project with an origin already known to Loom simply adds the independent clone without a duplicate warning. During a relink flow, a matching origin with a different or absent local ID is reported as a different local clone and can be added separately, never silently substituted for the missing Project.

### Repository availability and change detection

- Loom never searches a Device for a moved or missing Project. It marks the Project unavailable, retains its history, and asks the person to select a replacement path. The replacement must pass the same validation and match the local repository ID to relink the Project.
- A dirty primary checkout, changed branch, or changed `HEAD` is displayed as current Project status and does not prevent new Agent sessions. Loom records the exact base commit when it starts a session, so later primary-checkout changes cannot alter that session's basis.
- A changed `origin` leaves the Project available but is flagged as an unconfirmed core-repository change. Loom updates the recorded association only after the person confirms it.

### Agent worktrees and completion

- Starting an implementation Agent session creates a dedicated Agent worktree and Loom-managed branch from the primary checkout's current `HEAD`. The base branch and exact commit are shown to the person and recorded for the Ticket/session.
- A paused or otherwise resumable session keeps and reuses its Agent worktree. If a session reaches terminal `failed`, its worktree is preserved for inspection and a replacement Agent session for the Ticket starts in a fresh worktree.
- An Agent works only in its Agent worktree. When its work is accepted, Loom does not merge automatically: a person must authorize an explicit Apply-to-primary-checkout Action. That Action performs the merge and reports conflicts.
- After a successful apply, Loom automatically removes the merged Agent worktree and branch; the merged commit remains in the primary checkout. Rejected, cancelled, and failed worktrees remain until explicitly discarded. If automatic cleanup cannot safely remove a worktree, Loom never force-deletes it; it reports the failure and offers an explicit human cleanup Action.

### Recovery

- If an active or resumable session's Agent worktree is missing, moved, or cannot be validated, Loom does not search for it. It records the Git diagnostics, marks the session unrecoverable and the Ticket `failed`, and lets the person start a replacement Agent session in a fresh worktree.
- The primary checkout is treated as Loom-read-only except for a person-authorized apply Action. Local work may continue there independently; its state remains visible to the person.
