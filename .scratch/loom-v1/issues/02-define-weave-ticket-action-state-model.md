# Define the Weave, Ticket, Action, and agent-session state model

Type: grilling
Status: resolved
Blocked by: none

## Question

What are the canonical states, transitions, ownership, and audit records for a Weave, its Tickets, human Actions, and device-local agent sessions? In particular, how do waiting-for-human, paused, resumed, cancelled, failed, and completed states interact without losing the agent's context or presenting ambiguous next actions?

## Answer

Loom has four related but distinct state machines. The control plane owns their durable lifecycle records and append-only history; the execution plane owns actual local processes, worktrees, and Pi context, and reports confirmations back to the control plane.

### Ownership and history

- A Weave belongs to one Project. A Ticket belongs to one Weave. An Action belongs to the Ticket that requested it.
- An Agent session belongs to one Ticket and the Device that ran it. Its exact Pi context and worktree stay on that Device; the control plane records only the lifecycle and local-context pointer needed to reconnect it.
- A Ticket has at most one active Agent session. Other Tickets in the same Weave may run sessions concurrently. Finished attempts remain immutable history.
- Every transition appends an audit event recording the subject and related Weave/Ticket/Action/session IDs, previous and new state, time, actor (person, Agent, or system), Device when applicable, reason or error, and a shared correlation ID for cascades. A cancellation request and the Device acknowledgement that work stopped are separate events.

### Weave states

| State | Meaning and transitions |
| --- | --- |
| `active` | Normal state. Its Tickets may independently run, pause, wait, fail, or complete. |
| `pausing` | A person requested a Weave-level pause; Loom is awaiting Device confirmation for the sessions it paused. |
| `paused` | Every session affected by that pause has stopped. Resuming returns only those sessions to running; individually paused, failed, or waiting Tickets remain unchanged. |
| `cancelling` | A person requested cancellation; unfinished Tickets, pending/running Actions, and active sessions are being stopped. |
| `cancelled` | Every affected Device has confirmed the cancellation. Completed Ticket history remains intact. |
| `completed` | Set automatically, not manually, when every Ticket currently in the Weave is `completed`. |

A cancelled Ticket prevents automatic completion. To continue cancelled work, Loom creates a linked replacement Ticket and the person explicitly removes or re-scopes the cancelled Ticket; Loom never silently treats it as complete.

### Ticket states

| State | Meaning and primary next action |
| --- | --- |
| `ready` | No session exists yet. Start an Agent session. |
| `active` | Its one Agent session is running. Pause, cancel, or let the Agent continue. |
| `paused` | A person paused the session. Resume the same session from retained context. |
| `waiting_for_human` | A named human decision is required: approve/decline an Action, review a completed session, or resolve a user-fixable problem. The paused session and its context are retained. |
| `waiting_for_action` | The person approved the Action and Loom is awaiting its result; the session remains paused. |
| `failed` | The active session could not be recovered. A person may start a new session for this Ticket. |
| `cancelling` | Ticket cancellation has been requested and Loom is awaiting acknowledgement that its Action/session stopped. |
| `cancelled` | Terminal, intentional abandonment of this Ticket. Continuing the work creates a linked replacement Ticket. |
| `completed` | A person accepted the result of a completed session. |

`waiting_for_human` always carries the specific waiting reason and linked Action or review result, so it has one clear next decision rather than an ambiguous generic pause.

Key transitions are: `ready → active` when a session starts; `active ↔ paused` when a person pauses/resumes; `active → waiting_for_human` when an Action is requested or a session finishes for review; `waiting_for_human → waiting_for_action` on Action approval; `waiting_for_action → active` only after Action completion; and `failed → active` only when a person starts a replacement session. Any unfinished Ticket may enter `cancelling → cancelled` by an explicit cancellation.

### Agent-session states

An Agent session is device-local and is `running`, `paused`, `recovering`, `cancelling`, or terminally `completed`, `failed`, or `cancelled`.

- A network loss or Device shutdown enters `recovering`; Loom makes a fair attempt to reopen the pinned local Pi session from its durable context.
- A recoverable problem that needs the person keeps the session `paused` and moves the Ticket to `waiting_for_human`.
- Only an unrecoverable session reaches terminal `failed`, which moves its Ticket to `failed` until the person starts a new session.
- A completed session is not itself Ticket completion: the Ticket waits for human review and acceptance.

### Action states

An Action follows this tree:

```text
requested → running → completed
          ↘ declined
          ↘ cancelled
running   ↘ failed
          ↘ cancelling → cancelled
```

`requested` pauses the Agent session and puts its Ticket in `waiting_for_human`. Approval begins `running` and changes the Ticket to `waiting_for_action`. Only `completed` resumes the same paused session automatically. `declined`, `failed`, or `cancelled` are immutable audit outcomes; the Ticket returns to `waiting_for_human` with its retained context and a clear choice to create a new Action, provide revised instructions, start a new session when appropriate, or cancel the Ticket. Cancelling an Action alone never discards the Ticket or its paused session.
