# Define the orchestration, scheduling, and recovery protocol

Type: grilling
Status: resolved
Blocked by: 01, 02, 04, 05

## Question

How does the control plane schedule unblocked tickets on their owning Device, limit concurrent agents, turn a Pi request into an inspectable Action, pause/resume a ticket, reconcile local and server state after restart or disconnect, and preserve safe execution ordering?

## Answer

The control plane owns durable intended state, scheduling queues, commands, and audit history. The execution plane owns its local Agent processes, Pi context, worktrees, and observations. Neither plane guesses the other's outcome: they exchange versioned, idempotent records and choose a safe human-visible state when their observations disagree.

### Scheduling and capacity

- A new Weave has scheduling enabled. The Weave controls whether Loom may automatically start its ready Tickets; disabling it prevents new starts only and never pauses current work.
- The owning Device controls capacity, with a default of two concurrently running Agent sessions. It schedules ready Tickets in first-ready order, while a person may reorder the queue.
- A ready Ticket stays ready when its Device is offline, lacks a valid Device lease, or has no capacity. The control plane may remember that it is next, but it becomes `active` only when the owning Device creates and confirms its Agent session.
- Agent sessions waiting for an Action or other human input retain their context but do not consume Device capacity. Reducing capacity similarly prevents later starts; it never interrupts already running Agent sessions.

### Commands, Actions, and human control

- Every command and report carries an idempotency key, Ticket revision, Agent-session ID, and ordered local sequence number. The receiving plane records the outcome once; a duplicate returns the earlier outcome, and a stale command cannot change a later Ticket revision.
- When Pi requests an Action, the execution plane first pauses the Agent session and persists its local context, then publishes the `requested` Action. A network loss leaves the Agent paused and the Action is synchronized later.
- Any enrolled Device may review and approve an Action. Its approval applies to the exact current Action state. The control plane voids a stale answer without changing state and tells that reviewing Device why; the owning Device also rechecks its local Project conditions before execution. A changed Action or failed local precondition does not run and leaves a clear human next step.
- A requested Action on a paused Ticket remains visible but cannot be approved. Resuming returns the Ticket to `waiting_for_human` for that Action. Cancelling the Ticket cancels its pending Action.
- Any enrolled Device may request pause or cancellation. The Ticket remains `pausing` or `cancelling` until the owning Device confirms the Agent session stopped. If the Device is offline, Loom shows the request as pending rather than claiming a local process has stopped.

### Recovery and synchronization

- The execution plane keeps a durable ordered local journal of state summaries and sends missing entries after reconnection. The control plane accepts each matching Agent-session record once, persists it in the audit history, and returns its current versioned commands. Pi transcripts and provider credentials never synchronize.
- An interrupted Agent session automatically continues only when Loom can reopen that exact retained Pi context and the Device has a valid lease. This is recovery of existing work, not a new scheduled start; Loom never creates a replacement Agent session automatically.
- A temporary uncertainty puts the Ticket in `waiting_for_human` with its Agent session paused. After Loom has tried and cannot reopen the exact retained context, the session is terminally `failed` and the Ticket is `failed`; a person may then start a replacement session.
- A control-plane cancellation that was issued while the owning Device was offline takes precedence over a later reported local completion. Loom preserves that local result in Agent-session history, but the Ticket remains cancelled and Loom neither accepts nor applies the result automatically.
- Device-lease expiry and Device revocation use the account/device contract: expiry pauses authority after its allowed offline period, and a received revocation stops local work immediately. The control plane rejects synchronization from a revoked or expired Device independently of what the Device reports.
