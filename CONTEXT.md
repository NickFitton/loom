# Loom context

## Glossary

### Weave

A scoped journey for one Project that starts from a destination, maps the decisions and work needed to reach it, and—in the initial Loom v1—continues through implementation rather than ending at a planning handoff. Its lifecycle is explicit and separate from its Ticket summary, because Tickets may run and wait independently. It becomes completed automatically only when every one of its Tickets is completed; a cancelled Ticket prevents completion until it is replaced or removed from the Weave. Cancelling a Weave cascades cancellation to all unfinished Tickets and becomes effective only once affected device-local sessions have stopped. Pausing a Weave pauses only its currently running sessions; resuming it resumes only the sessions paused by that command.

### Ticket

A discrete unit of planning or implementation work within a Weave. A new Ticket is `ready` until a person starts its first Agent session. A Ticket may have at most one active Agent session, while other Tickets in the same Weave may run their own Agent sessions concurrently. An Agent's completed session awaits human review; the Ticket is completed only when a person accepts its result. When its current session is unrecoverably failed, the Ticket is `failed` until a person explicitly starts a replacement session. A person may pause and resume a Ticket's active session. Cancelling a Ticket is terminal and cancels its pending Action and active session. Work that must continue after cancellation is a new, linked replacement Ticket rather than a restoration of the cancelled one.

### Agent session

A device-local attempt by an Agent to advance one Ticket on one Device. Loom attempts to recover an interrupted session from its Yesble local context after events such as network loss or device shutdown. An unrecoverable session reaches terminal `failed`; a person may pause and later resume the same session from its retained local context. Finished, failed, cancelled, or superseded attempts remain as the Ticket's session history, and only one attempt may be active at a time.

### Action

A human-authorized operation an Agent requests from its Ticket when it cannot safely proceed on its own. An Action starts as `requested`; approval makes it `running`, and success makes it `completed`. It may instead end as `declined`, `failed`, or `cancelled`. While an Action awaits a person, its Ticket is waiting for human input and its active Agent session is paused with its local context retained. Once approved, the Ticket waits for the Action's result while that session remains paused. Only a completed Action resumes that session; every other terminal Action outcome remains immutable history and requires a new Action to retry. Cancelling an Action does not discard its paused session or Ticket.

### Project

A Device-owned local Git repository connection, identified by a user-provided name and filesystem path. A Project is available only on its owning Device in Loom v1; a remote Git origin is not required.

### Device

A user-controlled computer that has authenticated to an Account. It can be named and removed from the Account, except for the device currently in use.

### Control plane

The hosted NestJS application and Postgres database that store Loom records and coordinate authenticated devices. It does not access a Project's filesystem or run Git commands.

### Execution plane

The Electron application on a Device. It validates and operates the local Project, launches and controls Pi agent sessions, and reports their state to the control plane.

### Account

The single-user identity authenticated through Better Auth. An Account owns its Devices but, in Loom v1, does not share Projects or Weaves with other Accounts.

### Provider credential

A user-supplied model-provider API key, stored only in the secure credential store of the Device that uses it. It is never sent to Loom's control plane.

### Agent worktree

A Git worktree and branch created for an implementation Ticket's active Agent session. A paused or otherwise resumable session retains and reuses its Agent worktree. After a terminal session failure, a replacement Agent session receives a new Agent worktree; the failed session's worktree remains preserved for inspection until a person explicitly cleans it up. An agent may read and modify files only within its Agent worktree. It may not push a remote, change machine-wide settings, or access files outside the Project unless the user triggers an explicit Action.
