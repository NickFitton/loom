# Pi desktop integration contract

**Research date:** 2026-07-30  
**Scope:** Official Pi documentation and its linked official API/source documentation only.

## Recommendation

Use Pi's pinned Node SDK inside an Electron **main-process Wayfinder Runtime Supervisor**. Give each active Project one `AgentSessionRuntime` and keep the renderer on a narrow, validated IPC contract. Pi explicitly positions this SDK for embedding in custom desktop interfaces; it exposes the typed `AgentSession` lifecycle directly. Keep Pi's RPC mode as a later isolation option, not the primary v1 integration: it is a JSONL subprocess protocol for headless custom UIs, but a Node/TypeScript host gets type-safe state, programmatic extensions, and custom tools through the SDK. [Pi SDK](https://pi.dev/docs/latest/sdk) · [Pi RPC mode](https://pi.dev/docs/latest/rpc)

The product must define its own semantic run protocol. Pi exposes execution events, but it has no native `wayfind`, `frontier`, `subtask`, decision batch, or approval-gate concepts. The managed Wayfinder skill and extension should emit those concepts explicitly; generic Pi events are supporting operational evidence only.

## Proposed runtime boundary

```text
Electron renderer
  └─ validated IPC (commands + derived, privacy-safe events)
       └─ main-process Wayfinder Runtime Supervisor (one active run per Project)
            ├─ pinned Pi SDK + app-managed ResourceLoader
            ├─ Project AgentSessionRuntime / local JSONL session
            ├─ managed Wayfinder skill and extension
            └─ narrow backend-provider tools
                 └─ Wayfinder backend ── encrypted GitHub credential ── GitHub
```

The renderer must never receive a Pi object, provider credential, raw session file, or an unrestricted shell/network channel. The Supervisor translates Pi events into a versioned local event envelope such as `run.started`, `turn.started`, `activity.tool_started`, `step.updated`, `human_gate.requested`, `run.paused`, `run.settled`, and `run.failed`. Only the permitted structured run summary is sent to the backend.

## Supported Pi contract

### Lifecycle, stream, and status

`AgentSession.subscribe()` provides message deltas, tool-execution start/update/end, message and turn boundaries, agent start/end, queue changes, compaction, and retry events. The SDK documentation shows `tool_execution_*` events for live tool activity and `turn_end` with its assistant message and tool results. The extension lifecycle distinguishes `agent_end` from `agent_settled`: use `agent_settled` for a final Control Station status because retries, compaction, or queued continuations can follow `agent_end`. [SDK event API](https://pi.dev/docs/latest/sdk) · [extension lifecycle](https://pi.dev/docs/latest/extensions)

Map the stream as follows:

| Pi signal | Control Station meaning |
| --- | --- |
| `agent_start`, `turn_start` | Run/turn is working |
| `tool_execution_start/update/end` | Inspectable activity row; progress only, not a semantic subtask |
| `queue_update` | Pending steer/follow-up input |
| `compaction_*`, retry events | System state requiring an honest “recovering” indicator |
| `agent_settled` | Safe terminal/awaiting-next-input boundary |
| managed `wayfinder_progress` / `wayfinder_human_gate` tool result | Named step, subtask, blocker, or decision-batch state |

The final row is app-defined. Implement it as a typed custom tool/extension event and retain its structured result locally; Pi supports custom tools with a `details` payload and tool streaming, but does not create the semantic categories itself. [SDK custom tools](https://pi.dev/docs/latest/sdk) · [Pi extensions](https://pi.dev/docs/latest/extensions)

### Start, interrupt, steer, and continue

`session.prompt()` starts a run. While streaming, `session.steer()` queues an instruction after the current assistant turn's tool calls and before its next model call; `session.followUp()` waits until the agent is otherwise finished. `session.abort()` stops the current operation. This maps to Control Station controls of **respond now**, **add after this run**, and **stop**. Do not present steer as an immediate stop: its documented delivery point is after the current tool calls. [SDK `AgentSession` and prompt queueing](https://pi.dev/docs/latest/sdk)

For the decision-batch workspace, submit the entire resolved batch as a single structured prompt when the user selects “answer and continue.” A card-level challenge or clarification is a scoped steer/follow-up only when an agent run is active; otherwise it starts the next batch-resolution turn. The batch/card data remains Wayfinder-owned UI state and session metadata, rather than Pi TUI state.

### Persistence and resume

Pi sessions are append-only JSONL trees, retaining messages, model changes, labels, compactions, branch summaries, and extension entries. The SDK supports persistent session creation, continuing the most recent session, opening a specified session, and `AgentSessionRuntime` operations for new, switch, fork, and import flows. Session replacement changes the session instance, so the Supervisor must re-subscribe to events and rebind extensions after each replacement. [Pi sessions](https://pi.dev/docs/latest/sessions) · [session file format](https://pi.dev/docs/latest/session-format) · [SDK runtime contract](https://pi.dev/docs/latest/sdk)

Persist each local session under application-controlled storage and store a local `runId → sessionFile/sessionId` mapping. Keep raw JSONL and the full transcript local; sync only the agreed privacy-safe projection. The RPC protocol's `get_entries` cursor demonstrates that entry IDs are stable durable cursors, but v1 does not need RPC merely to resume an SDK-backed session. [RPC session protocol](https://pi.dev/docs/latest/rpc)

### App-managed, versioned Wayfinder runtime

Pin the Pi package version and bundle versioned Wayfinder resources (skill, extension, prompts, and tool schema). Record both versions on every local run. Construct the resource loader explicitly and point it only to the bundled resources plus the intended Project context. This is important because Pi's default resource discovery considers global and project extensions, skills, prompts, context files, settings, credentials, and sessions; loading ambient configuration would violate reproducibility and the provider boundary. Pi supports a custom `ResourceLoader`, additional extension paths/factories, and overrides for skills, context files, and prompts. [SDK resources and directories](https://pi.dev/docs/latest/sdk)

**Open implementation spike:** verify the exact `DefaultResourceLoader` configuration that prevents *all* ambient global/project discovery for the pinned Pi version. The documented overrides are additive examples, so use a dedicated loader or prove the resulting loaded-resource manifest in an automated test before claiming strict isolation.

### Backend-managed provider tools and approval gates

Use `defineTool({ ... })` supplied through `customTools`, or an app-managed extension using `pi.registerTool()`, for narrow capabilities such as:

- `tracker.read_map`
- `tracker.get_issue`
- `tracker.create_ticket`
- `tracker.update_own_ticket`
- `tracker.propose_scope_expanding_change`
- `wayfinder.report_run_summary`
- `wayfinder.record_step`

The tool executes against the Wayfinder backend using the desktop's own backend session; the backend uses the encrypted GitHub credential. The tool never receives or persists the GitHub credential. Pi supports typed custom tools, runtime tool enablement, and custom/extension tools alongside selected built-ins. When a `tools` allowlist is supplied, custom tool names must be included explicitly. [SDK tools](https://pi.dev/docs/latest/sdk) · [extension tool API](https://pi.dev/docs/latest/extensions)

Default to a strict built-in allowlist (or no `bash` tool) so that generic shell/network access cannot bypass those backend-managed tools. Pi extensions can inspect and block tool calls, but removal/allowlisting is the stronger baseline. [extension interception](https://pi.dev/docs/latest/extensions)

For an approval gate, have the managed extension/tool emit a typed pending action to the Supervisor and wait for the UI's signed response before calling the backend's mutating endpoint. The backend must independently enforce the policy and request identity; the Pi-side gate is a usability boundary, not authorization. Pi tool handlers are asynchronous and receive an abort signal; propagate it to cancellable backend requests. [extension tool definition and cancellation context](https://pi.dev/docs/latest/extensions)

## RPC fallback

If crash/process isolation later outweighs same-process SDK access, run `pi --mode rpc` in an app-owned child process. Commands and events are JSON objects on stdin/stdout using strict LF-delimited JSONL; do not use a generic Node `readline` parser because Pi documents that it can split valid JSON strings incorrectly. RPC supports prompt, steer, follow-up, abort, session switching, model state, and an observable queue. [Pi RPC protocol](https://pi.dev/docs/latest/rpc)

In RPC mode, extension UI requests can be bridged to the host, but the batch-grilling cards, threaded discussions, and approval presentation should remain native Electron UI. Do not base the non-conversational UX on Pi's TUI component APIs.

## Risks and acceptance checks

- **Runtime drift:** lock the Pi and Wayfinder runtime versions; record and display both. Test resource manifests contain no unapproved user/project extension or skill.
- **Event/session replacement:** automated test resumes, switches, and forks a session and confirms the Supervisor re-subscribes and rebinds before accepting another command.
- **Semantic visibility:** automated test a decision batch and confirm the UI gets named steps and individual cards without parsing assistant prose.
- **Credential escape:** test that the tool list excludes unrestricted `bash`/network paths and that all tracker mutations reach the backend without a GitHub token in local logs, Pi session data, or renderer IPC.
- **Cancellation and connectivity:** test abort while a managed tool call is pending; test loss of backend/model/tracker connectivity pauses the run and creates no deferred offline mutation.
- **Package/extension trust:** Pi extensions execute code. Treat the managed runtime bundle as trusted application code, do not auto-load ambient extensions, and surface resource-load diagnostics to local diagnostics only. [Pi extensions](https://pi.dev/docs/latest/extensions)

## Primary sources

- [Pi SDK](https://pi.dev/docs/latest/sdk)
- [Pi RPC mode](https://pi.dev/docs/latest/rpc)
- [Pi extensions](https://pi.dev/docs/latest/extensions)
- [Pi sessions](https://pi.dev/docs/latest/sessions)
- [Pi session file format](https://pi.dev/docs/latest/session-format)
