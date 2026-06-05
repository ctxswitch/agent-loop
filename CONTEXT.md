# Agent Loop

This context defines the domain language for the Pi session-scoped agent-loop
runtime.

## Language

**Loop Run**: A phase-isolated workflow instance bound to one active branch path
inside a Pi session.
_Avoid_: Project run, session-wide run

**Session Branch Path**: The ordered path of Pi session entries from the session
root to the current active leaf.
_Avoid_: Session id, session file

**Ledger Mirror**: A project-local export of loop evidence reconstructed from Pi
session entries.
_Avoid_: Source of truth, canonical ledger

**Loop Event**: An append-only Pi custom entry that records a state change or
evidence addition for a loop run.
_Avoid_: Snapshot, mutable state

**Phase Attempt**: One execution of a workflow phase, including failed or
superseded attempts.
_Avoid_: Current phase artifact, latest result

**Loopback Packet**: A structured artifact that carries defects or findings from
one phase attempt into a later implementation attempt.
_Avoid_: Failure summary, retry note

**Accepted Transition**: The event that advances a loop run from one phase state
to the next.
_Avoid_: Phase start, artifact write

**Lifecycle Command**: A user-facing command that starts, stops, resumes, or
reports a loop run without choosing a phase transition.
_Avoid_: Transition command, manual phase control

**Continue Command**: A lifecycle command that resumes the orchestrator from the
last reconstructed loop state.
_Avoid_: Advance command, transition command

**Paused Loop**: A loop run that is intentionally not executing but remains
resumable with continue.
_Avoid_: Cancelled run, terminal stop

**Closed Loop**: A loop run that has completed its workflow and no longer
accepts continue on the same branch path.
_Avoid_: Paused loop, stopped loop

**Retro Attempt**: The required final phase attempt that records process
friction and workflow improvements before a loop can close.
_Avoid_: Optional aftercare, post-close note

**Retro Recommendation**: A non-applied proposal to improve agent memory or
harness behavior.
_Avoid_: Repo edit, retro edit, automatic fix

**Human Intervention Reason**: A structured explanation of why the orchestrator
cannot continue without a human decision or external action.
_Avoid_: Blocked, unsure, needs help

## Relationships

- A **Loop Run** belongs to exactly one **Session Branch Path**.
- A Pi session file can contain multiple **Session Branch Paths**.
- A **Loop Run** is reconstructed by replaying **Loop Events** on its
  **Session Branch Path**.
- A **Loop Run** can have multiple **Phase Attempts** for the same phase when
  verification or review routes work back to implementation.
- A **Loopback Packet** links a failed **Phase Attempt** to the implementation
  **Phase Attempt** that addresses it.
- An **Accepted Transition** is the only event that advances a **Loop Run**.
- A started **Phase Attempt** without an **Accepted Transition** remains
  historical context, not current phase state.
- A **Lifecycle Command** never chooses the next phase; phase transitions are
  orchestrator decisions.
- A **Continue Command** picks up from the last reconstructed state after user
  feedback or session interruption.
- A stopped loop is a **Paused Loop**, not a cancelled run.
- One **Session Branch Path** can have at most one non-closed **Loop Run**.
- A **Loop Run** becomes a **Closed Loop** only after an accepted
  **Retro Attempt**.
- A **Retro Attempt** records **Retro Recommendations** for memory or harness
  behavior and does not modify the repository.
- A **Paused Loop** caused by user feedback must include a **Human Intervention
  Reason**.
- A **Ledger Mirror** is derived from Pi session entries for one **Loop Run**.
- Sibling **Session Branch Paths** must not share active **Loop Run** state.

## Example Dialogue

> **Dev:** "Can I find the active loop from the Pi session id?"
> **Domain expert:** "No. Use the current session branch path; sibling branches in
> the same session file can have different loop state."
> **Dev:** "Can we store the current phase as one mutable record?"
> **Domain expert:** "No. Store append-only loop events and derive the current
> phase by replaying the active branch path."
> **Dev:** "When review fails and we go back to implementation, do we replace the
> old review artifact?"
> **Domain expert:** "No. Keep every phase attempt so implementation can receive
> the defect context and retro can explain the loop history."
> **Dev:** "How does the next implementation know why it is running again?"
> **Domain expert:** "It references the loopback packet produced by the failed
> verification or review attempt."
> **Dev:** "If a phase starts but the child agent times out, did the loop move?"
> **Domain expert:** "No. The loop only moves when the orchestrator records an
> accepted transition."
> **Dev:** "Can the user manually advance from verification to review?"
> **Domain expert:** "No. Users can start, stop, continue, and check status; the
> orchestrator owns transitions."
> **Dev:** "After Pi quits or the loop waits for feedback, should the user choose
> the next phase?"
> **Domain expert:** "No. They run continue, and the orchestrator resumes from
> the last reconstructed state."
> **Dev:** "Does stop cancel the loop?"
> **Domain expert:** "No. Stop pauses the loop so continue can resume it later."
> **Dev:** "Can start create another loop on a branch that already has a paused
> loop?"
> **Domain expert:** "No. Continue the existing loop, or create a new branch or
> session first."
> **Dev:** "Can close finish the loop without retro?"
> **Domain expert:** "No. Retro is required before the loop is closed."
> **Dev:** "Should retro automatically edit prompts, docs, or code?"
> **Domain expert:** "No. Retro does not modify the repository; it suggests
> memory or harness behavior improvements."
> **Dev:** "Can the loop pause with just 'blocked'?"
> **Domain expert:** "No. The pause must record why human intervention is
> required."

## Flagged Ambiguities

- "Session-specific" means scoped to the active **Session Branch Path**, not the
  whole Pi session file.
