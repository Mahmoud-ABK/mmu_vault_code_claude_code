# L8 , Headless, Scheduling and Power Patterns

**Where you are:** Your single-session toolkit is complete — context, permissions, hooks, MCP, skills, agents. Now you want Claude working beyond one interactive session: in parallel, in CI, and on a schedule.  
**Goal:** Run multiple agents in isolated worktrees, drive Claude headlessly for automation, schedule recurring work, and apply the prompting patterns that make autonomous runs reliable.

---

## Git worktrees for parallel agents

The canonical pattern for running multiple agents on the same repo without file conflicts. Each agent gets an isolated working directory on its own branch.

```bash
# create worktrees
git worktree add ../feature-a feature/feature-a
git worktree add ../feature-b feature/feature-b

# run an agent in each
cd ../feature-a && claude
cd ../feature-b && claude

# clean up after merging
git worktree remove ../feature-a
git worktree remove ../feature-b
```

Each session has its own context and reads the CLAUDE.md from its worktree root. Agents have no awareness of each other. Coordination happens at the git level, not inside Claude.

---

## Headless mode

Run Claude Code unattended with no REPL.

```bash
# basic
claude -p "your task here"

# restrict tools
claude -p "your task" --allowedTools "Bash(your-command *)"

# structured output
claude -p "your task" --output-format json

# cost ceiling
claude -p "your task" --max-budget-usd 3.00

# bare mode - no hooks, LSP, or plugins. Fastest, for CI
claude -p "your task" --bare

# pipe input
cat file.log | claude -p "summarize the errors in this log"
```

Use `--bare` in CI where you do not need the full local stack. Use `--output-format json` when a downstream script consumes the result.

---

## CI template

```yaml
name: Claude Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "
            Review git diff origin/main...HEAD.
            Check: logic errors, missing error handling, security issues,
            architecture violations per CLAUDE.md, missing test coverage.
            Output JSON only:
            {
              findings: [{file, line, severity, description, suggestion}],
              verdict: 'ready|needs-work|blocked',
              summary: string
            }
          " --output-format json --bare > review.json

      - name: Post comment
        uses: actions/github-script@v7
        with:
          script: |
            const review = JSON.parse(require('fs').readFileSync('review.json', 'utf8'));
            const body = review.findings.map(f =>
              `**${f.severity.toUpperCase()}** \`${f.file}:${f.line}\`\n${f.description}\n> ${f.suggestion}`
            ).join('\n\n');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude Review\n\n**Verdict: ${review.verdict}**\n\n${body}\n\n${review.summary}`
            });
```

---

## Scheduling

### /loop - recurring prompt in session

Runs a prompt on a timer while the session is open.

```bash
/loop 5m your task here
```

Intervals: `s`, `m`, `h`, `d`. Stops when the session closes.

Good for: polling CI, monitoring a deploy, watching test results.

### /schedule - cloud-scheduled prompt

Runs on Anthropic's infrastructure. Works when your machine is off.

```bash
# cron syntax
/schedule 0 9 * * 1-5 "your recurring task"
```

Results are delivered back to your session or via your configured notification channel.

### Ctrl+B - background command

Moves a running command to the background, freeing the session.

```bash
Ctrl+B
your-long-running-command
# session is now free for other work
```

---

## Auto-memory

Claude saves useful context across sessions automatically. Manage it with:

```bash
/memory                         # view current memory
/memory add "note about your project"
```

Auto-memory persists across `/clear` and session restarts. Use it to store things Claude discovered mid-session that would otherwise be lost: entry points, known bugs to avoid, deploy constraints, non-obvious conventions.

---

## Output style

Define how Claude formats its responses. Location: `~/.claude/output-styles/<name>.md`

```markdown
---
name: engineering
description: Dense, technical, no filler. For engineering tasks.
---

- No preamble. Start with the answer.
- Code before explanation.
- Use tables for comparisons.
- If showing multiple options, rank them , don't just list.
- Never use filler phrases ("Great question", "Certainly", "Of course").
- Omit explanation of things not asked about.
```

Activate: `/config` → Output Style → your style name.

---

## Status line

A script that runs after every Claude turn and displays at the bottom of the terminal.

```bash
#!/bin/bash
# ~/.claude/statusline.sh
BRANCH=$(git branch --show-current 2>/dev/null || echo "no-git")
DIRTY=$(git status --short 2>/dev/null | wc -l | tr -d ' ')
TESTS=$(cat .claude/last-test-result 2>/dev/null || echo "unknown")

echo "branch:$BRANCH  uncommitted:$DIRTY  tests:$TESTS"
```

Register in `~/.claude/settings.json`:

```json
{
  "statusLine": "~/.claude/statusline.sh"
}
```

Configure interactively: `/statusline`

---

## Neovim integration

### Option 1 , Terminal split (zero config)

```lua
-- ~/.config/nvim/lua/claude.lua
vim.keymap.set('n', '<leader>cc', function()
  vim.cmd('vsplit | terminal claude --dangerously-skip-permissions')
  vim.cmd('startinsert')
end, { desc = 'Claude Code vertical split' })
```

### Option 2 , `claudecode.nvim` (recommended)

Implements the same MCP protocol as the VS Code extension. Claude knows your current buffer, cursor position, and open files.

```lua
-- lazy.nvim
{
  "coder/claudecode.nvim",
  dependencies = { "folke/snacks.nvim" },
  config = true,
  keys = {
    { "<leader>cc", "<cmd>ClaudeCode<cr>",      desc = "Toggle Claude Code" },
    { "<leader>cf", "<cmd>ClaudeCodeFocus<cr>", desc = "Focus Claude Code" },
    { "<leader>cs", "<cmd>ClaudeCodeSend<cr>",  mode = "v", desc = "Send selection" },
    { "<leader>cd", "<cmd>ClaudeCodeDiff<cr>",  desc = "Review diff" },
  },
}
```

### Option 3 , `claude-preview.nvim` (diff review before apply)

Uses `PreToolUse`/`PostToolUse` hooks to intercept edits and show a side-by-side diff in Neovim before Claude applies them.

```lua
{
  "Cannon07/claude-preview.nvim",
  config = function()
    require("claude-preview").setup({ neo_tree = { enabled = true } })
  end
}
```

Install hooks from inside Neovim: `:ClaudePreviewInstall`

---

## Power prompting patterns

**Self-verifying loop**

```
Implement X, run the tests, fix any failures.
Do not stop until the suite is green. Do not ask for input.
```

**Plan before execute**

```
Think through the best approach before writing any code.
Show me the plan. I will approve before you start.
```

**Interview before executing**

```
Ask me clarifying questions before you start.
Do not write any code until you've confirmed your understanding of the requirements.
```

**Adversarial review**

```
Find every edge case, race condition, and assumption in this implementation.
Grill me on the design. Do not let me proceed until you are satisfied the logic is correct.
```

**Cross-model adversarial**

```
Critique this plan as a staff engineer who is skeptical of the approach.
Look for hidden complexity, premature abstraction, and missing failure modes.
```

**Scope bounding**

```
Fix only the specific issue in [function]. Do not refactor anything else.
Make the minimal change that resolves it.
```

**Feedback loop**

```
Implement this, then run ./gradlew test.
Fix any failures. Do not stop until the suite is green.
Report the final test output when done.
```

**Recovery after poor result**

```
Scrap that. Knowing everything you know now, implement the clean solution.
The constraint I did not mention earlier: [missing constraint].
```

---

## The full stack

```
CLAUDE.md       ->  Claude knows the project
settings.json   ->  Claude is restricted
hooks           ->  Claude is mechanically constrained
MCP servers     ->  Claude has extended tools
skills          ->  Claude applies domain behavior automatically
agents          ->  Complex tasks are isolated and parallelized
LSP             ->  Claude understands code semantically
auto-memory     ->  Claude retains context across sessions
```

Each layer addresses a different failure mode. Together they move Claude from a capable assistant to an agent that operates within your constraints, knows your project, and improves over time.

---

## Operational checklist

- Start every new task with `/clear`
- Check `/context` on long sessions
- Use `/compact retain [specific thing]` not bare `/compact`
- Give Claude a feedback loop in the prompt (tests, linter) rather than just saying "implement"
- Review auth, payments, and data mutations regardless of how good the output looks
- `Esc+Esc` before trying an approach you are not confident in
- `/memory add` anything useful Claude discovered mid-session
- `/cost` at session end to build intuition for which tasks are expensive

---

## Quick reference

```
/clear                              wipe context
/compact [instructions]             compress context
/context                            show context usage %
/cost                               show session token cost
/model                              switch model and effort
/memory                             manage cross-session memory
/hooks                              manage hooks
/mcp                                show MCP server status
/plugin                             install and manage plugins
/statusline                         configure terminal status line
/loop Nm <task>                     recurring prompt (session-scoped)
/schedule <cron> <task>             cloud-scheduled prompt
/rewind                             checkpoint browser
/reload                             reload config and hooks without restart
/config                             output style and preferences
/install-github-app                 automated PR review
/help                               list all commands

Esc                                 stop Claude, keep context
Esc+Esc                             open checkpoints
Ctrl+B                              background a command
Ctrl+O                              toggle hook output visibility
Shift+Tab                           cycle permission modes

claude -c                           resume last session
claude -r                           session picker
claude -p "task"                    headless single task
claude -p "task" --bare             headless, no hooks or plugins
claude -p "task" --output-format json
claude -p "task" --max-budget-usd N
claude --model claude-opus-4-7 --effort max

CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=60  # compact at 60% instead of 95%
CLAUDE_MCP_MAX_OUTPUT_TOKENS=15000  # lower MCP output cap
CLAUDE_DEBUG=1                      # debug hook execution
```

---

## What's next

L8 covers the execution patterns. L9 closes the loop , **context cost of the harness**: what each component you've configured costs in tokens, how global vs project scope amplifies that cost, and how to tune the system to stay within budget.