# agent-loop

`agent-loop` is a workflow runtime concept for coding agents. The goal is to
make an agent work through a disciplined development loop with real phase
isolation, structured evidence, and explicit transitions.

The loop is:

1. Implementation
2. Verification
3. Review
4. Close
5. Retro

Each phase should run as a fresh subagent. The orchestrator owns the state
machine, prepares a narrow context packet for the phase, launches the subagent,
records the returned artifact, and decides the next transition.

For the Pi runtime, an active loop is session-specific. A Pi session owns its
current run, and loop commands must resolve state through that session rather
than through a single project-wide active run. Durable run ledgers can still live
inside the project, but Pi-managed ledgers should be mirrors of Pi session state
and namespaced by session.

The core design principle is artifact exchange instead of chat-history exchange.
Fresh subagents should receive the goal, relevant repo context, diffs, evidence,
and prior phase outputs. They should not inherit the full implementation
conversation unless the orchestrator intentionally includes it.

## Intended Shape

- `agent-loop-core`: shared schemas, ledger, transition engine, and phase packet
  builder.
- `pi-agent-loop`: Pi extension that enforces the protocol, launches phase
  agents through child Pi processes, injects phase prompts, and updates
  UI/status.
- `codex-agent-loop`: Codex skill plus optional CLI adapter that uses the same
  protocol and ledger.

Pi is the primary runtime target because Pi extensions can register commands and
tools, subscribe to events, persist session state, inject messages, prompt
through UI, and run extension code that delegates to child Pi processes. Codex
can share the workflow spec and use a CLI-mediated state machine, but it does
not get the same host session lifecycle from a plain skill.

## Phase Overview

### Implementation

Purpose: produce the smallest working change.

Inputs:

- User goal
- Repo instructions
- Relevant context
- Constraints
- Prior verification or review failures, if any

Outputs:

- Changed files
- Diff summary
- Tests or checks run
- Implementation notes
- Known risks or unresolved questions

### Verification

Purpose: prove the change works from fresh eyes.

Inputs:

- User goal
- Diff
- Implementation evidence
- Known risks

Outputs:

- Scenario matrix
- Commands or checks run
- Pass/fail evidence
- Defects, if any
- Confidence note

### Review

Purpose: inspect correctness, maintainability, style, and principle fit.

Inputs:

- User goal
- Diff
- Verification evidence
- Repo principles

Outputs:

- Findings ordered by severity
- Required fixes
- Waived findings, if any
- Explicit no-findings statement when clean

### Close

Purpose: reconcile evidence and prepare handoff.

Inputs:

- Goal
- Final diff
- Test and scenario evidence
- Review result
- Git status

Outputs:

- Evidence packet
- Final status
- Commit or PR proposal
- Remaining risks

### Retro

Purpose: improve the next run.

Inputs:

- All phase artifacts
- Timings
- Failures and regressions
- User interventions

Outputs:

- Process friction
- Prompt or tool failures
- Proposed workflow fixes
- Candidate docs or automation changes

## Transition Rules

```text
implementation -> verification
verification pass -> review
verification fail -> implementation with defect packet
review pass -> close
review fail -> implementation with finding packet
close incomplete -> relevant previous phase
close complete -> retro
retro complete -> done
```

## Current Status

This repository is a design scaffold. The next step is to define the protocol
schema, evidence schema, session lifecycle behavior, and minimal phase packet
format before implementing the Pi extension.
