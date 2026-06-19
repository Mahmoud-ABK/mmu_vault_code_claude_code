# L5 , Hooks

**Where you are:** You have a fully configured project. You've noticed that CLAUDE.md rules get ignored sometimes, and you want certain things to happen mechanically every time.  
**Goal:** Understand the hook system, wire up the core practical hooks, know when hooks are the right tool and when they're overkill.

---

## The core distinction

**CLAUDE.md** , advisory. Claude interprets it and applies judgment. Good for conventions, style guidelines, team preferences. Compliance is probabilistic.

**Hooks** , deterministic. They execute outside Claude's reasoning, the same way every time. Good for enforcement, formatting, safety guardrails. Compliance is unconditional , hooks run even in `--dangerously-skip-permissions` mode.

If you catch yourself writing "ALWAYS do X" or "NEVER do Y" in CLAUDE.md, that belongs in a hook instead.

---
[claude code hooks](https://code.claude.com/docs/en/hooks)

the claude code documentation is rich and explanatory enough no need for me to re-explain 

---
## Tips

- [ ] Make scripts executable before deploying: `chmod +x .claude/hooks/*.sh`
- [ ] Test every hook manually with piped JSON before deploying it
- [ ] `exit 1` does not block , use `exit 2` or JSON `permissionDecision: "deny"` to block a tool call
- [ ] Write block reasons to stderr on the `exit 2` path, or to `permissionDecisionReason` on the JSON path , Claude reads both to re-plan
- [ ] The `stop_hook_active` check is mandatory in any Stop hook that returns `decision: "block"`
- [ ] Hooks in `.claude/settings.json` are committed and apply to the whole team. Personal hooks go in `.claude/settings.local.json`
- [ ] Set `timeout` on every command hook , a hanging script hangs Claude's entire turn
- [ ] Prefer `command` for deterministic checks, `prompt` for context re-injection, `agent` only when validation needs multi-file reasoning
- [ ] HTTP hooks cannot block via status codes , block by returning a `2xx` response with `permissionDecision: "deny"` in the JSON body
- [ ] Use the `if` field on handlers to pre-filter before spawning the script. `"if": "Bash(rm *)"` avoids spawning a process on every Bash call

---

## What's next

Hooks give you mechanical enforcement over Claude's actions inside a session. L6 extends Claude's capabilities outward , **MCP servers** connect Claude to external services, databases, and APIs, giving it new tools beyond the built-in set.