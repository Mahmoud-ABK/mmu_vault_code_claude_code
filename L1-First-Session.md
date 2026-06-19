# L1 , First Session & Basic Usage

**Where you are:** Claude Code is installed. You've never opened it, or you've opened it but have no idea what you're doing.  
**Goal:** Launch a session, understand the interaction model, learn the essential commands.

---

## The interaction model

Claude Code is **not a chatbot in a terminal**. It's an agent. The difference in practice:

- A chatbot gives you text you paste somewhere
- Claude Code reads your files directly, writes changes, runs commands, and iterates on its own

You describe what you want. Claude figures out how to do it, executes it, reads the result, and continues until the task is done. Your role is to steer, not drive.

The paradigm shift from inline AI tools (Copilot, Cursor completions) is architectural:

|Copilot / inline tools|Claude Code|
|---|---|
|Suggestion at cursor|Autonomous multi-step execution|
|You apply the diff|Claude applies diffs, runs tests, reads failures, fixes|
|Stateless per suggestion|Stateful across a full session|
|Reads one file|Reads the whole repo|
|You drive|You steer|

The practical consequence: your job becomes **specification and review**, not implementation. The limiting factor is no longer typing speed , it is the quality of your context, constraints, and verification.

---

## Launching

```bash
# basic launch , runs in the current directory
claude

# resume the most recent session
claude -c

# session picker (see all past sessions)
claude -r

# non-interactive: run one task and exit
claude -p "describe what this file does" < README.md
```

> **Context:** Claude Code uses the directory you launch from as its working directory. It can read and edit any file it can access from there. Always `cd` into your project before launching.

---

## Your first session

When you open Claude Code you land in the REPL. Type naturally.

```
> What does this project do?
> Explain the main entry point
> Add a null check to the getUser function in src/user.service.ts
> Write a test for the processPayment method
```

Claude will:

1. Read the relevant files
2. Plan what to do
3. Execute (edit files, run commands)
4. Report back

You don't give it file paths unless you want to be specific. It finds them.

---

## Essential slash commands (session control)

These are typed inside an active session starting with `/`.

|Command|Purpose|
|---|---|
|`/help`|List all available commands|
|`/clear`|Wipe the conversation , start fresh. Use this between tasks.|
|`/compact`|Compress history to free up context while keeping the thread|
|`/exit`|End the session|
|`/cost`|Show token usage for the current session|
|`/context`|Show how much context is used (%)|

Pass an argument to `/compact` to tell Claude what to prioritize in the summary:

```
/compact retain the auth module changes and the list of modified files
```

---

## Essential keyboard shortcuts

|Shortcut|Effect|
|---|---|
|`Esc`|Stop Claude mid-action. Work done so far is preserved.|
|`Esc + Esc`|If the prompt has text: clears the draft and saves it to history. If the prompt is empty: opens the rewind menu to restore a prior state.|
|`Ctrl+C`|Interrupts a running operation. If nothing is running, the first press clears the prompt input; a second press exits Claude Code.|
|`Ctrl+D`|Exit session (EOF signal)|
|`Shift+Tab`|Cycle permission modes. The default cycle is `default` → `acceptEdits` → `plan`. Additional modes appear based on your account and launch flags , see L3.|

---

## Referencing files

You can point Claude to specific files using `@`:

```
> Look at @src/payment/PaymentService.java and explain the retry logic
> What's different between @src/old/UserRepo.java and @src/new/UserRepo.java?
```

- `@path/to/file.java` , injects the full file content into context immediately
- `@path/to/dir/` , injects a directory listing (not file contents)

The difference from just asking Claude to read the file: `@` is explicit and immediate. Claude doesn't need to decide to read it first.

---

## Running shell commands inside a session

```bash
# run directly, output goes into Claude's context
!git status
!npm run test
!./gradlew build
```

Use `!` when you want to give Claude the output of a command without asking it to run the command itself. It's faster and uses fewer tokens than asking Claude to run it conversationally.

---

## The permission prompt (default behavior)

By default, Claude will ask before doing things:

```
Claude wants to run: npm run test
Allow? (y/n/always)
```

- `y` , allow once
- `n` , deny
- `always` , allow this command for the rest of the session

This gets in the way during long tasks. You'll configure this properly in L3. For now, know it exists and why.

---

## Tips

- [ ] Always launch Claude from inside your project directory , it uses `cwd` as its root
- [ ] `/clear` between tasks , context from a previous task is noise, not help
- [ ] Ask questions the way you'd ask a senior engineer: "Why does X call Y here instead of Z?"
- [ ] Use `Esc` to interrupt and redirect, not `Ctrl+C` , `Esc` stops the current action and keeps the session alive
- [ ] `/cost` after a session gives you a sense of token burn per task type
- [ ] `Shift+Tab` once enters `acceptEdits` mode , Claude stops asking permission for file edits and common filesystem commands

---

## What's next

L1 covers raw interaction. The next problem you'll hit is **Claude losing track of what it was doing** in a long session, or **not knowing your project's conventions**. That's L2 , context and CLAUDE.md.