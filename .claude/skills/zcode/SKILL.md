---
name: zcode
description: Delegate a task to zcode (Z.ai GLM coding agent) running as a separate, non-interactive CLI process. Use when the user says "zcode", "ask zcode", wants a second opinion from a different model/agent, or wants a task run by a sibling AI coding agent instead of (or alongside) Claude.
---

# /zcode — Delegate to Z.ai's GLM coding agent

> zcode is a separate AI coding CLI (Z.ai / GLM models), installed at `/Applications/ZCode.app`, aliased as `zcode` in the shell. It behaves like Claude Code — its own agent loop with file read/write, tools, skills, and MCP support. Calling it means handing a task to a **different model**, not spawning a Claude subagent.

## When to use

- User explicitly asks for zcode / GLM / a second opinion from another agent
- Wants a second opinion from a different model on the same problem
- Wants a task run in parallel by a sibling coding agent (e.g. one review from Claude, one from zcode)

Do not reach for this by default — it's an explicit user choice, not a general-purpose subagent replacement. For ordinary parallel work inside a single Claude Code session, its own Task/Agent tool is still the default.

## Non-interactive invocation

```bash
zcode -p "<prompt>" --cwd "<absolute/path/to/repo>"
```

- **`zcode` is a shell alias** (`alias zcode="node /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs"`, set up during install — see [SETUP.md](../../../SETUP.md)), not a real `PATH` binary — `which zcode` / `type zcode` report "not found." Foreground shell calls work fine (they source your shell config, which picks up the alias), but a **backgrounded** shell call that skips shell init will not — the alias fails with `command not found: zcode` (exit 127). For any backgrounded/scripted zcode call, skip the alias and call the underlying command directly instead: `node /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs -p "<prompt>" --cwd "<path>"` — same flags, same behavior, resolves regardless of foreground/background. (`agy`, by contrast, is a real installed binary and has no equivalent issue.)
- `-p, --prompt` — run one prompt, print result, exit (no TUI). Always use this for scripted/one-shot calls; the bare `zcode` command opens a full-screen interactive TUI and will hang.
- `--cwd <path>` — **always pass this explicitly** rather than relying on the shell's current directory. zcode is a full agent with file write access; an unset `--cwd` risks it operating on the wrong project.
- `--attach <path>` — attach a file to the prompt (repeatable for multiple files).
- `--disallowedTools "Bash(git push*) Edit"` — deny specific tools/patterns, same shape as Claude Code's own tool-permission strings. Use this to keep a delegated task read-only when it doesn't need file writes (e.g. "just review this and report back").
- `--locale en-US|zh-CN|auto` — UI/response locale if needed.

## Example

```bash
zcode -p "Review src/auth.ts for race conditions, report findings only, don't edit anything" \
  --cwd "/path/to/repo" \
  --disallowedTools "Edit Write"
```

```bash
zcode -p "Implement the CSV export button described in the ticket" \
  --cwd "/path/to/repo"
```

## Multiple instances (swarm)

zcode is just a shelled-out process, so nothing stops running several at once — either alongside other zcode calls, or alongside `/agy` calls — each scoped to a different file/module/question. Launch them as parallel shell calls in the same turn (or background them, using the full `node .../zcode.cjs` path per the alias caveat above, not the alias).

- **Split by scope, not by duplication** — give each instance a different file, module, or angle (e.g. one on `auth.ts` for race conditions, one on `db.ts` for query safety) rather than pointing multiple instances at the same thing hoping for consensus.
- **Read-only by default when swarming** — pass `--disallowedTools "Edit Write"` on every instance unless the task genuinely requires each one to write, since parallel writers can race on the same files.
- **Collect and synthesize yourself** — read all the stdout results and reconcile them; don't relay raw output from N processes verbatim to the user.
- zcode only exposes GLM as a backend (no `--model` switch like agy), so a zcode swarm is about splitting scope, not diversifying models — pair it with `/agy` instances if you want model diversity too.

## Other useful subcommands

- `zcode doctor` — sanity-check the install (version, runtime, platform)
- `zcode skills list` — see zcode's own local skills
- `zcode login` / `zcode logout` — Z.ai OAuth session management (each user logs in with their own account)
- `zcode version` — print CLI version

## Safety notes

- zcode is a real autonomous coding agent with file write access in whatever `--cwd` you give it — treat a delegated task with the same care as any risky/hard-to-reverse action: confirm with the user before delegating anything that writes files, runs git operations, or touches shared state, same as you would before doing it yourself.
- Prefer `--disallowedTools` to scope a delegated task down to read-only when the task is "review/analyze" rather than "implement."
- zcode runs as a **separate process** you shell out to — it is not a Claude subagent, so its actions won't show up in the calling session's tool-call history except as the shell command that launched it and its final stdout. Summarize what it reported back to the user rather than assuming they saw it.

## Verifying the install

```bash
zcode doctor
zcode -p "reply with exactly one word: pong"
```

`zcode doctor` should print a version and runtime/platform info; the prompt should return `pong`.
