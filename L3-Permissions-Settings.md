# L3 , Permissions & `settings.json`

**Where you are:** Sessions work, CLAUDE.md is set up, but Claude interrupts constantly for permission. You want to configure what it can and can't do in this project.  
**Goal:** Understand the permission model, configure project-level `settings.json`, and eliminate unnecessary friction without removing real guardrails.

> Global scope configuration , `~/.claude/settings.json` and shell aliases , is covered in [[L4-Scoping]].

---

## The permission model

Every action Claude wants to take is evaluated against three lists. Evaluation order is fixed:

```
deny  →  ask  →  allow  →  defaultMode (if no list matches)
```

The first matching rule wins. Deny always beats ask, ask always beats allow. If nothing matches, `defaultMode` decides.

```
explicitly denied   →  blocked silently, no prompt ever
explicitly asked    →  prompt always, regardless of mode
explicitly allowed  →  executes silently, no prompt ever
unlisted            →  behavior determined by defaultMode
```

This is why Claude asks so often by default: the `allow` list is empty and `defaultMode` is `"default"` (prompt for everything unlisted).

---

## Rule syntax

Rules follow the format `Tool` or `Tool(specifier)`. A bare tool name with no specifier matches every use of that tool.

### Tool types

|Tool|What it covers|
|---|---|
|`Bash`|Shell command execution|
|`PowerShell`|PowerShell execution (Windows)|
|`Read`|File reads, Grep, Glob, `@file` mentions|
|`Edit` / `Write`|File writes and edits|
|`WebFetch`|HTTP fetch requests|
|`WebSearch`|Web search (all-or-nothing, no specifier)|
|`Grep` / `Glob`|Read-only search, generally safe to allow|
|`NotebookEdit`|Jupyter notebook edits|
|`Agent`|Sub-agent spawning|
|`mcp__server__tool`|MCP server tools|

### Bash patterns

`*` matches any sequence of characters including spaces, so one wildcard can span multiple arguments. A space before `*` enforces a word boundary: `Bash(ls *)` matches `ls -la` but not `lsof`, while `Bash(ls*)` matches both. The `:*` suffix is equivalent to a trailing `*`, so `Bash(npm:*)` and `Bash(npm *)` are identical.

```json
"allow": [
  "Bash(npm run *)",        // any npm run subcommand
  "Bash(git commit *)",     // git commit with any args
  "Bash(git * main)",       // any git command targeting main
  "Bash(* --version)",      // --version on any command
  "Bash(./gradlew *)"       // any gradlew task
],
"deny": [
  "Bash(git push *)",       // block all pushes
  "Bash(rm -rf *)",         // block destructive removes
  "Bash(kubectl *)"         // block k8s entirely
]
```

Compound commands are parsed and each subcommand checked independently. `Bash(safe-cmd *)` does not cover `safe-cmd && other-cmd` — the other subcommand needs its own rule.

A built-in set of read-only commands (`ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only git forms) run without a prompt in every mode. This set is not configurable.

Claude Code strips a fixed set of process wrappers before matching: `timeout`, `time`, `nice`, `nohup`, `stdbuf`, and bare `xargs`. So `Bash(npm test *)` also covers `timeout 30 npm test`.

> **Do not use Bash patterns to filter URLs.** `Bash(curl https://api.example.com/ *)` is trivially bypassed by reordered flags, variables, or redirects. Deny `curl` and `wget` outright and use `WebFetch(domain:...)` rules for URL control instead.

### Read and Edit path patterns

Read and Edit rules follow the gitignore specification. Four anchoring forms:

|Pattern|Anchored to|Example|Matches|
|---|---|---|---|
|`//path`|Filesystem root|`Read(//home/alice/secrets/**)`|`/home/alice/secrets/**`|
|`~/path`|Home directory|`Read(~/.zshrc)`|`~/.zshrc`|
|`/path`|Project root|`Edit(/src/**/*.ts)`|`<project>/src/**/*.ts`|
|`path` or `./path`|Current directory|`Read(.env)`|`<cwd>/**/.env`|

`*` matches within a single directory. `**` matches recursively across directories. A bare filename like `Read(.env)` is equivalent to `Read(**/.env)` and matches at any depth under the current directory.

> A pattern like `/Users/alice/file` is **not** an absolute path — it is relative to the project root. Use `//Users/alice/file` for an absolute path.

```json
"deny": [
  "Read(.env)",           // .env at any depth under cwd
  "Read(//**/.env)",      // .env anywhere on the filesystem
  "Read(~/.ssh/**)",      // everything under ~/.ssh
  "Read(*.pem)",          // .pem files at any depth under cwd
  "Edit(/src/auth/**)"    // block edits to the auth module specifically
]
```

### WebFetch patterns

`WebFetch` takes a `domain:` specifier. `WebSearch` is all-or-nothing — no specifier is supported.

```json
"allow": [
  "WebFetch(domain:api.github.com)",
  "WebFetch(domain:docs.anthropic.com)",
  "WebSearch"
],
"deny": [
  "WebFetch(domain:pastebin.com)"
]
```

### MCP tool patterns

MCP tools use `mcp__server__tool` naming. Two scoping levels:

```json
"allow": [
  "mcp__github",                     // all tools from the github server
  "mcp__github__*",                  // equivalent wildcard form
  "mcp__postgres__execute_query"     // single tool only
],
"deny": [
  "mcp__postgres__drop_table",       // block one destructive tool
  "mcp__database__*"                 // block an entire server
]
```

`mcp__server` and `mcp__server__*` are equivalent and both match all tools from that server.

### Agent patterns

Control which sub-agents Claude can spawn:

```json
"deny": [
  "Agent(deploy-agent)"    // prevent Claude from spawning this agent
]
```

### Bare tool names vs scoped rules

There is an important behavioral difference in deny rules specifically. A bare `Bash` in the deny list removes the tool from Claude's context entirely — Claude never sees it and cannot attempt anything with it. A scoped `Bash(rm *)` leaves the tool available but blocks matching calls when Claude attempts them. Use the bare form when you want to eliminate the tool entirely; use the scoped form when you want the tool available but with specific restrictions.

### The `ask` list

A third list forces a confirmation prompt even for actions that would otherwise auto-run under `acceptEdits` or `auto` mode:

```json
"ask": [
  "Bash(git push *)"    // always prompt before pushing, regardless of mode
]
```

---

## `--dangerously-skip-permissions` , what it actually does

This flag removes the **prompt layer** only. It does not touch `allow`, `ask`, or `deny` rules.

```
policy layer (allow / ask / deny rules)   ← unaffected by the flag
       ↓ if action passes policy
prompt layer ("Allow? y/n")               ← this is what the flag removes
       ↓
execution
```

The practical pattern: solid deny list in `settings.json`, flag on the session.

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push *)",
      "Bash(kubectl *)"
    ]
  }
}
```

```bash
claude --dangerously-skip-permissions
```

Deny rules remain fully enforced. Claude never gets interrupted for safe commands, and can never run the blocked ones regardless of what it decides.

---

## `defaultMode` , baseline behavior

`defaultMode` controls what happens to actions that match no rule in `allow`, `ask`, or `deny`.

|Value|What runs without a prompt|
|---|---|
|`"default"`|Read-only commands only|
|`"acceptEdits"`|Reads, file edits, and common filesystem commands (`mkdir`, `touch`, `mv`, `cp`, etc.) for paths in the working directory|
|`"plan"`|Reads only; Claude proposes changes rather than making them|
|`"auto"`|Everything, with background safety checks (research preview, requires eligible account)|
|`"dontAsk"`|Only pre-approved tools; anything not in `allow` is denied|
|`"bypassPermissions"`|Everything (root and home directory removals still prompt as a circuit breaker)|

`"acceptEdits"` is the practical middle ground for most development work. File operations go through silently; shell commands still prompt.

`"bypassPermissions"` in `defaultMode` is silently ignored when placed in project-level `.claude/settings.json`. Claude Code does not allow a repository to grant itself bypass permissions, since that would let an untrusted repo opt out of all safety checks when cloned. To use `bypassPermissions` as a persistent default, set it in `~/.claude/settings.json` (user scope) or pass `--dangerously-skip-permissions` at the CLI.

`Shift+Tab` in the CLI cycles through `default` → `acceptEdits` → `plan`. `auto` and `bypassPermissions` appear in the cycle only when your account is eligible and they are enabled.

---

## Project `.claude/settings.json`

This file is **committed to git** and shared with the team. It defines what Claude is allowed to do in this project.

```json
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": [
      "Bash(./gradlew *)",
      "Bash(docker compose *)",
      "Bash(git *)",
      "Bash(npm run *)",
      "WebFetch(domain:docs.gradle.org)",
      "WebSearch"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(kubectl delete *)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Read(.env)",
      "Read(~/.ssh/**)",
      "Read(*.pem)",
      "Read(*.key)"
    ]
  }
}
```

---

## `/permissions` , inspecting the active config

`settings.json` is the source of truth, but it does not tell you what the _effective_ merged config looks like after all six layers combine. `/permissions` does.

```
/permissions
```

It opens an interactive panel that shows:

- Every active `allow`, `ask`, and `deny` rule
- Which `settings.json` file each rule came from (global, project, local, managed)
- The current `defaultMode`

Use it to diagnose why Claude is prompting for something you thought you allowed, or why it ran something you thought you blocked. The source column is especially useful when a global rule and a project rule are interacting unexpectedly.

You can delete individual rules from the panel. To add rules in bulk, edit `settings.json` directly , the panel is read-heavy, the file is write-heavy.

```
/status
```

`/status` is the companion command: it shows which settings files are loaded and from which paths, without listing individual rules. Use it first if you are not sure which config files Claude is reading in the current session.

---

## `.claude/settings.local.json` , personal overrides

Not committed. For things like extra permissions you need personally but shouldn't apply to the team.

```json
{
  "permissions": {
    "allow": [
      "Bash(kubectl get *)",
      "Bash(kubectl logs *)",
      "WebFetch(domain:internal.company.dev)"
    ]
  },
  "effortLevel": "max"
}
```

Add to `.gitignore`:

```
.claude/settings.local.json
```

---

## Model and effort configuration

```json
{
  "model": "claude-sonnet-4-6",
  "effortLevel": "high"
}
```

|`effortLevel`|Behavior|Use when|
|---|---|---|
|`"low"`|Minimal reasoning, fastest|Trivial edits, quick lookups|
|`"medium"`|Balanced|General dev work|
|`"high"`|Extended thinking|Complex logic, refactoring|
|`"xhigh"`|Between high and max, Opus 4.7 only|Deep architectural reasoning without full max cost|
|`"max"`|Maximum reasoning|Architecture sessions, hardest bugs|

The pragmatic default: Sonnet + `high` for everyday work. Use Opus + `max` only for sessions where the reasoning depth genuinely matters , it burns your daily budget fast. `xhigh` gives a useful middle point when `max` is too costly but `high` is not enough.

Change mid-session without restarting:

```
/model
/effort
```

---

## Pro subscription , practical reality

| Scenario                        | Recommendation                                                          |
| ------------------------------- | ----------------------------------------------------------------------- |
| Routine dev work (2–3 hrs/day)  | Pro is sufficient. Use Sonnet + `high`.                                 |
| Architecture sessions           | Switch to Opus for those sessions via `/model`                          |
| Hitting limits on heavy days    | Lower `effortLevel` to `medium`; reserve `high`/`max` for complex tasks |


On Pro: `max` effort + Opus for anything but short sessions will hit limits before noon. The pragmatic strategy: Sonnet + `high` as default, Opus + `max` only for architectural planning sessions that genuinely benefit from extended reasoning.

---

## Sandboxing (advanced, optional)

Claude Code supports isolating bash commands from your filesystem and network. On macOS it uses Seatbelt out of the box. On Linux and WSL2, install `bubblewrap` and `socat` first. 
```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["git", "docker", "gh"],
    "network": {
      "allowLocalBinding": true
    }
  }
}
```

When sandboxing is enabled, `WebFetch` domain rules and `Read`/`Edit` deny rules both contribute to the sandbox boundary. Deny rules alone stop Claude from attempting an action; sandboxing stops the underlying process from completing it even if Claude tries. Use both for defense in depth on untrusted codebases.

Enable interactively with `/sandbox`.

---

## Tips

- [ ] Run `/permissions` any time Claude's behavior doesn't match what you configured , it shows the full merged ruleset and which file each rule came from
- [ ] Run `/status` first if you are not sure which config files Claude is reading in the current session
- [ ] Start with `defaultMode: acceptEdits` , it eliminates most prompt interruptions with minimal risk
- [ ] Use the `ask` list for actions that warrant a human checkpoint (git push, database migrations) rather than trusting yourself to catch them in review
- [ ] Deny `curl` and `wget` outright; use `WebFetch(domain:...)` rules for URL control , Bash patterns for URLs are trivially bypassed
- [ ] Bare `Bash` in deny removes the tool entirely; scoped `Bash(rm *)` blocks only matching calls , pick the right form for your intent
- [ ] Commit `.claude/settings.json` to every project , the team gets the same guardrails
- [ ] `.claude/settings.local.json` → always in `.gitignore`
- [ ] Never put API keys or credentials in settings files , use environment variables
- [ ] Do not put `defaultMode: bypassPermissions` in project `.claude/settings.json` , it is silently ignored. Use `~/.claude/settings.json` or `--dangerously-skip-permissions` instead.
- [ ] `/model` mid-session lets you switch without restarting , use Opus for planning, Sonnet for execution

---

## What's next

Project-level permissions control what Claude can do in this project. L4 zooms out , it covers **global scope** (`~/.claude/`), how project and global settings compose, and maps the full set of primitives you'll master in L5–L9.