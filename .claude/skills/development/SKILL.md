---
name: Development
description: Implement features using strict test-driven development. Use when writing new code, implementing features, fixing bugs, or when TDD is requested. Enforces RED-GREEN-REFACTOR cycle.
---

# Development (TDD)

## Core Rules (Non-Negotiable)

1. **NEVER write production code without a failing test first**
2. One behaviour per test
3. Write minimum code to make the test pass
4. Run tests and verify green state before proceeding
5. Keep project documentation up to date
6. Use the vault skill

## The TDD Cycle

```
📋 TASK   → Create/update task file (vault skill)
🔴 RED    → Write failing test. You must run the test to verify that it fails before continuing.
🟢 GREEN  → Write minimum code to pass using a language specific skill if one is present at the project level of globally.
✅ VERIFY → Run all tests, confirm passing
💾 COMMIT → Save working state (use commit-helper)
👀 REVIEW → Use code-reviewer skill
⚠️ FIX    → Address review issues
🔵 REFACTOR → Improve code (use a language specific, project level of global refactoring skill if one is presewnt)
💾 COMMIT → Save refactored state
📝 LOG    → Update task file with decisions/outcomes
```

See `references/tdd-workflow.md` for detailed step-by-step instructions.

## Quick Reference

| Phase | Announce | Action |
|-------|----------|--------|
| RED | `🔴 RED → Writing failing test for [behaviour]` | Write ONE test, verify it fails |
| GREEN | `🟢 GREEN → Writing minimum code` | Pass current test only |
| VERIFY | `✅ All tests passing` | Run ALL tests |
| REFACTOR | `🔵 REFACTOR → [improvement]` | Small changes, tests stay green |

## Test Structure (AAA Pattern)

```
-- Arrange: set up test data
testData = createTestData()

-- Act: execute the code
result = functionUnderTest(testData)

-- Assert: verify result
assertEqual(result, expectedValue)
```

Language-specific test syntax provided by language skills (scala-developer, ruby-developer, etc.)

## Handling Scenarios

| Scenario | Approach |
|----------|----------|
| New feature | Start with simplest criterion, build complexity gradually |
| Bug fix | Write test reproducing bug first, then fix |
| Refactoring | Add tests for current behaviour first, then refactor |
| Delegating implementation to a sub-agent | Write the behaviour-pinning tests yourself first (the agent implements to green — prevents a false green from agent-authored tests). Don't split an atomic cross-module refactor across the boundary; delegate only separable net-new or mechanical work. |
| Enabling an assertion that never ran (vacuous, skipped, commented-out) | Mutation-check it — a green proves nothing until you have seen it fail. See below. |

## Never-run assertions

RED exists because a test that has never failed is not known to test anything. That applies
retroactively: when you bind a discarded assertion, un-skip a test, or uncomment one, **a passing
result is ambiguous.** It means either "the behaviour is correct" or "it still isn't asserting" —
and those are indistinguishable from the outside.

Resolve it by mutation: change the expectation to something you know is wrong, confirm it fails
with the message you expect, then change it back. It costs one test run and is the only thing that
separates a live assertion from a second silent no-op.

**Corollary — a comment near a never-run assertion is not evidence.** If it describes behaviour the
assertion never actually verified, it records what someone believed at the time, not what the code
does. Verify the real behaviour, then delete the comment rather than leave a false claim in the file.

## Code Style

- **Default to no comments.** Add one only when the WHY is non-obvious — a hidden constraint, a subtle invariant, a workaround for a specific bug.
- **Don't reference the current task, fix, issue number, PR, or callers in comments** — those belong in the commit message and rot as the codebase evolves.
- **Don't explain WHAT the code does** — well-named identifiers do that. Comments say WHY, not WHAT.

## Common Mistakes

See `references/common-mistakes.md` for anti-patterns to avoid.

## Vault Integration

Use the **vault skill** to track work:

**Before starting:**

```bash
# Find project directory
fd -t d -d 1 -i "<project>" ~/Documents/Notes/Projects
```

Create task file: `Projects/<project>/Tasks/<YYYY-MM-DD HHMMSS> <Title>.md`

**During development:**
Log significant decisions in the task file:

```markdown
### <timestamp>

- [decision] Used X approach because Y
- [gotcha] Watch out for Z
- [ ] Still need to handle edge case
```

**After completing:**

1. Update task frontmatter: `status: done`
2. Write summary in Outcome section
3. Create knowledge notes for reusable patterns
4. Update memory skill with atomic facts
