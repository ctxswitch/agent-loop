# Agent Guidance

This repository designs and implements `agent-loop`, a protocol and runtime for
phase-isolated coding-agent workflows.

## Purpose

- Build a state-machine workflow for coding agents.
- Treat each workflow phase as a fresh subagent, not a role-played mode inside
  one long conversation.
- Exchange structured artifacts between phases instead of raw chat history.
- Keep the core workflow protocol portable across Pi and Codex.
- Use Pi extensions for strong runtime orchestration.
- Use Codex skills and an optional CLI adapter for cross-tool compatibility.

## Working Principles

- The orchestrator owns phase state. Worker agents do not decide what phase they
  are in.
- Phase subagents receive narrow context packets and return structured
  artifacts.
- For Pi, phase subagents are child Pi processes through an explicit runner or
  subagent package, not an assumed built-in extension primitive.
- Evidence is first-class. Do not rely on prose summaries when a structured
  ledger entry can capture the same fact.
- Verification and review must be genuinely fresh. Do not pass implementation
  chain-of-thought or full implementation transcript by default.
- Manual interaction should be reserved for real decisions, destructive actions,
  blocked scope, or final approval.
- Prefer a small protocol that can be enforced over a broad checklist that can
  be ignored.

## Expected Structure

- `README.md`: short explanation and current status.
- `docs/plan.md`: design plan and implementation roadmap.
- `packages/agent-loop-core/`: shared schema, ledger, transition engine, and
  packet builder.
- `packages/pi-agent-loop/`: Pi extension runtime.
- `packages/codex-agent-loop/`: Codex skill and optional CLI adapter.

## Design Constraints

- Keep Pi-specific APIs out of the core protocol package.
- Keep Codex-specific skill wording generated from or aligned with the same
  phase definitions used by Pi.
- Store run state in a durable ledger. For Pi-managed runs, Pi custom session
  entries are authoritative; project-local ledgers are inspectable mirrors
  namespaced by session, for example
  `.agent-loop/sessions/<session-id>/run.json`.
- Make phase artifacts easy to inspect, diff, and attach to a PR.
- Avoid hiding important state in extension memory only.
- Treat project-local Pi agents, prompts, and settings as repository-controlled
  code. Automated phase execution should use trusted packaged prompts unless the
  user explicitly approves project-local overrides.
- Commit messages should use the user's normal `<action>: <message>` style, for
  example `docs: clarify Pi session lifecycle`.

## Validation

- Validate JSON schemas and sample ledgers.
- Add fixtures for phase transitions.
- Validate evidence requirements for each transition.
- Test transition behavior before adding richer Pi UI.
- For Pi extension work, test session start, resume, switch, fork, and branch
  recovery behavior.
- For Pi extension work, prefer small local package tests before installing into
  a global Pi configuration.
