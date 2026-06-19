# Appendix 1 — Ignore Files

How Claude Code decides which files exist, which it reads, and what you can do about it.

---

## The critical distinction upfront

`.claudeignore` does not exist as a functional feature of Claude Code.

It was invented by Claude itself in response to user questions, spread across blog posts and engineering wikis, and Claude then trained on that content and kept recommending it. The file is harmless to create — it is just a text file — but no part of Claude Code reads it. If you have one in your project believing it protects sensitive files, it provides zero protection.

The actual mechanisms are `.gitignore`, `respectGitignore` in `settings.json`, and `permissions.deny` rules. Each does a different job.

---

## How Claude Code sees your filesystem

Claude Code uses built-in tools to interact with files: `Read`, `Grep`, and `Glob`. Each has different behavior with respect to ignore files.

| Tool | Respects .gitignore? | Notes                                                                                                                                 |
| ---- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Read | No                   | Reads any file Claude decides to read,regardless of gitignore status.<br>Blocked only by permissions.deny rules.                      |
| Grep | Yes                  | Skips gitignored files by default.<br>To search a gitignored file, Claude passes its path directly.                                   |
| Glob | Inconsistent         | In principle respects .gitignore, but confirmed bugs exist where it does not in all cases. Do not rely on it for security exclusions. |


Note on version history: as of Claude Code v2.1.117 (April 2026), the standalone `Grep` and `Glob` tools were removed on macOS and Linux native builds. Search is now handled through `Bash` using embedded `ugrep` and `bfs` binaries. Windows builds and npm-installed versions retain the original tool-based behavior. The gitignore-respecting behavior of Grep carries over to the new search layer.

The `@`-mention autocomplete in the REPL respects `.gitignore` — gitignored files do not appear in suggestions. This is controlled by `respectGitignore` in `settings.json`.

---

## `.gitignore` — what it actually does in Claude Code

`.gitignore` has two reliable effects in Claude Code:

**Effect 1: `@`-mention autocomplete filtering.** Files matched by `.gitignore` do not appear when you type `@` in the REPL. On by default, controlled by `respectGitignore` in `settings.json`.

**Effect 2: Grep/search scoping.** The search layer skips gitignored files by default. To search a gitignored file, Claude must pass its explicit path.

**What `.gitignore` does NOT do:**

- It does not prevent Claude from calling `Read` on a gitignored file. If Claude decides to read `.env`, it will, regardless of `.gitignore`.
- It does not prevent `Bash` commands like `cat .env` from running.
- It does not reliably prevent `Glob` from returning gitignored paths in all cases — confirmed bugs exist where Glob returns results that should be excluded.

The `respectGitignore` key in `settings.json` controls the `@`-mention autocomplete behavior. It does not control `Read` access.

### The `.gitignore` hierarchy Claude Code reads

Claude Code walks the standard git ignore chain:

```
~/.config/git/ignore          # global gitignore (git's own global config)
your-project/.gitignore       # project-level
your-project/src/.gitignore   # subdirectory-level (git standard)
```

Git's own global ignore file (configured via `git config --global core.excludesfile`) is respected. On most systems this lives at `~/.config/git/ignore` or `~/.gitignore_global`.

> Claude Code has been observed automatically creating or modifying `~/.config/git/ignore` to add `.claude/settings.local.json`, without explicit user consent. This has been filed as a bug (issue #10230). Be aware that Claude Code may touch your global git config.

---

## `respectGitignore` in `settings.json`

```json
{
  "respectGitignore": true
}
```

|Value|Effect|
|---|---|
|`true` (default)|Gitignored files are hidden from `@`-mention autocomplete|
|`false`|All files appear in autocomplete regardless of `.gitignore` status|

This setting controls the `@`-mention file picker. It does not control `Read` access.

To temporarily include a gitignored file in autocomplete for one session, set this to `false` in `.claude/settings.local.json`. Settings files are watched and reloaded live — no restart required.

---

## `permissions.deny` — blocking Read access

A `deny` rule with a `Read(pattern)` entry prevents Claude from opening a matched file through the `Read` tool. When Claude attempts to read a blocked file, it receives an error and cannot access the content.

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(secrets/**)",
      "Read(*.pem)",
      "Read(*.key)",
      "Read(~/.ssh/**)"
    ]
  }
}
```

**Critical limitation: `Read` deny rules do not cover `Glob` or `Grep`.** A file blocked from being read via `permissions.deny` can still be discovered by file pattern search and its path returned to Claude. The deny rule prevents Claude from reading the file's contents, not from knowing the file exists. For build artifacts and irrelevant directories, `.gitignore` is the more comprehensive exclusion because it affects the search layer. For secrets, use both `.gitignore` and `permissions.deny` together.

**Bash is not fully covered.** `Read` and `Edit` deny rules apply to Claude's built-in file tools and to a set of recognized Bash file commands (`cat`, `head`, `tail`, `sed`). They do not apply to arbitrary subprocesses — a Python or Node script that opens files itself is not blocked. For OS-level enforcement that blocks all processes, enable the sandbox.

**`.claudeignore` is not a substitute.** The deny list in `settings.json` is the correct, working mechanism. `.claudeignore` does nothing.

---

## Layered defense for secrets

No single mechanism covers every access path. Use all three layers:

```
1. .gitignore          -> keeps secrets out of Grep/search results
                          and out of @-mention autocomplete
                          does NOT block Read

2. permissions.deny    -> blocks Read tool access
                          does NOT cover Glob path discovery
                          does NOT cover arbitrary subprocesses

3. sandbox             -> OS-level enforcement, blocks all processes
                          highest protection, most friction
```

The practical baseline for any project handling secrets:

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(secrets/**)",
      "Read(*.pem)",
      "Read(*.key)",
      "Read(*.p12)",
      "Read(~/.ssh/**)",
      "Read(//etc/passwd)",
      "Read(//etc/shadow)"
    ]
  }
}
```

Plus in `.gitignore`:

```gitignore
.env
.env.*
*.pem
*.key
*.p12
*.pfx
secrets/
```

Commit both. The team inherits the deny rules from `.claude/settings.json` and the gitignore exclusions from `.gitignore`.

---

## Token efficiency — why this matters beyond security

Every file path Glob returns and every search result Grep produces costs tokens before Claude reads anything. The biggest offenders are generated files and dependency directories.

`.gitignore` is the primary tool here. Add build output and dependency directories to it and they stay out of search results entirely. `permissions.deny` does not help with token efficiency — it blocks reads, not discovery.

---

## Canonical templates

### Project `.gitignore` — token-efficiency additions

```gitignore
# Build output — no value in context
target/
bin/
obj/
dist/
build/
out/
*.class
*.o
*.a
*.so
*.dll
*.exe
*.pyc
__pycache__/

# Dependencies — never read these
node_modules/
vendor/
.venv/
venv/

# Generated files
*.generated.*
*.min.js
*.min.css
coverage/
.nyc_output/

# Logs and runtime data
*.log
*.log.*
logs/

# Secrets — gitignore as the first line of defence
.env
.env.*
*.pem
*.key
*.p12
*.pfx

# Claude Code personal files
CLAUDE.local.md
.claude/settings.local.json
```

### Global gitignore — `~/.config/git/ignore`

Patterns that apply across all repositories. Keep this short — only things that are universally irrelevant:

```gitignore
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# Linux
*~

# Editor artifacts
.idea/
.vscode/settings.json
*.swp
*.swo
.vim/

# Personal Claude Code files (global)
CLAUDE.local.md
```

### Project `settings.json` — deny rules for secrets

Put this in `.claude/settings.json` and commit it so the whole team benefits:

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(secrets/**)",
      "Read(*.pem)",
      "Read(*.key)",
      "Read(*.p12)",
      "Read(~/.ssh/**)",
      "Read(//etc/passwd)",
      "Read(//etc/shadow)"
    ]
  }
}
```

---

## Decision guide

|You want to...|Use|
|---|---|
|Keep build artifacts out of search results|`.gitignore`|
|Keep gitignored files out of `@`-mention autocomplete|`.gitignore` + `respectGitignore: true` (default)|
|Temporarily include a gitignored file in autocomplete|`respectGitignore: false` in `settings.local.json`|
|Block Claude from reading a specific file|`permissions.deny` in `settings.json`|
|Block file access at the OS level (all processes)|sandbox in `settings.json`|
|Use `.claudeignore`|Do not — it does nothing|

---

## Summary

`.gitignore` is the broadest ignore mechanism Claude Code respects. It affects the search/Grep layer and `@`-mention autocomplete. It does not reliably affect `Glob` path discovery and does not affect `Read` at all. `permissions.deny` in `settings.json` controls `Read` access but does not prevent Glob from discovering file paths. Secrets need both `.gitignore` and `permissions.deny`. For full OS-level isolation, enable the sandbox. `.claudeignore` is a hallucination with no implementation.