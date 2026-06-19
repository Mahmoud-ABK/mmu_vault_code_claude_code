# L7 , Skills, Agents & Custom Commands

**Where you are:** Your project is configured — context, permissions, hooks, MCP. But you keep re-typing the same multi-step prompts, and long subtasks crowd your main session context.  
**Goal:** Package repeatable workflows into commands and skills, and delegate large or parallel subtasks to isolated sub-agents.

---

## The three primitives

```
Custom Commands  -> you decide when to invoke  (/command)
Skills           -> Claude decides when to invoke  (natural language trigger)
Agents           -> Claude spawns a sub-process with isolated context
```

|Primitive|Invocation|Context|Use for|
|---|---|---|---|
|Custom command|You type `/command`|Shares main|Repeatable workflows you control|
|Skill|Claude detects it|Shares main|Domain behavior Claude applies auto|
|Agent|Claude calls Task()|Isolated window|Long or parallel subtasks|

---

## Custom Commands

Location: `.claude/commands/` (project) or `~/.claude/commands/` (personal). Filename becomes the command name: `do-thing.md` -> `/do-thing`.

### Template

```markdown
---
description: What this command does
allowed-tools: Read, Edit, Bash(command *)
argument-hint: [optional-arg]
context: fork
---

Do the following with $ARGUMENTS:

1. Step one
2. Step two
3. Step three

Output format: describe what you expect back.
```

### Frontmatter options

| Key             | Purpose                                      |
| --------------- | -------------------------------------------- |
| `description`   | Shown in /help                               |
| `allowed-tools` | Restrict which tools this command can use    |
| `argument-hint` | Shown in autocomplete                        |
| `context: fork` | Isolates command into its own context window |
| `model`         | Override model for this command              |

Use `context: fork` for commands that do heavy file reading to avoid polluting the main session context.

---

## Skills

Skills are loaded automatically when Claude detects a trigger phrase. You do not invoke them manually.

Location: `.claude/skills/<skill-name>/SKILL.md`

### Template

```markdown
---
name: skill-name
description: >
  One or two sentences describing what this skill does and when Claude
  should load it. List the natural language phrases that should trigger it.
  The more specific the triggers, the more reliable the activation.
---

# Skill Name

## When to apply
Describe the conditions that should activate this skill.

## Behavior
Describe exactly what Claude must do differently when this skill is active.
Use numbered steps for ordered processes, bullets for rules.

## Constraints
- What this skill must never do
- Scope limits
```

**Invocation:**

```
# explicit , inject the skill file into context
@.claude/skills/pre-commit.md

# natural language , Claude detects the trigger
> run the pre-commit skill
> apply the TDD cycle skill to this feature
```

Reference skills in `CLAUDE.md` to increase activation reliability:

```markdown
## Skills
- skill-name: .claude/skills/skill-name/SKILL.md - apply when [condition]
```

---

## Agents

An agent is a Claude instance spawned by the main agent via `Task()`. It has an isolated context window, its own system prompt, and a restricted tool set. It does not inherit the main session context.

Location: `.claude/agents/<agent-name>.md`

### Template

```markdown
---
name: agent-name
description: >
  One sentence on what this agent does. Used by the main Claude to decide
  when to spawn it.
tools: Read, Edit, Bash(allowed-command *)
memory: project
---

# Agent Name

## Purpose
One sentence.

## Inputs
What the agent receives when spawned.

## Process
1. Step one
2. Step two
3. Step three

## Output
What the agent returns to the main Claude. Be specific about format.

## Constraints
- What this agent may not do
- What it may not modify
- Scope limits
```

### When to use agents vs inline

Use an agent when:

- The subtask is long enough to consume significant context if done inline
- The subtask is specialized and self-contained
- You want parallel execution across independent subtasks

Do not use an agent when:

- The task is short
- The task needs to share state tightly with the main session

### Parallel invocation

Prompt the main Claude:

```
Use the Task tool to run <agent-name> in parallel on these targets:
- target-one
- target-two
- target-three

Each agent is independent. Do not wait for one to finish before starting the next.
```

### Orchestration pattern

```
orchestrator reads shared context once
    spawns agent-a on subtask-1   (isolated context)
    spawns agent-b on subtask-2   (isolated context)
    spawns agent-c on subtask-3   (isolated context)
orchestrator collects results and synthesizes
```

Each spawned instance reads CLAUDE.md as its base. The orchestrator coordinates; agents do not communicate with each other.

### Master-clone pattern

Instead of hand-coding agent orchestration, put all shared context in CLAUDE.md and prompt the main agent to spawn clones of itself:

```
You are the orchestrator. Implement the order fulfillment flow end to end.

Spawn sub-agents for:
1. Domain model changes (Order, OrderItem, FulfillmentEvent)
2. Application use case (FulfillOrder, port interfaces)
3. Infrastructure adapter (connecting to the external fulfillment API)
4. Integration tests

Each sub-agent must read CLAUDE.md before starting.
Run them in the order that respects the dependency graph.
```

The main agent manages orchestration dynamically. You don't write the coordination logic , you describe the decomposition.

---

---

## Best Use Cases

### Custom Commands

High-signal community patterns - these are the ones that consistently appear across public repos and discussions:

|Command|What it does|
|---|---|
|`/commit`|Sanity-checks staged changes for TODOs, debug code, and unintended changes, then generates a commit message|
|`/review`|Reviews a diff for logic errors, security issues, and coverage gaps before a PR|
|`/gate`|Runs the full quality pipeline (lint, typecheck, tests) and stops on first failure|
|`/fix`|Takes an issue number or error message as `$ARGUMENTS` and implements the fix|
|`/write-tests`|Generates unit and integration tests for a given file or function|
|`/explain`|Deep-explains a file or function: what it does, how, callers, edge cases|

The pattern behind all of these: a workflow you repeat verbatim, with optional arguments for scope. If you are re-typing the same prompt structure more than three times, it belongs in a command.

### Skills

Skills work best for enforcing a consistent approach Claude would otherwise vary on. Community consensus on the highest-value ones:

|Skill|Trigger condition|
|---|---|
|TDD cycle|Any new feature implementation - forces test-first|
|API design|Designing or reviewing any endpoint|
|Commit convention|Any commit or changelog work|
|Error handling|Writing any I/O, network, or async code|
|Documentation sync|Writing or modifying any doc file (doc.md, CLAUDE.md etc.)|

The pattern: behavior Claude should apply consistently without you asking, scoped to a clearly detectable condition.

### Agents

The community separates agent use cases into two modes:

**Parallel - independent tasks with no shared state needed:**

|Agent|What it does in parallel|
|---|---|
|Code reviewer|Reviews separate modules simultaneously, reports by severity|
|Codebase explorer|Searches the codebase multiple ways at once, keeps results out of main context|
|Security auditor|Audits backend, frontend, and infra independently|
|Research worker|Investigates multiple topics or competitors at the same time|
|Doc syncer|Syncs documentation across separate modules simultaneously|

**Sequential - assembly-line workflows where output feeds the next agent:**

|Pipeline|Agent chain|
|---|---|
|Feature from ticket to code review|PM agent -> Engineer agent -> Reviewer agent|
|Bug from report to verified fix|Debugger agent -> Fixer agent -> Test writer|
|API from spec to implementation|Designer agent -> Builder agent -> Reviewer|

The key distinction the community agrees on: use parallel agents when workers are independent, use sequential agents when each step consumes the previous output. Do not use agents when the task is short enough to run inline without filling the main context - the token overhead is roughly 4-7x a single thread.

---

## Tips

- Create a command for any prompt you type more than three times.
- Use `~/.claude/commands/` for personal commands, `.claude/commands/` for team.
- Keep skills focused: one skill per workflow.
- Always scope agent `tools:` to the minimum needed.
- More than 10 custom commands is a sign of over-engineering.
- Set `CLAUDE_CODE_SUBAGENT_MODEL` to a lighter model for agents doing focused tasks. Run the orchestrator on Opus, agents on Sonnet. Cuts cost significantly.
- Subagent-heavy sessions use roughly 4-7x the tokens of a single thread. Only parallelize when the time saving justifies it.

---

## What's next

Skills, agents, and commands handle workflow reuse and task delegation inside a single session. L8 goes further , **multi-agent execution across git worktrees, headless mode for CI, scheduling, and power prompting patterns** that scale to fully autonomous operation.