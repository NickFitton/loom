# Loom context

## Glossary

### Weave

A scoped journey for one Project that starts from a destination, maps the decisions and work needed to reach it, and—in the initial Loom v1—continues through implementation rather than ending at a planning handoff. Its lifecycle is explicit and separate from its Ticket summary, because Tickets may run and wait independently. Scheduling is enabled by default. A Weave independently enables or disables scheduling of its ready Tickets; disabling scheduling prevents new Agent sessions but does not pause current work. Its ready Ticket queue is first-ready by default and a person may reorder it. It becomes completed automatically only when every one of its Tickets is completed; a cancelled Ticket prevents completion until it is replaced or removed from the Weave. Cancelling a Weave cascades cancellation to all unfinished Tickets and becomes effective only once affected device-local sessions have stopped. Pausing a Weave pauses only its currently running sessions; resuming it resumes only the sessions paused by that command.

### Ticket

A discrete unit of planning or implementation work within a Weave. A new Ticket is `ready` until Loom starts its first Agent session; the Weave scheduler may do this automatically when the owning Device has capacity. A Ticket may have at most one active Agent session, while other Tickets in the same Weave may run their own Agent sessions concurrently. An Agent's completed session awaits human review; the Ticket is completed only when a person accepts its result. When its current session is unrecoverably failed, the Ticket is `failed` until a person explicitly starts a replacement session. A person may pause and resume a Ticket's active session. Cancelling a Ticket is terminal and cancels its pending Action and active session. Work that must continue after cancellation is a new, linked replacement Ticket rather than a restoration of the cancelled one.

### Agent session

A device-local attempt by an Agent to advance one Ticket on one Device. Loom attempts to recover an interrupted session from its Yesble local context after events such as network loss or device shutdown. It continues automatically only when it can reopen that exact retained context with a valid Device lease; a replacement session is always explicit. A temporary uncertainty leaves the Ticket waiting for human input with its Agent session paused; an unrecoverable session reaches terminal `failed`. A person may pause and later resume the same session from its retained local context. Finished, failed, cancelled, or superseded attempts remain as the Ticket's session history, and only one attempt may be active at a time.

### Action

A human-authorized operation an Agent requests from its Ticket when it cannot safely proceed on its own. An Action starts as `requested`; approval makes it `running`, and success makes it `completed`. It may instead end as `declined`, `failed`, or `cancelled`. A human response applies only to the exact current Action state; a stale response is void, changes nothing, and informs its sender. An Action requested by a paused Ticket remains visible but cannot be approved until the Ticket resumes. While an Action awaits a person, its Ticket is waiting for human input and its active Agent session is paused with its local context retained. Once approved, the Ticket waits for the Action's result while that session remains paused. Only a completed Action resumes that session; every other terminal Action outcome remains immutable history and requires a new Action to retry. Cancelling an Action does not discard the Ticket or its paused session.

### Project

A Device-owned local Git repository connection, identified by a user-provided name and filesystem path. A Project is available only on its owning Device in Loom v1; a remote Git origin is not required.

### Device

A user-controlled computer enrolled by a verified Account. It can be named and removed from the Account, except for the Device currently in use. Removal is terminal for that Device identity; a later sign-in enrols a new Device. A Device may temporarily be offline without being removed, and has a configurable capacity, defaulting to two, for concurrently running Agent sessions. Reducing its capacity affects later starts only.

### Idempotency key

An identifier for one requested state change. Loom records its outcome so a repeated delivery returns the earlier outcome instead of repeating the change.

### Device lease

The time-limited authority an enrolled Device holds to execute Agent work and synchronize Loom state. Its expiry pauses that authority but does not remove or revoke the Device.

### Control plane

The hosted NestJS application and Postgres database that store Loom records and coordinate authenticated devices. It does not access a Project's filesystem or run Git commands.

### Execution plane

The Electron application on a Device. It validates and operates the local Project, launches and controls Pi agent sessions, and reports their state to the control plane.

### Account

The single-user identity authenticated through Better Auth. An Account owns its Devices but, in Loom v1, does not share Projects or Weaves with other Accounts.

### Provider credential

A user-supplied model-provider API key, stored only in the secure credential store of the Device that uses it. It is never sent to Loom's control plane and is deleted when that Device is removed.

### Agent worktree

A Git worktree and branch created for an implementation Ticket's active Agent session. A paused or otherwise resumable session retains and reuses its Agent worktree. After a terminal session failure, a replacement Agent session receives a new Agent worktree; the failed session's worktree remains preserved for inspection until a person explicitly cleans it up. An agent may read and modify files only within its Agent worktree. It may not push a remote, change machine-wide settings, or access files outside the Project unless the user triggers an explicit Action.

### Agent containment

The technically enforced execution boundary for an Agent session. It exposes only the session's allocated Agent worktree, its required local session data, and explicitly approved services; an Agent's working directory or prompt is not itself containment. Operations beyond that boundary require a distinct human Action.

### Capability profile

The control-plane-owned baseline authority assigned to a Ticket before an Agent session begins. It determines the resources and operation classes the session may use automatically. A profile does not grant access to a Project merely because it names a kind of work; Project access is an explicit scoped capability. An Agent may request additional, narrowly scoped capabilities only through a human Action.

### Prototype artifact

The retained, inspectable output of a prototype Ticket, such as an HTML file. It is created in the Ticket's isolated prototype workspace, not in a Project. The control plane retains it as untrusted Ticket data so the Account's Devices can surface it for an Action. A person may review it during the Ticket and later, and a later Agent may receive it through an explicit read-only capability.

### Prototype artifact policy

A Project-owned setting that limits how many Prototype artifacts one prototype Ticket may retain for review. It makes the permitted prototype exploration visible before the Agent session starts; it does not grant a Prototype Agent access to Project files.
