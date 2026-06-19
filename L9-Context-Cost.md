# L9 , Context Cost of Harness

**Where you are:** You've built the full harness , CLAUDE.md at multiple scopes, settings, hooks, MCP servers, plugins, LSP, skills, and agents. Something feels slow, or Claude seems to be ignoring instructions buried deep in your config.  
**Goal:** Understand the token cost each harness component imposes, how global vs project scope amplifies or contains that cost, and how to tune the harness to stay within budget.

---

## What "harness cost" means

Every component you've added since L2 consumes tokens from Claude's context window before you type a single message. The harness is the total of:

- CLAUDE.md files (global + project + subdirectory + rules/ fragments)
- MCP server tool definitions
- Plugin tool definitions
- LSP tool definitions

You're not paying for hooks at startup , hooks execute out-of-band. But everything else loads before your first message.

---

## What loads at session start

```
Session opens
    ├── ~/.claude/CLAUDE.md                    ← always, every session
    ├── ~/.claude/settings.json               ← always
    ├── your-project/CLAUDE.md                ← project sessions only
    ├── your-project/.claude/settings.json    ← project sessions only
    ├── MCP tool definitions                  ← each connected server
    │   ├── eager servers: loaded immediately
    │   └── lazy servers: deferred to first use
    └── Plugin + LSP tool definitions         ← each active plugin
```

Subdirectory CLAUDE.md files and `.claude/rules/` fragments are lazy , they load only when Claude reads files in matching paths, not at session start.

---

## Global vs project cost

The scope of a harness component determines how often you pay its cost:

```
Global scope (~/.claude/)
  → pays on every session, every project, always

Project scope (.claude/)
  → pays only when working in that project
```

A 500-line global CLAUDE.md consumes those tokens in your nvim session, your Spring Boot session, your data analysis session , even when none of it is relevant. A well-scoped project CLAUDE.md is free in other contexts.

---

## Per-component cost

### CLAUDE.md files

| Component | When it loads | Target size | Risk |
|---|---|---|---|
| `~/.claude/CLAUDE.md` | Every session | < 50 lines | Wastes budget in unrelated projects |
| `project/CLAUDE.md` | Project sessions | < 150 lines | Diminishing returns past ~200 lines |
| `.claude/rules/*.md` | Lazy, path-matched | 30–80 lines each | None if path patterns are precise |
| Subdirectory `CLAUDE.md` | Lazy, path-matched | Any | None if directory scope is narrow |

Claude's attention distributes across a large input. CLAUDE.md content past ~200 lines gets partially ignored , not because it's truncated, but because attention is finite. The instructions are still there; Claude just stops weighting them equally.

### MCP servers and plugins

Each server injects its full tool schema into context on connection:

```
Small server (5 tools)     →  ~1,000–2,000 tokens
Medium server (20 tools)   →  ~4,000–8,000 tokens
Large server (50+ tools)   →  ~10,000–20,000 tokens
```

**Mitigation:** `lazyLoad: true` on all servers. Tool definitions inject only when Claude first calls a tool from that server, not at session start.

```json
"mcpServers": {
  "github": {
    "command": "npx",
    "args": ["-y", "@anthropic-ai/mcp-server-github"],
    "lazyLoad": true
  }
}
```

Global MCP configuration (in `~/.claude.json`) means those servers are available in every project. Prefer project-scoped `.mcp.json` for servers that are project-specific.

### LSP

LSP tools load when enabled globally. The tool definitions themselves are lightweight , LSP's context cost comes from the results it returns (call graphs, reference lists), not from loading the tool. The tradeoff runs the other direction: LSP saves context by replacing broad grep searches with precise, scope-aware results.

---

## How to audit your harness cost

Inside any session, run `/context` before doing any work. The number you see is your baseline harness overhead.

```bash
# Rough token estimate for your CLAUDE.md files
# ~1 token per 0.75 words (English prose), ~1 token per character (code/config)
wc -w ~/.claude/CLAUDE.md
wc -w your-project/CLAUDE.md
```

For a full picture:

```bash
# See all CLAUDE.md files that could load
find ~/.claude -name "CLAUDE.md"
find your-project -name "CLAUDE.md" -o -name "*.md" -path "*/.claude/rules/*"

# List active MCP servers and plugins
cat ~/.claude.json
cat your-project/.mcp.json
ls ~/.claude/plugins/
```

---

## Signs your harness is too heavy

- `/context` shows > 15% usage before you've done any work
- Claude ignores CLAUDE.md instructions it followed earlier in the conversation
- Long sessions degrade faster than expected
- Claude confuses conventions from different projects (global CLAUDE.md bleeding through)

---

## Tuning table

| Symptom | Fix |
|---|---|
| High global baseline | Trim `~/.claude/CLAUDE.md` aggressively; move project-specific content out |
| MCP context overhead | Enable `lazyLoad: true` on all servers; remove servers unused weekly |
| Plugin overhead | Audit `~/.claude/plugins/`; uninstall anything not used across multiple projects |
| Claude ignoring CLAUDE.md mid-session | File is too long , split into `.claude/rules/` (lazy-loaded); use hooks for critical rules |
| Conventions bleeding across projects | Global CLAUDE.md has project-specific content , move it to the project |

---

## The right tool for enforcement

CLAUDE.md rules are advisory and context-dependent. As sessions grow, adherence degrades. For rules that must always apply:

```
CLAUDE.md rule         →  ~80% adherence, degrades with context fill
Hook (PreToolUse)      →  100% adherence, context-independent, deterministic
```

If a rule matters enough that you'd be upset when Claude ignores it, it belongs in a hook, not CLAUDE.md.

---

## Anti-patterns

| Anti-pattern                                     | Why it hurts                                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Global CLAUDE.md > 100 lines                     | Pays context cost in every session, including unrelated projects                                        |
| More than 20k tokens of active MCP definitions   | Drops usable working context to near zero                                                               |
| Using `max` effort on Pro for everything         | Burns your daily budget on tasks that don't need extended reasoning                                     |
| Sessions longer than the task                    | A degraded 3-hour session produces worse output than a clean fresh session                              |
| `@`-embedding documentation files in CLAUDE.md   | Bakes full file content into every session regardless of relevance , reference paths as strings instead |
| Ignoring Claude's output without running tests   | Claude's code looks correct more often than it is , always run the test suite                           |
| Global MCP servers for project-specific services | Pays their context cost in every project, not just the one that uses them                               |

---

## Tips

- [ ] Run `/context` at session start , your baseline is your harness cost
- [ ] Global CLAUDE.md under 50 lines , it loads in every session
- [ ] `lazyLoad: true` on all MCP servers is the correct default
- [ ] Prefer `.claude/rules/` over a long CLAUDE.md , rules are lazy-loaded
- [ ] Hooks are the right tool for rules that must survive long sessions , they're context-independent
- [ ] Audit `~/.claude/plugins/` and `~/.claude.json` periodically , remove what you don't actively use
- [ ] If Claude stops following a CLAUDE.md rule mid-session, it's usually context saturation , `/compact retain [rule]` and continue
- [ ] Project-scope MCP servers cost nothing when you're outside that project; global servers cost always
