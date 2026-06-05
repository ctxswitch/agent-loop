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
- The session-to-run binding
- Phase transitions
- Phase packets
- Subagent launch and handoff
- Evidence validation
- Pi session lifecycle recovery

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
  "evidenceIds": [],
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
  "scenarios": [
    {
      "id": "",
      "description": "",
      "evidenceIds": [],
      "result": "pass"
    }
  ],
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
  "findings": [
    {
      "severity": "P2",
      "category": "",
      "location": "",
      "problem": "",
      "evidenceIds": [],
      "requiredFix": ""
    }
  ],
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

## Evidence Model

Evidence is stored as structured ledger entries. Phase artifacts should point to
evidence by id instead of embedding prose-only summaries.

Minimum evidence entry shape:

```json
{
  "id": "",
  "kind": "command",
  "phase": "verification",
  "createdAt": "",
  "cwd": "",
  "summary": "",
  "details": {}
}
```

Supported evidence kinds:

- `command`: command, cwd, startedAt, endedAt, exitCode, stdoutExcerpt,
  stderrExcerpt, fullOutputPath when output is truncated, and toolCallId when
  available.
- `diff`: baseRef, headRef, diffHash, changedFiles, and patchPath when captured.
- `file`: path, hash, relevantLines, and reason.
- `scenario`: scenarioId, description, expected result, actual result, and
  linked command or file evidence.
- `review_finding`: severity, location, problem, fix requirement, and supporting
  evidence ids.

Transition validation should reject a phase pass when required evidence is
missing. Examples:

- Implementation cannot advance without at least one `diff` evidence entry.
- Verification cannot pass without scenario entries and command or file evidence
  for each scenario.
- Review cannot pass or fail without an explicit finding list or explicit
  no-findings statement.
- Close cannot complete unless it references the final diff, verification
  evidence, review result, and final git status evidence.

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

The run ledger is the structured data model for a Pi-managed run. It should be
durable, inspectable, and reconstructable from append-only events.

Minimum event stream:

- `run_started`
- `phase_started`
- `evidence_recorded`
- `artifact_recorded`
- `transition_requested`
- `transition_accepted`
- `transition_rejected`
- `run_closed`
- `mirror_exported`

Runs are session-specific. In the Pi runtime, an active loop belongs to one Pi
session, not to the whole project or a global extension process. Starting,
resuming, stopping, or advancing a loop must use the current session identity to
select the run. A different Pi session in the same repository can have a
different active loop without overwriting or advancing the first session's run.

For Pi-managed runs, Pi's session JSONL is the authority for active phase state.
The extension should append compact custom entries for loop starts, phase
artifacts, evidence, transitions, and closeout decisions. The project-local
ledger is an inspectable export mirror, not the source that commands mutate
first.

The project mirror should still live in the repository so it is easy to inspect,
diff, or attach to a PR. It must be namespaced by session. A default shape such
as `.agent-loop/sessions/pi/<session-id>/run.json` is preferable to a single
`.agent-loop/run.json`. Shared project-level indexes or summaries may exist, but
they must not be the authority for phase state.

When Pi restores a session, the extension should rebuild in-memory loop state
from the session entries and then refresh the project mirror. If the session
entries and project mirror conflict, session entries win and the mirror is
rewritten with a conflict note.

Example:

```json
{
  "sessionId": "",
  "sessionFile": "",
  "runId": "",
  "goal": "",
  "phase": "implementation",
  "transitions": [],
  "evidence": [],
  "implementation": {
    "changedFiles": [],
    "evidenceIds": [],
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

The ledger mirror is not a compatibility surface for other hosts. It exists so
users can inspect, diff, and attach run evidence without reading Pi's session
files directly.

## Pi Runtime

Pi is the strongest runtime target because Pi extensions can:

- Register commands and tools.
- Subscribe to lifecycle, session, agent, model, and tool events.
- Persist state.
- Inject messages.
- Prompt through UI when needed.
- Load packages from npm, git, or local paths.

The extension should use the Pi session manager for:

- Session id lookup.
- Session file lookup when persisted.
- Custom entries for durable extension state.
- Session lifecycle events for shutdown, start, switch, and fork handling.
- Branch/tree navigation state recovery.

The Pi extension should expose a small surface:

- Start a loop from a user goal.
- Show loop status.
- Stop or reset a loop.
- Record evidence through tools.
- Launch phase subagents.
- Gate transitions through the ledger.

The Pi extension must bind those operations to the current Pi session. Commands
such as start, status, stop, reset, record, and next should resolve the active
run through the session id before reading or writing the ledger. The extension
may expose project-wide discovery of historical sessions, but project-wide
operations should not implicitly mutate another session's active loop.

Manual interaction should be minimal. The extension should not require the user
to manually advance through every phase.

### Pi Subagent Execution

The Pi runtime must not assume that "subagent" is a native extension primitive.
The extension should implement a `PhaseAgentRunner` adapter backed by child Pi
processes. The preferred backend is an installed Pi subagent package that runs
normal child `pi` invocations. If that package is not available, the adapter can
fall back to an explicit child-process wrapper with the same contract.

The runner contract must define:

- `cwd` for each phase process.
- Phase prompt and packet passed over stdin or an equivalent non-argv channel.
- Tool allowlist per phase.
- Model and thinking inheritance rules.
- Artifact schema expected from stdout or structured tool details.
- Timeout, abort, and cleanup behavior.
- Full-output capture path for truncated child output.
- A policy that blocks recursive delegation unless explicitly enabled with a
  depth cap.

Default phase tool policy:

- Implementation can inherit the parent write-capable tools.
- Verification can read files and run checks, but should not edit.
- Review is read-only by default.
- Close can read git state and prepare handoff text, but cannot push, publish,
  or create externally visible resources without explicit approval.
- Retro is read-only except for approved docs or issue creation workflows.

### Pi Session Lifecycle

Session-specific loops must follow Pi session lifecycle events.

- On session start or resume, rebuild loop state from custom session entries and
  refresh the project mirror.
- On session shutdown, flush any pending mirror write and record incomplete
  phase state when possible.
- On session switch, never carry in-memory state from the previous session into
  the new one.
- On fork or clone, create a new run id linked to the parent run id. Copy prior
  evidence as inherited context only when it is reachable from the selected
  branch point.
- On tree branch navigation, derive the active loop from the selected branch's
  custom entries. If a branch has no active loop entry, status should report no
  active loop rather than falling back to another branch.
- On compaction, preserve structured ledger entries outside model context and
  avoid using compacted prose as the source of truth.

### Pi Packaging And Trust

The Pi package must declare extension resources through the package manifest
instead of relying on incidental file discovery. The expected `package.json`
shape for `packages/pi-agent-loop` includes:

```json
{
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions/index.ts"],
    "prompts": ["./prompts"]
  },
  "peerDependencies": {
    "@earendil-works/pi-coding-agent": "*"
  }
}
```

Package policy:

- The runtime extension should be installed as trusted user or project package
  code before it can orchestrate a loop.
- Phase prompts bundled with the package are trusted with the package.
- Project-local prompt or agent overrides are disabled by default for automated
  phase execution unless the user explicitly trusts the repository.
- Project-local agents are repository-controlled prompts and can request tools
  from the parent allowlist, so the runner must confirm or block them in
  non-interactive runs.
- The package should ship only public runtime files. Tests, local `.pi/`
  settings, fixtures that are not needed at runtime, and generated coverage
  should not be published.
- Pre-release checks should include package tests, type checking, linting,
  security audit for production dependencies, and a package dry run.

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
        events.ts
        transitions.ts
        packets.ts
    pi-agent-loop/
      package.json
      extensions/
        index.ts
        phase-agent-runner.ts
      prompts/
        implementation.md
        verification.md
        review.md
        close.md
        retro.md
```

## Open Questions

- How much git metadata should be captured automatically?
- Should close create commits/PRs or only prepare evidence until approved?
- How should retro proposals become durable issues or docs changes?
