# Research and validate macOS Agent containment

Type: research
Status: resolved
Blocked by: 08

## Question

Which currently supported macOS execution boundary can enforce Loom's Agent containment for a Pi Agent session: writable access only to its allocated Agent worktree and local session data, read-only access only to explicitly granted source snapshots, and only explicitly approved network services? Compare viable primary-source-supported approaches for a signed Electron application, identify their limits around child processes, filesystem traversal and symlinks, credential exposure, and network access, then specify adversarial tests that must pass before unattended implementation sessions may run.

## Answer

Select a per-Agent-session Linux VM built with Apple's Virtualization framework as the macOS Agent containment candidate. Pi and all of its commands run in the guest, which receives only the allocated Agent worktree and session-data shares, explicitly granted read-only source snapshots, a fixed host/guest control socket, and no network device by default. App Sandbox helps protect Loom but is not a safe per-session Pi boundary: a child inherits the parent's static sandbox rights; `sandbox-exec` and `sandbox_init` are deprecated. NAT is not approved-service egress; a separately constrained host proxy is required for a Ticket with an approved network capability.

No unattended implementation Agent session may run until the signed packaged build passes the documented adversarial gates for VM provisioning, process-tree termination, host-file/traversal/symlink denial, share write scope, credential absence, denied and mediated network egress, and host-control-channel integrity on every supported macOS/CPU combination. Full findings and sources: [macOS Agent containment research](../research/macos-agent-containment.md).
