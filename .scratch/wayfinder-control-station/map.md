Label: wayfinder:map

# Wayfinder Control Station

## Destination

Produce an implementation-ready product and technical specification for Wayfinder: a local-first Electron desktop application that uses Pi to manage Wayfinds across Projects, with GitHub as the first tracker and code-host integration.

## Notes

- This is a planning map, not implementation work. Consult `CONTEXT.md` for the active domain language.
- The primary UI is a non-conversational **Control Station**: it surfaces the portfolio of Projects, active Wayfinds, runs, and actionable human gates, with project-level drill-down.
- A Project is one local working copy and has one issue-tracker connection and one code-host connection. Separate clones are separate Projects, even with the same remote.
- Pi is embedded through an app-managed, versioned Wayfinder runtime. It exposes app-managed provider tools rather than ambient machine configuration.
- GitHub is the first provider for both issues and code hosting, but the model must allow future provider adapters.
- Issue trackers are authoritative for issues and maps; Wayfinder owns Project correlation, local run supervision, and the backend projection of privacy-safe run summaries.
- The backend is personal-account-first. It stores encrypted provider credentials once for multi-device use. Pi reaches providers through backend-managed tools; raw transcripts, source content, local paths, prompts, model responses, and tool output are not synced.
- A run needs connectivity and pauses safely if it is lost. One run may be active per Project, with concurrent runs allowed across Projects.
- Routine updates to a run's own map/tickets are automatic. Destructive or scope-expanding actions require an explicit approval gate.
- Grills use reviewable decision batches. Cards can be accepted, challenged, or clarified; their persistent discussion remains attached to the card and the resolved batch is submitted before the agent continues.

## Decisions so far

<!-- Resolution pointers are appended here when a ticket is closed. -->

- [Establish Pi's desktop integration contract](issues/01-research-pi-integration-contract.md) — Use a version-pinned SDK Runtime Supervisor with an app-owned semantic event/tool contract; keep Pi sessions local and provider credentials behind backend tools.

## Not yet specified

- The exact information architecture and visual grammar of the Control Station and decision-batch workspace.
- Account authentication, encryption boundaries, credential-vault operations, and recovery.
- Local Project discovery, repository/worktree validation, setup, and lifecycle semantics.
- The detailed data model and reconciliation behavior between local runs, the backend projection, and external tracker state.
- The GitHub capability scope and the provider-adapter extension contract.
- Desktop platform, installer, auto-update, and diagnostics expectations.

## Out of scope

- Shared team workspaces and multi-user collaboration in the first release.
- Syncing source code, raw transcripts, prompts, model responses, tool output, or local filesystem paths to the backend.
- Offline Wayfinder runs.
