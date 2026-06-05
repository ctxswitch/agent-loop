# Agent Guidance

This repository designs and implements `agent-loop`, a protocol and runtime for
phase-isolated coding-agent workflows.

## Purpose

- Build a state-machine workflow for coding agents.
- Treat each workflow phase as a fresh subagent, not a role-played mode inside
  one long conversation.
- Exchange structured artifacts between phases instead of raw chat history.
- Use Pi extensions for strong runtime orchestration.
- Keep the current scope focused on Pi extension behavior.

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
  blocked scope, credentials or permissions, external actions, or final
  approval. Any pause for user feedback must record the explicit reason human
  intervention is required.
- Prefer a small protocol that can be enforced over a broad checklist that can
  be ignored.

## Expected Structure

- `README.md`: short explanation and current status.
- `docs/plan.md`: design plan and implementation roadmap.
- `packages/agent-loop-core/`: shared schema, ledger, transition engine, and
  packet builder used by the Pi extension.
- `packages/pi-agent-loop/`: Pi extension runtime.

## Design Constraints

- Keep Pi-specific APIs out of the core protocol package.
- Do not add Codex plugin, skill, CLI, or MCP adapter work unless the project
  scope explicitly changes.
- Store run state in a durable ledger. For Pi-managed runs, Pi custom session
  entries on the active session branch path are authoritative; project-local
  ledgers are inspectable mirrors namespaced by session, for example
  `.agent-loop/sessions/pi/<session-id>/run.json`.
- User-facing commands are only `start`, `stop`, `continue`, and `status`.
  Users never choose phase transitions directly.
- `stop` pauses a loop; it does not cancel it. `continue` resumes from
  reconstructed state.
- Retro is required before a loop closes and records memory or harness behavior
  recommendations only. Retro does not modify the repository.
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
