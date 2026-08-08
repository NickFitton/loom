# Research: Pi integration for durable local agent sessions

## Question

What documented Pi APIs, CLI capabilities, session/transcript formats, authentication mechanisms, and lifecycle controls can Loom safely use from Electron to start, observe, interrupt, and resume local agents? What integration architecture and version constraints follow?

## Conclusion

Loom should run Pi **locally on the owning device**, in an Electron-main-owned Node agent host, and use Pi's TypeScript SDK as the primary integration. Pi documents the SDK specifically for desktop/custom UI embedding, and documents `AgentSessionRuntime` for session replacement (resume, new, fork, import) when `cwd`-bound state must be rebuilt. A short-lived host process per active ticket is still useful for Loom's pause/recovery boundary: persist the Pi session file path and ID locally, stop the host after `abort()`, then create a fresh runtime for the ticket's worktree and reopen that exact session on resume. [Pi SDK](https://pi.dev/docs/latest/sdk)

Pi's documented RPC mode is a viable compatibility boundary if Loom chooses process isolation over an in-process SDK host. It is a JSON-over-stdin/stdout protocol designed for embedded custom UIs. It should not be treated as the primary TypeScript API, because Pi's own RPC documentation advises Node/TypeScript callers to consider `AgentSession` directly. [Pi RPC mode](https://pi.dev/docs/latest/rpc)

## Supported integration surface

### Start and observe

- Install/package Pi as `@earendil-works/pi-coding-agent`; the SDK is included in that package. Create an `AgentSession` with the ticket worktree as `cwd`, then call `session.prompt()`. It exposes the generated `sessionFile`, `sessionId`, state, message history, and streaming status. [Pi SDK installation and `AgentSession`](https://pi.dev/docs/latest/sdk)
- Subscribe to `AgentSessionEvent` for assistant text/thinking deltas, tool start/update/end, message and turn boundaries, queues, compaction, and retry lifecycle. This is sufficient to drive Loom's action modal and an inspectable agent activity feed. [Pi SDK events](https://pi.dev/docs/latest/sdk)
- If a subprocess protocol is preferred, start `pi --mode rpc --session-dir <Loom-controlled-local-dir>`. Commands are JSONL on stdin and responses/events are JSONL on stdout; request IDs correlate commands with responses. The client must implement LF-only framing rather than Node's generic `readline`, which is explicitly documented as non-compliant for this protocol. [Pi RPC protocol and framing](https://pi.dev/docs/latest/rpc)
- In RPC mode, `prompt`, `get_state`, `get_messages`, `get_tree`, `get_session_stats`, and the streamed lifecycle/tool events supply the equivalent control and observation surface. `message_end` is authoritative for a completed message; clients assemble live partial text from streaming events. [Pi RPC prompting and state](https://pi.dev/docs/latest/rpc)

### Interrupt, pause, and resume

- `AgentSession.abort()` is the documented SDK operation to abort the current operation; RPC exposes the same behavior as `{"type":"abort"}`. `abort_bash` is additionally available for a direct RPC bash command. A Loom pause should request abort, wait for the resulting settled state/event or bounded timeout, record that it was paused, and terminate/dispose the local host. [Pi SDK lifecycle](https://pi.dev/docs/latest/sdk) and [Pi RPC abort](https://pi.dev/docs/latest/rpc)
- A message submitted during a run must explicitly be a steering or follow-up message. Steering is delivered after the current assistant turn's tool calls; follow-up is delivered only after the agent stops. Loom should expose these as distinct user intentions rather than silently queueing an arbitrary prompt. [Pi SDK prompt queueing](https://pi.dev/docs/latest/sdk)
- Pi auto-saves durable sessions unless `--no-session` is used. CLI sessions are JSONL files, organized by working directory; `--session <path|id>` opens one specific session. CLI/RPC also support a custom `--session-dir`, which Loom should set to a per-device application-data location rather than Pi's default global directory. [Pi sessions](https://pi.dev/docs/latest/sessions) and [Pi RPC options](https://pi.dev/docs/latest/rpc)
- For SDK session replacement, use `AgentSessionRuntime` rather than trying to mutate an old `AgentSession`: Pi documents that the runtime owns new, switch, fork, and import flows, changes `runtime.session`, and requires consumers to resubscribe after replacement. [Pi runtime replacement](https://pi.dev/docs/latest/sdk)
- In the RPC alternative, resume using the stored exact session path and `switch_session`; do not use “continue most recent,” because Loom needs deterministic ticket-to-session ownership. Pi's session format documents the session ID and cwd in its header, so Loom can verify both before resuming. [Pi session file format](https://pi.dev/docs/latest/session-format)

## Persistence and data model

Pi session files are append-oriented JSONL transcripts with a versioned header and tree-shaped entries (`id`/`parentId`). The current documented format is v3; Pi says older v1/v2 sessions are automatically migrated when loaded. The header includes the session UUID, cwd, timestamp, and optional parent-session path. Assistant/tool messages can include model/provider, token/cost data, structured content, and tool results. [Pi session file format](https://pi.dev/docs/latest/session-format)

For each Loom ticket-agent attempt, persist backend metadata such as: owning device ID, project/worktree identity, Pi package version, model/provider selection, session ID, local session-file reference, status, process start/stop/heartbeat, latest materialized event cursor, and terminal reason. Keep the raw JSONL transcript local to the owning device: it is the local continuation artifact and can contain prompts, source excerpts, tool output, or secrets. The hosted backend should receive the UI-safe event summary/metadata needed for Loom's tree, not the full transcript by default.

Use Pi's tree/messages APIs or the local transcript to rehydrate UI after a restart. Treat Pi's session file as Pi-owned: Loom may retain and reference it, but should not write arbitrary transcript entries or rely on undocumented fields. Pi does document extension state through `appendEntry()`/custom entries, but Loom does not need an extension merely to associate its own ticket ID; its own database/local metadata is a cleaner boundary. [Pi extensions: state management](https://pi.dev/docs/latest/extensions)

## Authentication and secrets

Pi supports provider OAuth/subscription login and API-key providers. Its standard credential resolution includes runtime API-key overrides, stored `auth.json`, environment variables, and custom-provider fallback. Crucially for Loom's BYOK requirement, the SDK documents runtime API-key overrides as **not persisted**, and accepts an in-memory credential store. [Pi SDK authentication](https://pi.dev/docs/latest/sdk)

Therefore, store a user's key in the device OS credential store, retrieve it only in the Electron main/agent-host process, initialize Pi with in-memory credentials or set a runtime API key, and do not send the key to NestJS/Postgres or write it to Pi's `auth.json`. OAuth/subscription login is a future alternative UI flow; v1 can support API keys without relying on Pi's interactive `/login` storage. Pi's provider documentation confirms that its normal `/login` path stores tokens in `~/.pi/agent/auth.json`. [Pi providers](https://pi.dev/docs/latest/providers)

## Safety constraint

Pi's `cwd` controls project resource discovery and tool construction, but Pi explicitly states that it has **no built-in sandbox**: built-in tools and extensions run with the invoking user's permissions. Project trust only controls loading project-local resources; it does not restrict tool access after startup. [Pi security](https://pi.dev/docs/latest/security)

Loom's promised boundary—agents can only access the selected Project/worktree and may not alter global machine state—cannot be enforced by Pi configuration alone. Enforce it outside Pi by running each host in an OS sandbox/container/VM with only the ticket worktree mounted, a dedicated local session directory, minimum required credentials, and a network policy. Pi's own security guide recommends this form of containment for untrusted or unattended work. [Pi security: unmonitored work](https://pi.dev/docs/latest/security)

The worktree should be prepared by Loom before agent startup, mounted as the host's writable workspace, and verified again from the session header's `cwd` at resume. Never give Pi a shell with host-home or parent-project access just because its logical `cwd` is a worktree.

## Version and compatibility constraints

- Pin the exact `@earendil-works/pi-coding-agent` version in Loom's lockfile. Do not float a caret/range across releases: the externally consumed SDK, CLI/RPC protocol, JSON event schema, and session schema are evolving surfaces.
- Record the package version with each session attempt. Before upgrading the pinned version, run a compatibility gate against copied v3 sessions: open/resume, subscribe/reconnect after runtime replacement, pause/abort, inspect transcript, and verify GUI event translation.
- Follow Pi's documented v3 session schema and APIs rather than private source shapes. Automatic migration of older transcript versions means a downgrade after opening a session may not be safe; retain a local backup/copy before the first upgrade resume. [Pi session versioning](https://pi.dev/docs/latest/session-format)
- If Loom uses RPC, version and contract-test its JSONL adapter: Pi documents both strict record framing and an event-stream contract in which partial snapshots are deliberately omitted. [Pi RPC framing](https://pi.dev/docs/latest/rpc) and [Pi JSON event stream](https://pi.dev/docs/latest/json)

## Recommended Loom v1 architecture

1. Electron main process owns a local `AgentHost` for each active ticket and never exposes filesystem/process/key access to the renderer.
2. `AgentHost` uses the pinned Pi SDK and `AgentSessionRuntime`, with `cwd` set to that ticket's dedicated worktree and Pi sessions in device-local app data. It streams normalized, redacted events to the renderer and persists status metadata to NestJS.
3. A sandbox/container/VM encloses `AgentHost` and mounts only the worktree/session data required. This is mandatory to make Loom's declared safety boundary real.
4. On pause, request `abort()`, persist the exact session pointer plus a terminal heartbeat/status, then dispose the host. On resume, recreate the sandbox/host for the same worktree, validate session-header cwd/ID, reopen the exact session, resubscribe, and prompt it to continue the ticket.
5. Treat a human-required request as an action: persist an action summary and move the ticket to waiting-for-human only after Pi has stopped/settled. Use a terminal settled boundary rather than raw text completion, because the agent may still have queued work, retries, or compaction.

## Sources

- [Pi SDK documentation](https://pi.dev/docs/latest/sdk)
- [Pi RPC mode documentation](https://pi.dev/docs/latest/rpc)
- [Pi sessions documentation](https://pi.dev/docs/latest/sessions)
- [Pi session file format](https://pi.dev/docs/latest/session-format)
- [Pi JSON event stream mode](https://pi.dev/docs/latest/json)
- [Pi provider/authentication documentation](https://pi.dev/docs/latest/providers)
- [Pi security documentation](https://pi.dev/docs/latest/security)
- [Official Pi session-manager source](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)
