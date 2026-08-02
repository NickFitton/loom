# Ubiquitous Language

## Terms

- **Wayfinder**: A local-first desktop application for supervising planning runs, their decisions, and their project context; it is not the coding agent itself.
- **Wayfind**: A long-running planning effort represented as a map of decisions, frontier tickets, and resolved outcomes.
- **Project**: A locally configured codebase (one working copy) and its associated remote context. A Project has exactly one issue-tracker connection and one code-host connection; every Wayfind belongs to one Project. Separate clones are separate Projects even when they share a remote repository.
- **Wayfinder runtime**: The app-managed, versioned Pi configuration used for a Project’s Wayfinds, including the Wayfinder skill and integrations. It is distinct from a machine’s ambient Pi configuration.
- **Coordination record**: The backend-synced metadata used to manage a Project or Wayfind across devices. It excludes source code, local paths, and full agent transcripts by default.
- **Credential vault**: The backend-managed encrypted store for provider credentials, enabling an Owner to authorize a provider once and use that connection from their other devices.
- **App-managed provider tool**: A Pi tool provided by the app which reaches a provider through the backend and credential vault. It never receives a long-lived provider credential on the desktop.
- **Run status**: The lifecycle state of a local Pi run (for example starting, analysing, executing a tool, waiting for a decision batch, paused, completed, failed, or cancelled). It is distinct from issue state. A Wayfinder run requires connectivity; it pauses safely when connectivity is lost.
- **Automation policy**: The visible, Project-scoped rules governing a Wayfinder run’s external actions. The default permits creation and updates to its own map and tickets, while destructive or scope-expanding actions require explicit Owner approval.
- **Issue-tracker provider**: The provider through which a Project owns and manages its Wayfinder map and decision tickets. GitHub Issues is the first supported provider.
- **Code-host provider**: The provider through which a Project’s repository context is identified. GitHub is the first supported provider.
- **Owner**: The individual account that owns the coordination records for its Projects and Wayfinds. Shared workspace membership is not part of the first release.
- **Control Station**: The app’s primary portfolio-level workspace. It shows active Projects, Wayfinds, run state, and actionable items, with drill-down navigation to project-specific detail rather than a chat-first interface.
- **Decision batch**: A reviewable set of independently addressable questions produced during a grill. Each card has its own context, recommendation, and Accept / Challenge / Clarify controls. The Owner reads the whole set and submits its resolved cards together before the agent continues.
- **Decision-card discussion**: The persistent, card-scoped exchange created by a Challenge or Clarify action. It retains the agent’s follow-ups, evidence, and revised recommendation with the decision it concerns.
