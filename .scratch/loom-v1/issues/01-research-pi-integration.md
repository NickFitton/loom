# Research Pi integration for durable local agent sessions

Type: research
Status: resolved
Blocked by: none

## Question

What documented Pi APIs, CLI capabilities, session/transcript formats, authentication mechanisms, and lifecycle controls can Loom safely use from Electron to start, observe, interrupt, and resume local agents? What integration architecture and version constraints follow from the primary sources?

## Comments

- Research branch/context: `research/pi-integration`.

## Answer

Use a pinned Pi TypeScript SDK in an Electron-main-owned local agent host, with a ticket worktree as `cwd`, a Loom-controlled session directory, and persisted exact JSONL session pointers. Pause by aborting and settling the host; resume by recreating the host and reopening the exact session after validating its ID and `cwd`. The hosted control plane receives redacted state/event summaries, never transcripts or credentials. Pi provides no sandbox, so Loom needs OS/container/VM containment in addition to its worktree policy. Detailed, cited findings: [Pi integration research](../research/pi-integration.md).
