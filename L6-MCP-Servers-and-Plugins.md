# L6 , MCP Servers, Plugins & LSP

**Where you are:** Claude can manage your files, run your build, and enforce your architecture rules. But it can't talk to GitHub, your database, or external services , and it navigates code by text search rather than semantic understanding.  
**Goal:** Understand MCP, plugins, and LSP as three capability extension mechanisms: external services, marketplace packages, and language-aware code navigation.

---

## MCP , connecting to external services  

MCP (Model Context Protocol) is a standardized protocol for connecting language models to external data sources and services. An MCP server is a process that exposes a set of **tools** , functions Claude can call the same way it calls `Bash` or `Edit`.

Without MCP:
```
Claude → files, shell commands
```

With MCP:
```
Claude → files, shell commands, GitHub API, your database, Slack, deploy systems, anything
```

Each connected MCP server gives Claude a new vocabulary of actions. Claude decides when to invoke them the same way it decides when to read a file.

---

## The context cost tradeoff

Every active MCP server loads its tool definitions into the context window at session start. This is the primary constraint to be aware of.

The rule from Anthropic: **if a CLI tool exists for the service (`gh`, `aws`, `psql`, `gcloud`), prefer the CLI**. CLI tools have zero context overhead. MCP has startup cost.

Use MCP when:
- No good CLI exists
- The service has a structured API Claude benefits from (typed responses, not raw text)
- You're doing complex multi-step operations that need structured data exchange
- The CLI UX requires complex argument assembly

Use CLI when:
- The CLI is mature and expressive (`gh`, `git`, `kubectl`, `aws`)
- The task is one-off or simple
- You want to minimize context consumption

---

## MCP configuration

MCP servers are defined in two places:

```
~/.claude.json           # user-scope servers , available in all projects
your-project/.mcp.json   # project-scope servers , project-specific
```

Project-scoped servers in `.mcp.json` (committed to git):

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

Environment variables in `"env"` use `${VAR_NAME}` syntax , they're read from your shell environment at session start. Never hardcode credentials in the JSON.

---

## Lazy loading , the context fix

MCP tool search enables lazy loading. Tool definitions only inject into context when Claude actually needs them , not at session start.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" },
      "lazyLoad": true
    }
  }
}
```

With lazy loading you can connect many servers without burning context budget on tools Claude won't use in a given session.

---

## Managing servers inside a session

```
/mcp                     # show connected servers and status
/mcp add <server>        # add a server to this session
```

The `/mcp` command shows you which servers are connected, which tools are available, and their current auth status.

---

## Practical servers , what to actually connect

### GitHub

```json
"github": {
  "command": "npx",
  "args": ["-y", "@anthropic-ai/mcp-server-github"],
  "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
}
```

What Claude can do: create PRs, read/comment on issues, list files in repos, get PR diffs, merge PRs. More natural than piping through `gh` for complex multi-step workflows.

Generate a token: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens.

### PostgreSQL / SQLite

```json
"postgres": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-postgres"],
  "env": { "DATABASE_URL": "${DATABASE_URL}" }
}
```

What Claude can do: schema introspection, query execution, explain plans. Useful for debugging queries or exploring a schema Claude has never seen.

> Use read-only credentials unless you explicitly need writes. Add a PreToolUse hook on `postgres/execute_query` to log or block DDL operations.

### Playwright (browser automation)

```json
"playwright": {
  "command": "npx",
  "args": ["-y", "@playwright/mcp"]
}
```

What Claude can do: open a browser, navigate, interact, screenshot, extract data. Closes the loop for UI verification , Claude can see what it built in the browser, not just what it wrote.

### Filesystem (extended)

```json
"filesystem": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/mahmoud/Engineering"]
}
```

Gives Claude structured file operations beyond the built-in Read/Edit , directory listings with metadata, glob matching, recursive search. Useful when working across project boundaries.

---

## Adding MCP to hooks

You can fire hooks on MCP tool calls specifically:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__postgres__execute_query",
        "hooks": [{
          "type": "command",
          "command": "bash .claude/hooks/log-queries.sh",
          "timeout": 5
        }]
      }
    ]
  }
}
```

The `matcher` field uses the same `mcp__server__tool` naming as permission rules; a regex like `mcp__postgres__.*` matches every tool from one server. Useful for auditing, rate-limiting, or blocking specific MCP operations.

---

## GitHub App , automated PR review

```bash
# inside a session
/install-github-app
```

Installs Claude as a GitHub App on your repo. Claude automatically reviews every PR you open. Scope it with `.claude/claude-code-review.yml`:

```yaml
direct_prompt: |
  Review this PR for bugs and security issues only.
  Ignore style, formatting, and non-critical suggestions.
  Be concise. Only report actual problems with file, line, severity, and description.
```

---

## Plugins

Plugins are packages from the Claude Code marketplace that install capabilities into Claude , MCP servers, skills, agents, and commands , without manual configuration.

```bash
/plugin install <plugin-name>   # install a plugin
/plugin list                    # list installed plugins
/plugin                         # plugin manager
```

Installed plugins live in `~/.claude/plugins/` , globally available across all projects.

Well-maintained plugins:
- **context7** , fetches up-to-date library documentation on demand
- **Playwright** , browser automation (if you prefer plugin install over manual MCP config)
- **GitHub** , PR and issue management for complex multi-step workflows

Plugin tool definitions load the same way MCP server tools do. Use `lazyLoad: true` where available. Audit installed plugins periodically:

```bash
ls ~/.claude/plugins/
```

To configure a plugin's underlying MCP server with project-specific settings, override it in `.mcp.json`:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"],
      "lazyLoad": true
    }
  }
}
```

---

## LSP , semantic code navigation

LSP (Language Server Protocol) integration gives Claude language-server-level code understanding: find-references, go-to-definition, type information, rename impact analysis , rather than text search.

The token difference matters on large codebases. Grep reads every text match including comments and unrelated names. LSP returns only exact call sites with full scope awareness, at roughly a quarter of the token cost.

**Without LSP vs with LSP:**

| Without LSP | With LSP |
|---|---|
| Text grep , returns all string matches | `findReferences` , exact call sites with scope awareness |
| Infers types from naming conventions | Reads actual type signatures from language server |
| Runs compiler to see errors | Sees diagnostics after every edit, same turn |
| ~2,000+ tokens for grep on 100-file project | ~500 tokens for LSP reference lookup |

Install the LSP plugin for your language:

```bash
# official channel
/plugin install <language>-lsp@claude-plugins-official

# community marketplace (more languages, including Java)
claude plugin marketplace add Piebald-AI/claude-code-lsps
/plugin install java@Piebald-AI/claude-code-lsps
/plugin install python@Piebald-AI/claude-code-lsps
```

**Java setup (jdtls required):**

```bash
# Arch
sudo pacman -S jdtls

# SDKMAN
sdk install jdtls

# verify (requires Java 17+)
which jdtls && java -version
```

Enable in `~/.claude/settings.json`:

```json
{
  "env": {
    "ENABLE_LSP_TOOL": "1"
  }
}
```

Add to `CLAUDE.md` to enforce usage:

```markdown
## LSP usage
- Before modifying any unfamiliar function: use goToDefinition first
- Before any rename or refactor: use findReferences for impact analysis
- After any edit: verify diagnostics are clean before continuing
```

---

## Context budget discipline

Monitor with `/context` during sessions with multiple servers. Practical limits:

- Under 10k tokens of MCP + plugin definitions → negligible impact
- 10–20k tokens → monitor context usage closely
- Over 20k tokens → you're sacrificing working context for tool availability

If you have many servers and plugins, `lazyLoad: true` is the correct default for all of them. Only disable lazy loading for tools you use in literally every session.

Lower the MCP output token cap if large tool responses are crowding out your working context:

```bash
export CLAUDE_MCP_MAX_OUTPUT_TOKENS=15000   # default 25000
```

---

## Tips

- [ ] Store `GITHUB_TOKEN`, `DATABASE_URL`, etc. in your shell profile (`~/.zshrc`) , MCP env vars read from there
- [ ] Commit `.mcp.json` to the project , teammates get the same server configuration
- [ ] `lazyLoad: true` on all servers and plugins until you confirm a session needs them eager-loaded
- [ ] Use read-only DB credentials in MCP unless explicitly doing write operations
- [ ] Add a `PreToolUse` hook on DB MCP tools to log queries in sensitive environments
- [ ] `/mcp` inside a session shows which servers are connected and their tool inventory
- [ ] For GitHub: `gh` CLI covers 90% of common operations at zero context cost , use MCP for complex multi-step flows
- [ ] Playwright MCP is high-value for UI work , Claude verifying its own UI changes closes a feedback loop text-only tools can't close
- [ ] Audit `~/.claude/plugins/` periodically , unused plugins burn context budget in every session

---

## What's next

MCP, plugins, and LSP extend what Claude can perceive and act on. L7 covers **skills, agents, and custom commands** , how to package reusable workflows and delegate complex tasks to specialized sub-agents.
