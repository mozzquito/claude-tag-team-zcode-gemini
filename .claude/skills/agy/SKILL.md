---
name: agy
description: Delegate a task to agy — a separate, non-interactive multi-model coding agent CLI (Google Antigravity, offers Gemini 3.x and Claude Sonnet as backends). Use when the user says "agy", "ask agy", wants a second opinion from a different model/agent, or wants a task run by a sibling AI coding agent instead of (or alongside) Claude.
---

# /agy — Delegate to the agy multi-model coding agent

> agy is a separate AI coding CLI (Google Antigravity), installed at `~/.local/bin/agy`. It behaves like Claude Code / zcode — its own agent loop with file read/write and a plan/accept-edits mode split. It's multi-model: `agy models` lists Gemini flash/pro variants and Claude Sonnet. Calling it means handing a task to a **different model** (your choice which one), not spawning a Claude subagent.

## When to use

- User explicitly asks for agy / a second opinion from another agent
- Wants a specific model (e.g. Gemini) tried on the same problem without leaving the session
- Wants a task run in parallel by a sibling coding agent alongside Claude or zcode

Not a default — this is an explicit user choice. For ordinary parallel work inside a single Claude Code session, its own Task/Agent tool is still the default. For Z.ai/GLM specifically, use `/zcode` instead.

## Non-interactive invocation

```bash
agy -p "<prompt>" --mode plan
```

- `-p, --print` (alias `--prompt`) — run one prompt, print the response, exit. Always use this for scripted/one-shot calls; without it, `agy` opens an interactive session and a scripted call will hang.
- `--mode plan` — **read-only equivalent**: agy plans/analyzes but does not write files. Default choice for review/analysis tasks.
- `--mode accept-edits` — agy may write files without per-tool confirmation prompts. Only use when the task actually requires edits.
- `--model <name>` — pick a specific backend (`agy models` to list current options — e.g. Gemini flash/pro variants at various effort levels, and a Claude Sonnet option). Omit to use agy's default.
- `--add-dir <path>` — add an extra directory to agy's workspace (repeatable).
- `--sandbox` — run with terminal restrictions enabled; stacks with `--mode plan` for an extra-cautious read-only pass.
- `--effort low|medium|high` — reasoning effort, where the model supports it.
- `--dangerously-skip-permissions` — **avoid**. Auto-approves all tool permission requests with no prompting; only relevant for a fully unattended run the user has explicitly asked for, and even then confirm with them first.

## Example

```bash
agy -p "Review this hook for bugs and security issues, report findings only" \
  --mode plan
```

```bash
agy -p "Try this same code-review task with Gemini instead of the default model" \
  --mode plan --model gemini-3.6-flash-high
```

## Multiple instances (swarm)

agy is just a shelled-out process, so several can run at once — either multiple agy calls with different `--model`, or alongside `/zcode` calls — each scoped to a different file/module/question. Launch them as parallel shell calls in the same turn, or background them (agy has no alias-in-background issue, unlike zcode).

- **Split by scope, not by duplication** — give each instance a different file, module, or angle rather than pointing several at the same thing for a vote.
- **`--mode plan` on every instance when swarming** unless the task genuinely requires each one to write — parallel writers can race on the same files.
- **Collect and synthesize yourself** — read all the stdout results and reconcile them; don't relay N raw outputs verbatim.
- Since agy is multi-model, a swarm here can diversify *models*, not just scope — e.g. one instance on a Gemini flash model and one on Claude Sonnet reviewing the same code, to catch what a single model's blind spots would miss.

## Choosing a model for the task

`agy models` is the live list. Pick based on task shape, not habit:

| Task shape | Suggested model | Why |
|---|---|---|
| Quick sanity check, single small file, low stakes | a Gemini flash-high (or -medium if truly trivial) | Fast, cheap, good enough for a first pass |
| Deep review, architecture/design critique, cross-file reasoning | a Gemini pro-high variant | More reasoning depth for hard tradeoffs |
| Task benefits from a genuinely different model family (second opinion vs. Claude, or vs. zcode's GLM) | the Claude Sonnet backend or a Gemini variant — pick whichever is *not* what already reviewed it | The point is model diversity, not raw strength |
| Cost/latency-sensitive bulk sweep (many files, low individual stakes) | a Gemini flash-low/medium | Cheapest, run many in parallel |
| Default / no strong signal either way | omit `--model` (agy's default) | Don't over-optimize when the task doesn't call for it |

Match effort to stakes: don't reach for `-high`/`pro` on a one-line typo check, and don't settle for `-low` on something that will get merged and trusted without a human re-check.

## Other useful subcommands

- `agy models` — list available models for `--model`
- `agy agent` / `agy agents` — list configured agents (empty by default on a fresh project)
- `agy changelog` — release notes
- `agy install` — configure environment paths/shell settings (part of first-time setup — see [SETUP.md](../../../SETUP.md))

## Safety notes — same standard as /zcode

- agy is a real autonomous coding agent. `--mode plan` is the safe default for anything shaped like "review" or "analyze." Only switch to `--mode accept-edits` when the task actually needs changes, and confirm before delegating anything that writes files, runs git operations, or touches shared state.
- **Never trust a delegated fix as done just because agy reports success.** Treat any non-trivial delegated fix the same as your own work: write a few concrete test cases and run them against what agy produced before calling it done.
- agy runs as a **separate process** you shell out to — its actions won't show up in the calling session's tool-call history except as the shell command that launched it and its final stdout. Summarize what it reported back to the user.
- In headless/non-interactive use, agy can hit a permission wall if a task needs a tool that would normally prompt for approval (headless mode can't prompt, so the call is auto-denied). See the Troubleshooting table in [SETUP.md](../../../SETUP.md) for the fix.

## Verifying the install

```bash
agy --version
agy models
agy -p "reply with exactly one word: pong"
```
