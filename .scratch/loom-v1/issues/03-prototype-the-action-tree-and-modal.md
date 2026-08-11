# Prototype the action tree and action modal

Type: prototype
Status: resolved
Blocked by: 02

## Question

What homepage tree and action-modal interaction make the user's required next steps (Project → Weave → Ticket → Action) legible while exposing agent progress, pauses, and blocked work without turning the UI into a hidden chat transcript?

## Answer

**Navigator + Detail** (Variant A): collapsible source list on the left, detail panel on the right.

- **Hierarchy**: All projects → Project → Weave → Ticket — each level is a clickable node in the tree.
- **Roll-up**: clicking a Weave shows all its tickets' pending action cards + a compact status list; clicking a Project rolls up further; "All projects" at the root shows everything across all projects. Action-count badges on tree nodes let you see blocked work without opening anything.
- **Inline action cards**: the approve/decline card lives inside the detail panel for the selected ticket (or aggregated in the roll-up view). No modal interruption — the action stays in context.
- **Agent progress**: shown as a compact, timestamped activity log (tool-call events: "Read 12 files", "Wrote user.controller.ts", "Linter: 0 errors", "Wiring DI module…") — not a chat transcript. The current step is visually distinguished; completed steps are dimmed.
- **Status dots**: colored dots on every tree node (blue pulsing = active, orange = waiting/action, yellow = paused, green = done, dim = ready) give full-tree status at a glance without text labels.

## Comments

- Initial prototype (three variants, Variant A selected): [`prototypes/action-tree-modal-prototype.html`](../prototypes/action-tree-modal-prototype.html)
- Iterated prototype with roll-up panels and native macOS styling: branch `prototype/03-action-tree-modal`, file `.scratch/loom-v1/prototype/index.html`
