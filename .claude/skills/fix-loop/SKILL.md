---
name: fix-loop
description: Generic iterative review-fix cycle. Takes --reviewer and --fixer agent names as parameters. Runs reviewer, fixes critical findings, verifies, and repeats until clean or max iterations reached.
user_invocable: false
---

# Fix Loop

Generic autonomous review-fix cycle that iterates until all critical issues are resolved.

## Input

```
--reviewer <agent-name> --fixer <agent-name> [file paths or directory]
```

- `--reviewer` (required): Name of the reviewer agent to spawn
- `--fixer` (required): Name of the fixer agent to spawn
- Remaining arguments: file path(s), directory, or omit to check everything

## Workflow

```
🔍 REVIEW (reviewer agent) → 🔧 FIX (fixer agent) → 🔁 LOOP (max 5) → 💾 COMMIT?
```

### Agents

| Step   | Agent              | Purpose                    |
|--------|--------------------|----------------------------|
| REVIEW | `<reviewer agent>` | Find issues (read-only)    |
| FIX    | `<fixer agent>`    | Apply fixes + verify       |

### Constraints

- **Max iterations:** 5
- **Fix target:** Critical (🔴) findings only
- **Scope narrows** each iteration — only re-review files the fixer changed
- **Test gate** — fixer verifies tests pass before returning

---

## Execution

### Step 1: Parse Arguments

Extract `--reviewer` and `--fixer` agent names from the arguments. Remaining arguments are the file scope.

If either `--reviewer` or `--fixer` is missing, ask the user to provide them.

### Step 2: Determine Scope

If no files specified, run `git diff --name-only HEAD~3` to find recently changed files.

### Step 3: Review-Fix Loop

Set `iteration = 1` and `scope = <initial files>`.

**LOOP** while `iteration <= 5`:

1. **🔍 REVIEW (iteration N)** — Announce: `🔍 REVIEW iteration N/5`
    - Spawn the reviewer agent against `scope`
    - Receive the findings report

2. **TRIAGE** — Extract only 🔴 **Critical** findings from the report
    - If **zero** critical findings → break out the loop, we are done.
    - List the critical findings for visibility

3. **🔧 FIX (iteration N)** — Announce: `🔧 FIX iteration N/5 — addressing N critical issue(s)`
    - Spawn the fixer agent, passing it:
        - The list of 🔴 Critical findings (with file paths and line numbers)
        - The review context
    - Receive the fix report (fixed, unfixable, files modified, test status)

4. **NARROW SCOPE** — Set `scope` to only the files listed in the fixer's "Files Modified" output
    - If no files were modified (all unfixable) → break to Step 4

5. **INCREMENT** — `iteration += 1`

**END LOOP**

### Step 4: Final Report

Announce: `✅ FIX LOOP COMPLETE`

Aggregate findings from all iterations:

```markdown
# Fix Loop Report

## Iterations: N/5

## Resolved (🔴 Critical)
- [file:line] [issue] — fixed in iteration N

## Remaining (🔴 Critical)
- [file:line] [issue] — reason not fixed

## Noted (🟡 Warning / ℹ️ Suggestion)
- [file:line] [issue] — from iteration N (not actioned)

## Test Status: ✅ All passing / ⚠️ See notes
```

### Step 5: Commit

Ask the user: **Commit these changes?**

If yes → invoke the `commit-helper` agent.

---

## Announce Phase Transitions

```
🔍 REVIEW iteration 1/5 — reviewing N file(s)
🔧 FIX iteration 1/5 — addressing N critical issue(s) [fixer agent]
🔁 LOOP — N critical issue(s) remain, narrowing scope to N file(s)
✅ FIX LOOP COMPLETE — 0 critical issues after N iteration(s)
```
