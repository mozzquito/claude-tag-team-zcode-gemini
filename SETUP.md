# Setup: zcode + agy

This repo documents how to install and use two AI coding CLI agents that run
alongside [Claude Code](https://claude.com/claude-code):

- **zcode** — Z.ai's GLM coding agent (`ZCode.app`)
- **agy** — Google Antigravity's coding agent CLI

Both behave like Claude Code: their own agent loop with file read/write,
tools, and skills. Use them for a second opinion from a different model, or
to run tasks in parallel with Claude.

**Every person installs these under their own account.** No tokens, API
keys, or session files are shared between users — see [Security](#security)
below before you commit anything to a shared repo.

This repo also ships the actual `/zcode` and `/agy` [Claude Code
Skills](https://docs.claude.com/en/docs/claude-code/skills) used to drive
these tools from inside Claude Code — see
[Connecting zcode & agy to Claude Code](#connecting-zcode--agy-to-claude-code)
below.

---

## Prerequisites

- macOS (these steps are written for macOS; both tools also ship for
  Linux/Windows but paths will differ)
- [Node.js](https://nodejs.org/) v20+ (zcode runs on Node — `node --version`
  to check)
- A [Z.ai](https://z.ai/) account (for zcode)
- A Google account (for agy / Antigravity)

---

## Install zcode (Z.ai GLM)

1. Download **ZCode** from Z.ai's official site and install it like any
   macOS app (drag `ZCode.app` into `/Applications`).

2. zcode has no CLI binary on `PATH` by default — it ships as an Electron
   app whose CLI entry point you invoke via `node`. Add this alias to your
   shell config (`~/.zshrc` or `~/.bashrc`):

   ```bash
   alias zcode="node /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs"
   ```

   Then reload your shell:

   ```bash
   source ~/.zshrc
   ```

3. Log in with **your own** Z.ai account:

   ```bash
   zcode login
   ```

   This opens an OAuth flow in your browser. It stores your session locally
   — nothing to share or copy from anyone else.

4. Verify it works:

   ```bash
   zcode doctor
   zcode -p "reply with exactly one word: pong"
   ```

   `zcode doctor` should print a version and `node`/`platform` info. The
   prompt should return `pong`.

### Using zcode

```bash
zcode -p "<prompt>" --cwd "/absolute/path/to/your/repo"
```

- `-p, --prompt` — run one prompt non-interactively and exit (use this for
  scripting; the bare `zcode` command opens a full-screen TUI).
- `--cwd <path>` — always pass this explicitly. zcode has file write access
  wherever it runs.
- `--attach <path>` — attach a file to the prompt (repeatable).
- `--disallowedTools "Edit Write"` — run read-only (good for
  review/analysis tasks that shouldn't touch files).

```bash
zcode -p "Review src/auth.ts for race conditions, report findings only, don't edit anything" \
  --cwd "/path/to/repo" \
  --disallowedTools "Edit Write"
```

**Known quirk**: `zcode` is a shell alias, not a real `PATH` binary. It
works fine from a normal (foreground) shell command, but a backgrounded
shell call that doesn't source your `.zshrc` won't find it. If you need to
run zcode from a background process, call the underlying command directly:

```bash
node /Applications/ZCode.app/Contents/Resources/glm/zcode.cjs -p "<prompt>" --cwd "<path>"
```

---

## Install agy (Google Antigravity CLI)

1. Download and install the **Antigravity** app from Google's official
   site.

2. Launch it once and sign in with **your own** Google account.

3. Install the `agy` CLI shim via Antigravity's command palette (similar to
   VS Code's "install `code` in PATH" command) — look for an "Install `agy`
   command" action. This copies a CLI binary to `~/.local/bin/agy` and adds
   it to your `PATH` automatically (you'll see a block like
   `# Added by Antigravity CLI installer` appended to your shell config).

4. Reload your shell:

   ```bash
   source ~/.zshrc
   ```

5. Verify it works:

   ```bash
   agy --version
   agy models
   agy -p "reply with exactly one word: pong"
   ```

### Using agy

agy is multi-model — pick a backend per task:

```bash
agy -p "<prompt>" --mode plan
```

- `-p, --print` — run one prompt non-interactively and exit.
- `--mode plan` — read-only: agy plans/analyzes but does not write files.
  Default choice for review/analysis.
- `--mode accept-edits` — agy may write files without per-tool confirmation.
  Only use when you actually want it to make changes.
- `--model <name>` — pick a backend. Run `agy models` for the current list
  (e.g. Gemini flash/pro variants, Claude Sonnet).
- `--add-dir <path>` — add an extra directory to agy's workspace.

```bash
agy -p "Review this hook for bugs and security issues, report only" --mode plan
```

### Headless permission wall

In headless/non-interactive use (`-p`/`--print`), agy can hit a permission
wall if a task needs a tool that would normally prompt for approval —
headless mode can't prompt, so the tool call is auto-denied instead. You'll
see:

```
no output produced — a tool required the "<tool>" permission
```

Three ways to work around it, in order of preference:

**1. Rephrase the prompt to avoid needing that tool.** Most hits are a file
read or a shell command agy wanted to run on its own. Paste the content
inline instead of asking agy to fetch it itself:

```bash
# Hits the wall — agy wants to read the file itself
agy -p "Review src/hook.ts for bugs" --mode plan

# Works — no tool call needed, content is already in the prompt
agy -p "Review this file for bugs and security issues, report only:

$(cat src/hook.ts)" --mode plan
```

**2. Use `--sandbox`** (discovered via `agy --help`, 2026-09-03 — not yet
verified end-to-end in this repo's workflows): "Run in a sandbox with
terminal restrictions enabled." This may loosen the permission wall for
read-only/analysis tasks without a blanket skip — worth trying before
reaching for option 3 if you hit this again. If you verify its actual
behavior, replace this paragraph with what you found.

**3. `--dangerously-skip-permissions`** — auto-approves every tool call for
the session. Last resort; only use this for a task you'd trust running fully
unattended, since it also skips the prompts you'd normally want (e.g. before
a destructive shell command).

*(An earlier version of this note also mentioned an agy-side
`permissions.allow` settings rule as a fourth option. That claim could not
be independently verified against `agy --help` or this machine's installed
config in this pass — cut rather than left as an unverified instruction. If
agy does expose a persistent allow-list, document the real config path and
syntax here once confirmed.)*

---

## Connecting zcode & agy to Claude Code

Installing the CLIs (above) is enough to run `zcode` / `agy` by hand from
any terminal. But the actual point of this repo is using them **from
inside Claude Code** — typing `/zcode <task>` or `/agy <task>` and having
Claude shell out to them, read the result, and report back.

There is no API integration here — a **Skill** is just a Markdown file
that Claude Code reads and follows as instructions for that turn. This
repo ships the exact skill files used to drive `/zcode` and `/agy`:

```
.claude/skills/zcode/SKILL.md
.claude/skills/agy/SKILL.md
```

Each one has a YAML frontmatter block (`name`, `description`) that Claude
Code indexes to know when to trigger it, followed by plain-English
instructions — the exact CLI flags to use, when to run read-only, how to
run several instances in parallel, and known quirks to avoid. Claude Code
already has shell access, so "connecting" an agent is just documenting the
right shell command for Claude to run — nothing more.

### To wire this into your own project

Copy the skill folders into your project's `.claude/skills/` directory:

```bash
mkdir -p /path/to/your/project/.claude/skills
cp -r .claude/skills/zcode /path/to/your/project/.claude/skills/
cp -r .claude/skills/agy /path/to/your/project/.claude/skills/
```

Or make them available in **every** project by copying them to your
user-level skills directory instead:

```bash
mkdir -p ~/.claude/skills
cp -r .claude/skills/zcode ~/.claude/skills/
cp -r .claude/skills/agy ~/.claude/skills/
```

Restart Claude Code (or start a new session) in that project, then try:

```
/zcode reply with exactly one word: pong
/agy reply with exactly one word: pong
```

Claude Code will read the matching `SKILL.md`, follow its instructions,
run the CLI command described in it, and report the result back to you.
You can also just describe the task in plain language ("ask zcode to
review this file") — Claude Code matches intent against each skill's
`description` field the same way it matches an explicit `/name`.

### Why this pattern instead of a "real" integration

- No new auth, proxy, or middleware — each CLI keeps its own login
  (per user, per [Security](#security)), and Claude just calls it like any
  other shell command it already has permission to run.
- Read-only by default is enforced in the *instructions*, not in code —
  `--disallowedTools "Edit Write"` (zcode) / `--mode plan` (agy) are just
  flags the skill tells Claude to pass. Editing the `SKILL.md` changes the
  behavior; there's no separate config to keep in sync.
- The same pattern generalizes to any other CLI coding agent — write a
  `SKILL.md` describing its non-interactive flags and safety defaults, and
  Claude Code can delegate to it the same way.

---

## Running both together

Since both are just shelled-out processes, you can run zcode and agy in
parallel — e.g. one reviewing `auth.ts` for race conditions, another
reviewing `db.ts` for query safety — or use one as a second opinion on the
other's output. Split by scope (different files/questions) rather than
pointing both at the same thing hoping for consensus, and default to
read-only flags (`--disallowedTools "Edit Write"` / `--mode plan`) unless
the task genuinely needs file writes, since parallel writers can race on
the same files.

---

## Security

**Never commit secrets from either tool to this or any shared repo.**

- zcode's login (`zcode login`) and agy's Google sign-in each create a
  local session/credential on your machine. These are **per-user** —
  nobody should ever paste, screenshot, or commit their own token/session
  file so someone else can "reuse" it.
- If you see an API key or auth token in your own shell config (e.g. a
  `GLM_AUTH_TOKEN=...` line added by an installer), treat it as a live
  secret:
  - Never commit `~/.zshrc`, `~/.bashrc`, or any dotfile containing it to
    a public (or shared) repo.
  - If a token like this was ever committed, pushed, or shared, rotate/
    revoke it immediately from the provider's console (e.g. Z.ai account
    settings) rather than just deleting the line.
- Keep this repo's own `.gitignore` (see below) covering `.env`, local
  config, and anything else that could carry a credential.
- If you add an `.env.example` for future config, use empty placeholders
  only (`GLM_AUTH_TOKEN=`) — never a real value, even a "test" one.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `command not found: zcode` | Alias not loaded, or running in a script/background shell that didn't source `.zshrc` | Re-run `source ~/.zshrc`, or call the `node .../zcode.cjs` path directly |
| `command not found: agy` | `~/.local/bin` not on `PATH` yet | `source ~/.zshrc`, or re-run Antigravity's "Install `agy` command" |
| zcode/agy prompt hangs forever | Ran without `-p`/`--print` — opened the interactive TUI | Always pass `-p "<prompt>"` for scripted/one-shot use |
| agy: `"no output produced — a tool required ... permission"` | Headless mode can't prompt for tool permission | See [Headless permission wall](#headless-permission-wall) above — rephrase to skip the tool call, try `--sandbox`, or `--dangerously-skip-permissions` as a last resort |
