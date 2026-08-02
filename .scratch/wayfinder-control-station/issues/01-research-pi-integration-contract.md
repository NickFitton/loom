Label: wayfinder:research
Type: research
Status: resolved

# Establish Pi's desktop integration contract

## Question

What supported Pi SDK or RPC lifecycle, event stream, control operations, resource-loading mechanism, and custom-tool interfaces should an Electron application use to host a versioned Wayfinder runtime? Establish how to stream meaningful run steps, interrupt or steer a run, persist/resume sessions, and expose backend-managed provider tools. Capture primary-source evidence, constraints, and open risks that affect the runtime design decision.

## Answer

Use a version-pinned Pi SDK in an Electron main-process Runtime Supervisor, with one `AgentSessionRuntime` per active Project and a renderer-facing, privacy-safe event adapter. Pi's typed session API supports live lifecycle/tool events, `abort`, `steer`, `followUp`, and local JSONL session resume; use RPC only if later process isolation is worth its JSONL protocol boundary. Make steps, subtasks, grills, and gates explicit Wayfinder custom-tool/extension events, because Pi supplies observability rather than those domain concepts. Construct and test a managed loader so ambient Pi resources cannot enter the runtime; expose GitHub only through narrow backend-calling custom tools and omit generic shell/network tools by default. Full cited findings: [Pi desktop integration contract](../research-pi-integration-contract.md).
