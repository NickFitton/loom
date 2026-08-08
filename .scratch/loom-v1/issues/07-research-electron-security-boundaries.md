# Research Electron security boundaries for local agent execution

Type: research
Status: resolved
Blocked by: none

## Question

What current Electron security guidance and supported APIs should shape Loom's main/renderer separation, preload bridge, secure credential storage, filesystem/process access, CSP, and update posture when it launches Pi and Git for a user-selected Project?

## Comments

- Research branch/context: `research/electron-security-boundaries`.

## Answer

Treat the renderer as untrusted and sandbox it; Electron main exclusively owns project-root validation, credential encryption, Pi/Git lifecycle, and a small typed preload capability bridge. Enforce fixed executables, argument arrays, controlled environment, worktree `cwd`, and path/sender validation in main; Electron sandboxing does not confine Pi or Git. Detailed, cited findings: [Electron security research](../research/electron-security-boundaries.md).
