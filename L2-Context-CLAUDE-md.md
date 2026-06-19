# L2 , Context & CLAUDE.md

**Where you are:** You can run sessions. You've noticed Claude sometimes loses track of things, forgets conventions, or asks you to repeat information.  
**Goal:** Understand the context window as a constraint, learn to manage it, and give Claude persistent project knowledge via CLAUDE.md.

---

## The context window , the governing constraint

Everything Claude knows in a session exists in its **context window** , a fixed-size buffer measured in tokens. Every message you send, every file it reads, every tool result, every prior exchange consumes space in that buffer.

The critical insight: **Claude doesn't fail when the context fills , it degrades silently.** Reasoning gets shallower, it starts ignoring earlier constraints, it repeats mistakes you corrected before. You won't see an error , you'll see worse output.

```
0%─────────────────────────────────────────────100%
│              │              │                 │
OK          Check usage    Compact          Clear / broken
            with /context  if mid-task     if task is done
```

There are no fixed system-defined thresholds. The practical rule: run `/context` periodically during long sessions, compact when you're mid-task and usage is climbing, clear when switching to a new task entirely.

---

## Managing context , the four operations

### `/clear` , full reset

Wipes the entire conversation. File edits made during the session **persist** , only the memory of the conversation is gone.

Use when: switching to a different task, starting fresh, or context is high and the prior thread isn't needed.

```
/clear
```

### `/compact` , compress and continue

Replaces the conversation history with a compressed summary. Claude keeps a thread of what was discussed but in fewer tokens.

Use when: mid-task, context is climbing, and you still need continuity.

```
/compact
/compact retain the auth module changes and the list of modified files
```

Pass an argument telling Claude what to prioritize in the summary. Guided compaction produces significantly better summaries than bare `/compact`.

### `Esc+Esc` (rewind) , checkpoint restore

Before Claude edits any file, it snapshots the current contents. `Esc+Esc` behavior depends on the state of your prompt:

- **If the prompt has text:** clears the draft input and saves it to history so `Up` recalls it.
- **If the prompt is empty:** opens the rewind menu, letting you restore code and conversation from a previous checkpoint.

Use the rewind menu when Claude went down a wrong path and you want to roll back without losing other work.

> **Limitation:** Checkpoints only cover file edits. Actions that already ran against remote systems (migrations, deploys, installs) are not captured.

### `claude -c` / `claude -r` , session resume

```bash
claude -c          # resume most recent session
claude -r          # session picker , choose from all past sessions
```

Sessions persist across terminal closes. You don't lose context just because you closed the window.

---

## CLAUDE.md , persistent project memory

CLAUDE.md is a markdown file that Claude reads **at the start of every session**. It's how you give Claude persistent context it cannot infer from code alone.

Without it, you repeat yourself every session. With it, Claude knows your conventions, build commands, architecture rules, and known pitfalls from session one.

### Where it lives

```
your-project/CLAUDE.md       # project-level , loaded in this project
your-project/src/CLAUDE.md   # subdirectory , loaded lazily when Claude reads files there
```

> Global scope (`~/.claude/CLAUDE.md`) applies across all your projects , covered in [[L4-Scoping]].

### What goes in it

Rule of thumb: **document what Claude gets wrong, not what it should already know.**

```markdown
# Project: OrderService

## Build & test commands
- Build: `./gradlew build`
- Test all: `./gradlew test`
- Test one class: `./gradlew test --tests "com.example.*OrderService*"`
- Lint: `./gradlew checkstyleMain`

## Architecture
- Hexagonal: domain → application → infra
- Domain must never import from infra , enforced by hooks
- Ports (interfaces) live in domain/port/

## Code conventions
- Constructor injection only , no @Autowired on fields
- No public fields on domain entities
- Prefer Result<T,E> over throwing exceptions across layer boundaries

## Common mistakes to avoid
- Do not call repository.save() inside domain entities
- Never use Optional.get() without isPresent() check
- PaymentGatewayPort has a 30s timeout , account for it in integration tests

## Test conventions
- Unit tests: pure JUnit 5 + Mockito, no Spring context
- Integration tests: suffix IT, @SpringBootTest
- Never mock the class under test
```

### Generating a starter

```bash
# inside a session, Claude reads the project and generates a CLAUDE.md
/init
```

`/init` analyzes your codebase , build system, test framework, code patterns , and produces a draft. Always trim it heavily after generation. Remove boilerplate, keep only what Claude actually needs.

### Size discipline

Keep CLAUDE.md concise. Claude's attention distributes across a large input, so a bloated file produces diminishing returns , instructions buried at the end get less weight than instructions near the top. The practical target is under 150 lines for a project-level file.

If it's growing, split into `.claude/rules/` fragments (covered in L4). Rules files load lazily when Claude reads files in matching paths, so their cost is paid only when relevant.

Don't embed documentation files with `@`. Reference the path as a string instead: `"For FooBarError, see docs/errors.md"`.

---

## `@` , explicit context injection

The `@` syntax injects content directly into the current message, bypassing Claude's read-then-decide flow.

```
> Review @src/payment/PaymentController.java for error handling gaps
> Compare @src/v1/UserRepo.java and @src/v2/UserRepo.java
> What does @src/domain/ contain?
```

- `@path/to/file` , full file content in this message
- `@path/to/dir/` , directory listing (filenames only, not content)

Use `@` when you want Claude to have exact content in context right now, not when it decides it needs to read the file.

---

## Compaction instruction in CLAUDE.md

Add this to your project CLAUDE.md so Claude knows what to preserve when compacting:

```markdown
## On compaction
When compacting, always preserve:
- Current task description
- List of files modified this session
- Any constraints or blocked approaches already identified
- Current test pass/fail status
```

---

## Tips

- [ ] Add `/context` to your mental checklist on long tasks , check it periodically
- [ ] Run `/init` in any new project immediately, then trim the output
- [ ] Always pass an argument to `/compact` , guided compaction is significantly better than bare compaction
- [ ] Put build and test commands in CLAUDE.md so Claude can run them without asking you
- [ ] Subdirectory CLAUDE.md files are lazy-loaded , useful for monorepos where different packages have different conventions
- [ ] `Esc+Esc` with an empty prompt opens the rewind menu , use it before trying an approach you're not confident in
- [ ] CLAUDE.md is advisory. Claude reads and applies it, but adherence degrades as the context fills. For rules that must always hold, use hooks (L5).

---

## What's next

You now know how to manage context and give Claude persistent project knowledge. The next friction point: **Claude asking permission for every action**, and **not knowing what it's allowed to do**. That's the permissions system , L3.