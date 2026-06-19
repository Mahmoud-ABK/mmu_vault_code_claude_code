to internalise



# Claude Code — Progressive Guide Index

A level-by-level guide from first session to autonomous multi-agent workflows.

---

## Levels

| Level | Title | What you learn |
|---|---|---|
| [[L1-First-Session]] | First Session & Basic Usage | Launch, interaction model, slash commands, keyboard shortcuts, `@` syntax |
| [[L2-Context-CLAUDE-md]] | Context & CLAUDE.md | Context window mechanics, `/clear`, `/compact`, rewind, project-level persistent memory |
| [[L3-Permissions-Settings]] | Permissions & `settings.json` | Allow/deny rules, `defaultMode`, `--dangerously-skip-permissions`, project settings |
| [[L4-Scoping]] | Scoping: Project vs Global | Full `~/.claude/` layout, settings merge order, global vs project config, shell aliases, map of L5–L9 |
| [[L5-Hooks]] | Hooks | Lifecycle events, handler types, stdin payload, production hook scripts |
| [[L6-MCP-Servers-and-Plugins]] | MCP Servers, Plugins & LSP | Protocol, server config, lazy loading, plugin marketplace, semantic code navigation |
| [[L7-Skills-Agents-Commands]] | Skills, Agents & Commands | Reusable workflows, sub-agent isolation, parallel/sequential patterns |
| [[L8-MultiAgent-Headless-Power]] | Multi-Agent, Headless & Power | Git worktrees, CI integration, scheduling, power prompting patterns |
| [[L9-Context-Cost]] | Context Cost of Harness | Token budget of each harness component, global vs local cost, tuning |

---

## Concept map

```
Basic interaction (L1)
    └── Context management (L2)
            └── Project permissions (L3)
                    └── Scoping: project vs global (L4)
                            ├── Hooks — enforcement (L5)
                            ├── MCP + Plugins + LSP — capabilities (L6)
                            ├── Skills/Agents/Commands — reuse + delegation (L7)
                            │       └── Multi-agent + CI — full autonomy (L8)
                            └── Context cost — harness budget (L9)
```

---

## Your `~/.claude/` layout

```
~/.claude/
├── settings.json          ← L4: global model, permissions, defaultMode
├── CLAUDE.md              ← L4: universal conventions, all projects
├── commands/              ← L7: personal cross-project custom commands
├── projects/              ← L2: per-project auto-memory (managed by Claude)
├── sessions/              ← L1: session history (claude -r to browse)
├── plugins/               ← L6: installed plugins
├── cache/
├── backups/
├── history.jsonl
└── telemetry/
```

Your project adds:

```
your-project/
├── CLAUDE.md              ← L2: project conventions and build commands
├── .mcp.json              ← L6: project MCP server definitions
└── .claude/
    ├── settings.json      ← L3: team permissions + hooks (committed)
    ├── settings.local.json← L3: personal overrides (gitignored)
    ├── rules/             ← L4: scoped CLAUDE.md fragments
    ├── hooks/             ← L5: lifecycle hook scripts
    ├── commands/          ← L7: project slash commands
    ├── skills/            ← L7: reusable workflow instructions
    └── agents/            ← L7: sub-agent definitions
```

---

## Quick decision guide

| You want to...                                     | Tool                                 | Level |
| -------------------------------------------------- | ------------------------------------ | ----- |
| Give Claude persistent project knowledge           | Project `CLAUDE.md`                  | L2    |
| Give Claude conventions that apply to all projects | `~/.claude/CLAUDE.md`                | L4    |
| Stop Claude asking permission for every command    | `defaultMode` in `settings.json`     | L3    |
| Block Claude from doing something dangerous        | `deny` rules in `settings.json`      | L3    |
| Configure model and effort level globally          | `~/.claude/settings.json`            | L4    |
| Set up shell shortcuts for Claude                  | Shell aliases in `~/.zshrc`          | L4    |
| Enforce architecture rules mechanically            | Hook — PreToolUse + exit 1           | L5    |
| Auto-format after every edit                       | Hook — PostToolUse                   | L5    |
| Connect Claude to GitHub/DB/Slack                  | MCP server                           | L6    |
| Extend Claude with marketplace capabilities        | Plugin                               | L6    |
| Give Claude semantic code understanding            | LSP plugin                           | L6    |
| Package a repeatable workflow                      | Custom slash command                 | L7    |
| Delegate specialized review to a sub-agent         | Agent definition                     | L7    |
| Run parallel features without conflict             | Git worktrees + multi-agent          | L8    |
| Automate a task in CI                              | Headless `claude -p`                 | L8    |
| Run something on a schedule                        | `/schedule`                          | L8    |
| Diagnose why Claude ignores CLAUDE.md              | Context cost audit                   | L9    |
| Reduce session startup overhead                    | Lazy MCP loading + trim global scope | L9    |

---

## Tips that apply at every level

- [ ] `/clear` between tasks — context from a prior task is noise
- [ ] `/context` at 30-minute marks on long sessions
- [ ] Give Claude a verification loop in the prompt — tests, linter, compiler output
- [ ] Review auth, payment, and data mutation code yourself, always
- [ ] `Esc+Esc` before any approach you're not confident in — checkpoints are free
- [ ] Commit `.claude/settings.json`, `CLAUDE.md`, `.mcp.json` — your team benefits too
- [ ] Never put credentials in config files — use environment variables
