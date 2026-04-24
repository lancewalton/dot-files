---
name: Refactor
description: Improve code design without changing behaviour. Use when restructuring code, eliminating duplication, or improving readability. All tests must pass before and after.
---

# Refactor

## Core Rules (Non-Negotiable)
@
1. **NEVER change behaviour** — refactoring preserves existing functionality
2. **All tests must pass** before starting and after completing
3. **Small incremental changes** — commit after each successful refactoring
4. **Run tests after every change** — catch regressions immediately

## The Refactor Cycle

```
✅ VERIFY  → Run all tests, confirm green state
🔍 ANALYSE → Identify code smell or improvement opportunity
🔵 REFACTOR → Apply ONE transformation
✅ VERIFY  → Run all tests, confirm still green
💾 COMMIT  → Save working state
🔁 REPEAT  → Continue until goal achieved
```

## Goals

| Goal | Techniques |
|------|------------|
| Eliminate duplication | Extract method, extract variable, pull up method |
| Improve clarity | Rename, inline temp, introduce explaining variable |
| Simplify conditionals | Decompose conditional, consolidate conditional |
| Improve structure | Extract class, move method, replace inheritance with delegation |
| Reduce code size | Eliminate nesting by extracting helper functions/methods |

## Code Smells to Address

- **Duplication** — same code in multiple places
- **Long methods** — do too much, hard to understand
- **Large classes** — too many responsibilities
- **Long parameter lists** — difficult to call correctly
- **Feature envy** — method uses another class more than its own
- **Data clumps** — groups of data that appear together repeatedly
- **Primitive obsession** — using primitives instead of small objects
- **Inappropriate intimacy** — classes too dependent on each other's internals

## What Refactoring Is NOT

- Adding new features
- Fixing bugs (unless the fix is purely structural)
- Optimising performance (unless it doesn't change behaviour)
- Changing external APIs

## Announcing Changes

```
🔵 REFACTOR → [smell]: [transformation]
```

Examples:

- `🔵 REFACTOR → duplication: Extract method calculateTotal`
- `🔵 REFACTOR → long method: Split processOrder into validate and execute`
- `🔵 REFACTOR → feature envy: Move calculateDiscount to Order class`

## Safety Checklist

Before starting:

- [ ] All tests pass
- [ ] Working copy is clean (committed)
- [ ] Understand the code being refactored

After each change:

- [ ] Tests still pass
- [ ] Behaviour unchanged
- [ ] Code is cleaner/clearer

## Integration with Development Skill

Refactoring is the BLUE phase of TDD:

1. Complete RED-GREEN cycle
2. Invoke refactor skill
3. Apply improvements while keeping tests green
4. Commit refactored state
