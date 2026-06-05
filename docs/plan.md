# Agent Loop Plan

## Problem

Coding agents often blur implementation, verification, review, closeout, and
retro into one continuous context. That creates predictable failure modes:

- The agent verifies its own assumptions.
- Review is anchored by implementation reasoning.
- Evidence is scattered through chat and tool output.
- Closeout happens without a structured check of what was proven.
- Retro lessons do not become durable improvements.

The proposed fix is a workflow runtime that operates as a state machine and
delegates each phase to a fresh subagent.

## Core Idea

The main agent or extension is an orchestrator. It owns:

- The state machine
- The run ledger
- Phase transitions
- Phase packets
- Subagent launch and handoff
- Evidence validation

Each phase is a fresh subagent with a narrow context packet and a required
artifact schema.

```text
orchestrator
  owns state machine
  owns run ledger
  prepares phase packets
  launches phase subagents
  validates returned artifacts
  decides next transition
```

Subagents exchange artifacts, not chat history.

## Phases

### 1. Implementation

Role: builder.

Objective: produce the smallest working change that satisfies the goal.

Inputs:

- User goal
- Repo instructions
- Relevant files and context
- Constraints and non-goals
- Prior defects or review findings when looping back

Allowed behavior:

- Edit files
- Run focused tests and checks
- Update implementation notes
- Ask for user input only when blocked by an actual product or safety decision

Required artifact:

```json
{
  "phase": "implementation",
  "changedFiles": [],
  "summary": "",
  "tests": [],
  "knownRisks": [],
  "questions": []
}
```

Exit rule: implementation can advance only when there is a concrete diff and
enough local evidence to justify verification.

### 2. Verification

Role: skeptical operator.

Objective: prove behavior through scenarios and checks using fresh context.

Inputs:

- User goal
- Diff
- Implementation artifact
- Known risks

Excluded by default:

- Full implementation transcript
- Implementation reasoning that is not part of the artifact

Allowed behavior:

- Read changed files
- Run scenario tests
- Run focused checks
- Report defects

Required artifact:

```json
{
  "phase": "verification",
  "scenarios": [],
  "commands": [],
  "failures": [],
  "confidence": "",
  "nextAction": "pass"
}
```

Exit rule:

- `pass` advances to review.
- `fix_required` routes back to implementation with a defect packet.
- `blocked` asks the orchestrator or user for scope clarification.

### 3. Review

Role: code reviewer.

Objective: inspect the diff for correctness, maintainability, style,
architecture, security, and repo-principle fit.

Inputs:

- User goal
- Diff
- Verification artifact
- Repo guidance

Allowed behavior:

- Read changed files and nearby code
- Inspect tests
- Produce findings
- Recommend fixes

Required artifact:

```json
{
  "phase": "review",
  "findings": [],
  "waivedFindings": [],
  "summary": "",
  "nextAction": "pass"
}
```

Exit rule:

- `pass` advances to close.
- `fix_required` routes back to implementation with a finding packet.

### 4. Close

Role: release clerk.

Objective: reconcile evidence, check final state, and prepare handoff.

Inputs:

- User goal
- Final diff
- Implementation artifact
- Verification artifact
- Review artifact
- Git status

Allowed behavior:

- Check status and final diff
- Prepare commit or PR materials
- Confirm evidence completeness
- Ask for approval before irreversible or externally visible actions when
  required

Required artifact:

```json
{
  "phase": "close",
  "evidenceComplete": false,
  "status": "",
  "commit": null,
  "pr": null,
  "remainingRisks": []
}
```

Exit rule:

- Complete close advances to retro.
- Missing evidence routes to the relevant previous phase.

### 5. Retro

Role: process analyst.

Objective: analyze the run and propose improvements that make the next run more
efficient or accurate.

Inputs:

- All phase artifacts
- Transition history
- Timing information
- User interventions
- Failures and rework loops

Required artifact:

```json
{
  "phase": "retro",
  "friction": [],
  "toolingGaps": [],
  "promptFailures": [],
  "proposedFixes": [],
  "followUpIssues": []
}
```

Exit rule: retro completes the run after recording durable improvement
proposals.

## State Machine

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

## Run Ledger

The run ledger is the source of truth. It should be structured and durable.

Example:

```json
{
  "goal": "",
  "phase": "implementation",
  "transitions": [],
  "implementation": {
    "changedFiles": [],
    "tests": [],
    "knownRisks": []
  },
  "verification": {
    "scenarios": [],
    "failures": []
  },
  "review": {
    "findings": [],
    "waivedFindings": []
  },
  "close": {
    "statusClean": false,
    "commit": null,
    "pr": null
  },
  "retro": {
    "friction": [],
    "proposedFixes": []
  }
}
```

The ledger should be inspectable and portable. A project-local path such as
`.agent-loop/run.json` is a reasonable default.

## Pi Runtime

Pi is the strongest runtime target because Pi extensions can:

- Register commands and tools.
- Subscribe to lifecycle, session, agent, model, and tool events.
- Persist state.
- Inject messages.
- Prompt through UI when needed.
- Load packages from npm, git, or local paths.

The Pi extension should expose a small surface:

- Start a loop from a user goal.
- Show loop status.
- Stop or reset a loop.
- Record evidence through tools.
- Launch phase subagents.
- Gate transitions through the ledger.

Manual interaction should be minimal. The extension should not require the user
to manually advance through every phase.

## Codex Crossover

Codex should share the protocol, not the Pi runtime.

Portable pieces:

- Phase definitions
- Evidence schemas
- Transition rules
- Prompt packets
- Review principles

Codex adapter options:

- A Codex skill generated from the same phase definitions.
- A CLI tool, `agent-loop`, that stores the same ledger and exposes commands
  like `start`, `status`, `record`, `next`, and `close`.
- Repo-local guidance in `AGENTS.md` that tells Codex to use the CLI-backed
  protocol when available.

Codex will not get Pi's in-process subagent orchestration from a plain skill.
If real enforcement is needed, Codex needs an external adapter such as the CLI.

## Package Structure

Target structure:

```text
agent-loop/
  README.md
  AGENTS.md
  docs/
    plan.md
  packages/
    agent-loop-core/
      package.json
      src/
        schema.ts
        ledger.ts
        transitions.ts
        packets.ts
    pi-agent-loop/
      package.json
      extensions/
        index.ts
      prompts/
        implementation.md
        verification.md
        review.md
        close.md
        retro.md
    codex-agent-loop/
      package.json
      skills/
        agent-loop/
          SKILL.md
      bin/
        agent-loop
```

## MVP

1. Define the ledger schema and phase artifact schemas.
2. Implement transition validation in `agent-loop-core`.
3. Build sample phase packets from fixtures.
4. Build a Pi extension with one command to start a run.
5. Launch implementation as the first fresh phase subagent.
6. Record implementation artifact and advance to verification.
7. Add verification and review loops.
8. Add close and retro artifacts.
9. Generate or maintain a Codex skill from the same phase definitions.
10. Add a CLI adapter only after the core protocol stabilizes.

## Open Questions

- Which Pi subagent mechanism should be used for phase isolation?
- Should the ledger be project-local by default, session-local, or both?
- How much git metadata should be captured automatically?
- Should close create commits/PRs or only prepare evidence until approved?
- How should retro proposals become durable issues or docs changes?
- What is the minimal Codex adapter that gives useful enforcement without
  recreating Pi's extension runtime?
