# Repo Task Proof Loop

Repo Task Proof Loop is a repo-local workflow skill for non-trivial coding tasks.

It creates a durable task folder under `.agent/tasks/<TASK_ID>/`, installs project-scoped Codex and Claude subagents, updates repo guidance, and drives a strict loop:

`spec freeze -> build -> evidence -> fresh verify -> minimal fix -> fresh verify`

The point is simple: keep proof inside the repository, separate implementation from verification, and make task state easy to resume or audit later.

## What It Creates

Inside the target repository:

```text
.agent/tasks/<TASK_ID>/
  spec.md
  evidence.md
  evidence.json
  raw/
    build.txt
    test-unit.txt
    test-integration.txt
    lint.txt
    screenshot-1.png
  verdict.json
  problems.md

.codex/agents/
  task-spec-freezer.toml
  task-builder.toml
  task-verifier.toml
  task-fixer.toml

.claude/agents/
  task-spec-freezer.md
  task-builder.md
  task-verifier.md
  task-fixer.md
```

It also inserts managed workflow blocks into:

- `AGENTS.md`
- the repo's Claude guide file: `CLAUDE.md` or `.claude/CLAUDE.md`

## Install

Install the skill as a project skill.

### Codex

```text
.agents/skills/repo-task-proof-loop/
```

### Claude Code

```text
.claude/skills/repo-task-proof-loop/
```

If you use both tools on the same repository, install it in both locations or keep one canonical copy and sync it.

## Quick Prompts

Use the prompt that matches the current task state.

### Init

```text
Use $repo-task-proof-loop to initialize this repository for the repo-local spec -> build -> evidence -> verify -> fix workflow. Install or refresh the project-scoped subagents, update the managed workflow guidance, and stop after init completes.
```

### Status

```text
Use $repo-task-proof-loop to find the existing repo-local task that matches the task described below, inspect its artifacts, and report the matched task ID, current status, and next recommended step.
...
```

### Continue

```text
Use $repo-task-proof-loop to continue the task described below in this repository. Reuse the matching repo-local task if it already exists; if not, stop after explaining that init should be run first.
...
```

For `Status` and `Continue`, replace `...` with either `Task file: <path/to/task-file.md>` on the next line or the task text pasted on following lines.

## Quick Start

1. Install the skill in the repository.
2. Run the `Init` prompt once for the repo, or run `scripts/task_loop.py init` directly.
3. In Claude Code, if `init` just created or refreshed `.claude/agents/*`, start a new Claude Code session before relying on those agents.
4. Use the `Continue` prompt for real work. The expected loop is `freeze -> build -> evidence -> verify -> fix -> verify`.
5. Validate before sign-off.

## Helper Script

The bundled helper script currently ships three CLI commands:

- `init`
- `validate`
- `status`

The workflow phases `freeze`, `build`, `evidence`, `verify`, `fix`, and `run` are skill-level commands for the agent, not direct CLI subcommands in this package.

Set `SKILL_PATH` to the installed skill directory:

### Codex example

```bash
SKILL_PATH=.agents/skills/repo-task-proof-loop
```

### Claude Code example

```bash
SKILL_PATH=.claude/skills/repo-task-proof-loop
```

Initialize a task:

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" init \
  --task-id feature-auth-hardening \
  --task-file docs/tasks/auth-hardening.md
```

Or seed from inline text:

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" init \
  --task-id feature-auth-hardening \
  --task-text "Implement auth hardening for session refresh and logout."
```

Validate:

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" validate \
  --task-id feature-auth-hardening
```

Status:

```bash
python3 "$SKILL_PATH/scripts/task_loop.py" status \
  --task-id feature-auth-hardening
```

Useful options:

- `--guides auto|agents|claude|both|none`
- `--install-subagents both|codex|claude|none`
- `--force`

With `--guides auto`, the initializer preserves existing guide files, but it also ensures `CLAUDE.md` exists whenever Claude agents are being installed and `AGENTS.md` exists whenever Codex agents are being installed.

## Claude Notes

- Claude Code should use the installed project agents under `.claude/agents/`.
- If `init` created or refreshed `.claude/agents/*` during a running Claude Code session, start a new session before expecting those agents to appear.
- Use `/agents` to inspect the available Claude agents.
- Use `/memory` if you want Claude to open and refine the active memory files.
- Use `claude --agent <name>` only when you intentionally want a direct single-agent session instead of the parent-orchestrated proof loop.
- For this workflow, the default Claude path is to reuse the same builder child for evidence. Only fall back to a second builder in evidence-only mode if the original builder session is unavailable.

## Validation

The package includes a smoke test:

```bash
python3 "$SKILL_PATH/scripts/verify_package.py"
```

It checks the skill structure, initializes temporary repositories, installs the task artifacts and subagents, and verifies the generated task bundles and guide behavior.

## More Detail

The exact role prompts and platform-specific guidance live in:

- `references/COMMANDS.md`
- `references/SUBAGENTS.md`
- `references/REFERENCE.md`
- `SKILL.md`
