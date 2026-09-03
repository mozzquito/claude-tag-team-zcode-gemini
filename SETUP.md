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

**Known quirk**: in headless/non-interactive use, agy can hit a permission
wall if a task needs a tool that would normally prompt for approval
(headless mode can't prompt, so the tool call is auto-denied). If you hit
`"no output produced — a tool required the ... permission"`, either:

- rephrase the prompt to avoid needing that tool (e.g. paste content inline
  instead of asking agy to read a file), or
- add an explicit allow-rule under `permissions.allow` in your agy/Claude
  settings, or
- as a last resort, `--dangerously-skip-permissions` (auto-approves every
  tool call — only use this for a task you'd trust running fully
  unattended).

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
| agy: `"no output produced — a tool required ... permission"` | Headless mode can't prompt for tool permission | See the agy "Known quirk" note above |
