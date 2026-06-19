# Appendix 2 , Settings Examples

Annotated reference for every configurable key across the three `settings.json` files. Each block uses `//` comments as syntactic notes — these are not valid JSON and must be stripped before use. All keys are optional unless noted.

---

## `.claude/settings.json` — project scope, committed

```json
{
  // ── Model & effort ──────────────────────────────────────────────────────────
  // Scalar. Overrides the global default for this project.
  // Project value wins over ~/.claude/settings.json.
  "model": "claude-sonnet-4-6",
  // alternatives:
  //   "claude-opus-4-6"         standard Opus
  //   "claude-opus-4-7"         required for effortLevel "xhigh"
  //   "claude-haiku-4-5-20251001"  fastest, cheapest, low reasoning tasks

  // Scalar. How much extended reasoning Claude applies per turn.
  // Project value wins over global.
  "effortLevel": "high",
  // alternatives:
  //   "low"    trivial edits, quick lookups — fastest, cheapest
  //   "medium" general dev work — balanced
  //   "high"   complex logic, refactoring — default recommendation
  //   "xhigh"  deep architectural reasoning — Opus 4.7 only
  //   "max"    hardest bugs, architecture sessions — burns budget fast

  // ── Permissions ─────────────────────────────────────────────────────────────
  "permissions": {

    // Scalar. Baseline behavior for actions that match no allow/ask/deny rule.
    // Project value wins over global.
    // bypassPermissions is silently ignored here — set it in ~/.claude/settings.json.
    "defaultMode": "acceptEdits",
    // alternatives:
    //   "default"            prompt for everything unlisted (out-of-box default)
    //   "acceptEdits"        reads + file edits + filesystem cmds run silently (recommended)
    //   "plan"               reads only; Claude proposes, does not apply changes
    //   "auto"               everything, with background safety checks (preview, eligible accounts only)
    //   "dontAsk"            only pre-approved allow-listed tools; anything else denied
    //   "bypassPermissions"  everything — valid only in ~/.claude/settings.json or via CLI flag

    // Array. Concatenated with the global allow list — both sets are active.
    // Actions matching any allow rule run silently, no prompt.
    "allow": [
      // Bash — wildcard patterns
      "Bash(./gradlew *)",          // any gradlew task
      "Bash(npm run *)",            // any npm run subcommand
      "Bash(yarn *)",               // alternative: yarn instead of npm
      "Bash(pnpm *)",               // alternative: pnpm
      "Bash(git *)",                // all git operations (narrow below if needed)
      "Bash(git log *)",            // alternative: allow only read-side git
      "Bash(docker compose *)",     // compose up/down/logs etc.
      "Bash(make *)",               // any make target
      "Bash(pytest *)",             // alternative: python test runner
      "Bash(cargo *)",              // alternative: Rust toolchain
      "Bash(go *)",                 // alternative: Go toolchain
      "Bash(* --version)",          // --version on any command

      // Read — file path patterns (gitignore syntax)
      // /path   = relative to project root
      // path    = matches at any depth under cwd (equivalent to **/.env)
      // ~/path  = home-relative
      // //path  = filesystem root
      "Read(~/.zshrc)",             // allow reading your shell config
      "Read(/src/**)",              // everything under project src/

      // WebFetch — domain allowlist
      // Bash(curl *) is always denied; use WebFetch for URL access control.
      "WebFetch(domain:docs.gradle.org)",
      "WebFetch(domain:api.github.com)",
      "WebFetch(domain:registry.npmjs.org)",

      // WebSearch — all-or-nothing, no specifier supported
      "WebSearch",

      // MCP tools
      "mcp__github",                // all tools from the github server
      "mcp__github__*",             // equivalent wildcard form
      "mcp__postgres__execute_query" // single tool only
    ],

    // Array. Concatenated with the global ask list.
    // Actions matching any ask rule always prompt, regardless of defaultMode.
    // Use for checkpoints you want a human to confirm.
    "ask": [
      "Bash(git push *)",           // always prompt before pushing
      "Bash(git push --force *)",   // alternative: narrow to force push only
      "Bash(kubectl apply *)",      // k8s resource changes
      "Bash(terraform apply *)",    // infrastructure changes
      "Edit(/src/auth/**)"          // always prompt before editing auth module
    ],

    // Array. Concatenated with the global deny list — both sets are active.
    // Actions matching any deny rule are blocked silently, no prompt ever.
    // Deny is evaluated first and always wins over allow and ask.
    "deny": [
      // Bash — destructive or dangerous commands
      "Bash(rm -rf *)",             // block all recursive removes
      "Bash(git push --force *)",   // block force push (move to ask if you need it)
      "Bash(git reset --hard *)",   // block hard resets
      "Bash(kubectl delete *)",     // block k8s deletions
      "Bash(kubectl *)",            // alternative: block all k8s if not needed
      "Bash(terraform destroy *)",  // block infra teardown
      "Bash(curl *)",               // deny outright; use WebFetch for URL control
      "Bash(wget *)",               // same

      // Bare tool name — removes the tool from Claude's context entirely.
      // Claude cannot attempt anything with Bash at all.
      // Use when you want the tool eliminated, not just restricted.
      // "Bash",                    // example: remove Bash entirely in read-only sessions

      // Read — secrets and sensitive files
      "Read(.env)",                 // .env at any depth under cwd
      "Read(.env.*)",               // .env.local, .env.production, etc.
      "Read(//**/.env)",            // .env anywhere on the filesystem
      "Read(~/.ssh/**)",            // entire SSH directory
      "Read(*.pem)",                // TLS certificates
      "Read(*.key)",                // private keys
      "Read(*.p12)",                // PKCS12 keystores
      "Read(//etc/passwd)",         // system user database
      "Read(//etc/shadow)",         // password hashes

      // Edit — protect critical modules from writes
      "Edit(/src/auth/**)",         // block writes to auth module
      "Edit(/migrations/**)",       // block writes to migration files

      // MCP tools — block destructive operations
      "mcp__postgres__drop_table",
      "mcp__database__*",           // block an entire server

      // Agent — prevent spawning a specific sub-agent
      "Agent(deploy-agent)"
    ]
  },

  // ── Sandbox (optional, advanced) ────────────────────────────────────────────
  // Isolates bash commands from the filesystem and network.
  // macOS: uses Seatbelt, no install needed.
  // Linux/WSL2: requires bubblewrap + socat.
  // Enable interactively with /sandbox.
  "sandbox": {
    "enabled": true,
    // autoAllowBashIfSandboxed: let all Bash through the permission layer
    // when sandboxed (the sandbox itself is the enforcement boundary).
    "autoAllowBashIfSandboxed": true,
    // Commands excluded from sandboxing — they run in the real environment.
    "excludedCommands": ["git", "docker", "gh"],
    "network": {
      // Allow Claude to bind to localhost ports (e.g. for dev servers).
      "allowLocalBinding": true
    }
  }
}
```

---

## `.claude/settings.local.json` — project scope, gitignored

Same schema as `settings.json`. Higher priority — wins on all scalar conflicts. Arrays concatenate with both `settings.json` and the global file. Never committed.

```json
{
  // Personal scalar overrides for this project only.
  // Omit any key you don't want to override — the project or global value stands.
  "effortLevel": "max",
  // "model": "claude-opus-4-7",   // switch model locally without changing team config

  "permissions": {
    // Personal allow additions. Adds to, does not replace, the project allow list.
    "allow": [
      "Bash(kubectl get *)",               // read-only k8s access for your local setup
      "Bash(kubectl logs *)",
      "Bash(kubectl describe *)",
      "WebFetch(domain:internal.company.dev)" // internal tooling not shared with team
    ]
    // You can also add personal deny rules here.
    // They are active alongside the project deny list, not instead of it.
  }
}
```

> Add `/.claude/settings.local.json` to `.gitignore`. Never commit it.

---

## `~/.claude/settings.json` — global scope

Baseline for every session across all projects. Project files layer on top. Scalars here are the lowest-priority fallback; arrays here concatenate with every project's lists.

```json
{
  // ── Model & effort ──────────────────────────────────────────────────────────
  // Global defaults. Overridden by project settings.json if that file sets the same key.
  "model": "claude-sonnet-4-6",
  "effortLevel": "high",

  // ── UI preferences ──────────────────────────────────────────────────────────
  // Show elapsed time at the end of each turn.
  "showTurnDuration": true,
  // alternatives:
  //   false   hide turn duration (default)

  // How many days to retain session history in ~/.claude/sessions/.
  // Sessions older than this are pruned.
  "cleanupPeriodDays": 30,

  // ── Permissions ─────────────────────────────────────────────────────────────
  "permissions": {
    // Global defaultMode. Overridden by project settings.json if that file sets it.
    // bypassPermissions is valid here (unlike in project scope).
    "defaultMode": "acceptEdits",

    // Global allow list. Active in every project.
    // Concatenated with each project's allow list — both are active simultaneously.
    // Put things here that are safe across all your projects.
    "allow": [
      "Bash(git *)",          // git is safe everywhere; narrow per-project if needed
      "Bash(npm run *)",      // common across most JS projects
      "Bash(make *)",         // common build tool
      "Read(~/.zshrc)"        // let Claude read your shell config for context
    ],

    // Global ask list. Concatenated with each project's ask list.
    "ask": [
      "Bash(git push *)"      // always prompt before any push, across all projects
    ],

    // Global deny list. Active in every project.
    // A rule here cannot be overridden by any project or local file.
    // Put universal blockers here so you don't repeat them in every project.
    "deny": [
      "Bash(rm -rf /*)",      // filesystem root wipe — universally blocked
      "Bash(sudo rm *)",      // sudo removes — universally blocked
      "Bash(curl *)",         // deny outright everywhere; use WebFetch per-project
      "Bash(wget *)",         // same
      "Bash(ssh *)",          // block ssh initiation from Claude
      "Read(//etc/shadow)",   // system password hashes — universally blocked
      "Read(~/.ssh/**)"       // SSH keys — universally blocked
    ]
  }
}
```

---

## `your-project/.mcp.json` — project scope, committed

MCP server definitions for this project. Committed to git so the team shares the same server configuration. Global equivalent is `~/.claude.json` (auto-managed; not hand-edited).

```json
{
  "mcpServers": {

    // ── stdio server (most common) ───────────────────────────────────────────
    // Claude launches this process and communicates over stdin/stdout.
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"],
      // alternatives for args:
      //   ["-y", "@modelcontextprotocol/server-github"]   community package
      //   ["node", "./local-mcp-server.js"]               local server

      // env: values are read from your shell environment at session start.
      // ${VAR} syntax. Never hardcode credentials here.
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      },

      // lazyLoad: true = tool definitions inject into context only when Claude
      //   first calls a tool from this server (recommended default).
      // lazyLoad: false = tool definitions load at session start every time.
      "lazyLoad": true
    },

    // ── Database server ──────────────────────────────────────────────────────
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      // alternatives:
      //   ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"]
      //   ["-y", "@modelcontextprotocol/server-mysql"]
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
        // Use read-only credentials unless writes are explicitly needed.
        // Add a PreToolUse hook on mcp__postgres__execute_query for query logging.
      },
      "lazyLoad": true
    },

    // ── Browser automation ───────────────────────────────────────────────────
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp"],
      // No env needed — Playwright manages its own browser processes.
      "lazyLoad": true
    },

    // ── Extended filesystem access ───────────────────────────────────────────
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/mahmoud/Engineering"   // root path Claude can access via this server
        // alternatives: add multiple paths as additional array elements
      ],
      "lazyLoad": true
    },

    // ── SSE server (alternative transport) ──────────────────────────────────
    // Some servers use HTTP+SSE instead of stdio.
    // "some-remote-server": {
    //   "type": "sse",
    //   "url": "https://mcp.example.com/sse",
    //   "env": {
    //     "API_KEY": "${SOME_API_KEY}"
    //   },
    //   "lazyLoad": true
    // }
  }
}
```

---

## `settings.json` — hooks block

Hooks live inside `settings.json` under `"hooks"`, alongside `"permissions"`. Shown separately here for readability.

```json
{
  "permissions": { },  // ... as above

  "hooks": {

    // ── PreToolUse ───────────────────────────────────────────────────────────
    // Fires before a tool call executes. Supports blocking (exit 2 or JSON deny).
    "PreToolUse": [
      {
        // matcher: which tool calls trigger this group.
        // Omit or use "*" to match all tools.
        // "|"-separated list for multiple exact names.
        // Any other character = JavaScript regex.
        "matcher": "Edit|Write",
        // alternatives:
        //   "*"                        all tools
        //   "Bash"                     Bash only
        //   "mcp__postgres__.*"        all postgres MCP tools (regex)
        //   "mcp__postgres__execute_query"  single MCP tool

        "hooks": [
          {
            // type: how the hook handler runs.
            "type": "command",
            // alternatives:
            //   "http"      POST event JSON to a URL
            //   "mcp_tool"  delegate to a connected MCP server tool
            //   "prompt"    inject text into Claude's reasoning (not deterministic)
            //   "agent"     spawn a sub-agent with isolated context

            "command": "bash .claude/hooks/guard-arch.sh",
            // timeout: seconds before the hook is killed.
            // Always set this — a hanging script hangs Claude's entire turn.
            "timeout": 5,

            // if: pre-filter using permission rule syntax.
            // The script only spawns when this condition matches.
            // Avoids spawning a process on every Edit call when you only care about rm.
            // "if": "Bash(rm *)"
          }
        ]
      }
    ],

    // ── PostToolUse ──────────────────────────────────────────────────────────
    // Fires after a tool call succeeds. Does not support blocking.
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/format.sh",
            "timeout": 15
            // Never block on format failure here — exit 0 always.
            // A formatter error should not block the edit that preceded it.
          }
        ]
      }
    ],

    // ── SessionStart ─────────────────────────────────────────────────────────
    // Fires once when a session opens or resumes. Does not support blocking.
    // stdout on exit 0 is added as context Claude can see.
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/on-start.sh",
            "timeout": 10
            // example use: print branch, last test result, env warnings into context
          }
        ]
      }
    ],

    // ── SessionEnd ───────────────────────────────────────────────────────────
    // Fires once when the session terminates. Does not support blocking.
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/notify.sh",
            "timeout": 5
            // example use: desktop notification when a long task finishes
          }
        ]
      }
    ],

    // ── Stop ─────────────────────────────────────────────────────────────────
    // Fires when Claude finishes a turn. Supports blocking (return decision:block
    // to make Claude continue working). Check stop_hook_active in stdin payload
    // unconditionally — forgetting it creates an infinite loop.
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/enforce-tests.sh",
            "timeout": 120
            // example use: block Claude from stopping if the test suite is red
          }
        ]
      }
    ],

    // ── UserPromptSubmit ─────────────────────────────────────────────────────
    // Fires after prompt submission, before Claude sees it. Supports blocking.
    // Does not support matcher — always fires on every prompt.
    // stdout on exit 0 is added as context Claude can see.
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/prompt-guard.sh",
            "timeout": 5
            // example use: inject project context, block prompts matching a pattern
          }
        ]
      }
    ]

    // ── Other available events (less commonly configured) ────────────────────
    // "PostToolUseFailure"  fires after a tool call fails
    // "PostToolBatch"       fires after a full batch of parallel tool calls
    // "SubagentStart"       fires when a sub-agent is spawned
    // "SubagentStop"        fires when a sub-agent finishes
    // "InstructionsLoaded"  fires when CLAUDE.md or a rules file is loaded
    // "PreCompact"          fires before context compaction
    // "PostCompact"         fires after context compaction
  }
}
```

---

## Key reference

|Key|File(s)|Type|Merge behavior|
|---|---|---|---|
|`model`|all settings.json|string|scalar, highest-priority file wins|
|`effortLevel`|all settings.json|string|scalar, highest-priority file wins|
|`showTurnDuration`|all settings.json|boolean|scalar|
|`cleanupPeriodDays`|all settings.json|number|scalar|
|`permissions.defaultMode`|all settings.json|string|scalar, highest-priority file wins|
|`permissions.allow`|all settings.json|array|concatenated across all files|
|`permissions.ask`|all settings.json|array|concatenated across all files|
|`permissions.deny`|all settings.json|array|concatenated across all files|
|`sandbox.enabled`|settings.json|boolean|scalar|
|`hooks`|settings.json|object|each event array concatenated|
|`mcpServers`|.mcp.json|object|project-scoped only|

> `bypassPermissions` as a `defaultMode` value is silently ignored in `.claude/settings.json`. Set it in `~/.claude/settings.json` or use `--dangerously-skip-permissions` at the CLI instead.