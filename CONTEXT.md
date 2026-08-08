# Loom context

## Glossary

### Weave

A scoped journey for one Project that starts from a destination, maps the decisions and work needed to reach it, and—in the initial Loom v1—continues through implementation rather than ending at a planning handoff.

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

A Git worktree and branch created for one implementation ticket. An agent may read and modify files only within this worktree. It may not push a remote, change machine-wide settings, or access files outside the Project unless the user triggers an explicit Action.
