---
name: Scala Development
description: Senior Scala developer using functional programming and Typelevel ecosystem. Use when writing Scala code, implementing Scala features, or working with sbt/bloop projects. Follows TDD methodology via development skill.
---

# Scala Development

Uses `development` skill for TDD workflow.

## Principles

- Functional programming techniques
- Avoid `Any`, `null`, `throw`, and unsafe patterns
- Prefer Typelevel ecosystem (cats, cats-effect, fs2)
- Use ADTs for domain modelling
- Errors as values (Either, EitherT, MonadError)

## Compilation Priority

1. **Check for `.bloop` directory first** (`ls -la`)

**If `.bloop` exists — use bloop:**
```bash
bloop compile <module-name>
bloop test <module-name>
bloop test <module-name> -o "*<filename>*"
```
Module name is `root` for non-modular projects.

**Scala.js test modules don't run under bloop** (`Missing configured test frameworks`) — run them with
`sbt <jsModule>/testOnly '*Spec*'`. bloop still *compiles* JS, it just can't execute its tests.

**If no `.bloop` — use sbt:**
```bash
sbt compile
sbt test
sbt "testOnly *SpecName*"
```

2. **ALWAYS** run `sbt commitCheck` after completing code changes.

## Before Committing Scala Changes

Before committing any non-trivial Scala edit — TDD refactor phase, bug fix, feature work, standalone refactor — run these structural fix-loops on the changed files. `commitCheck` verifies correctness; these verify structural quality. Both are required.

1. Invoke `Skill fix-method-structure <files>` — runs the method-structure-reviewer/fixer fix-loop until the diff is clean of >2-line lambda/case bodies, multi-control-structure methods, and other extract-method opportunities.
2. Invoke `Skill fix-primitive-obsession <files>` — runs the primitive-obsession-reviewer/fixer fix-loop until method signatures and case-class fields no longer carry raw String/Int/Tuple primitives where domain types fit.

**Always pass an explicit file scope** — the GREEN-phase commit plus any uncommitted refactor changes. Compute it like:

```bash
git diff --name-only HEAD~1 HEAD  # GREEN commit files
git diff --name-only HEAD          # uncommitted refactor edits
```

Pass the union as positional args. Without an explicit scope, `fix-loop` defaults to `HEAD~3` and will pick up two previous commits' files unrelated to this change.

Both wrap their underlying agents via `fix-loop`, so iteration to clean is automatic. Order: method-structure first (extracts blocks into named methods), then primitive-obsession (catches the resulting signatures). Either order converges.

### Red Flags — don't skip the fix-loops
- "This is a targeted bug fix, not TDD, so the hooks don't apply." They apply to any non-trivial Scala edit. If the diff contains real logic changes, run them.
- "commitCheck passed, I'm ready to commit." commitCheck is necessary, not sufficient.

## Testing (MUnit)

```scala
class FeatureSpec extends munit.FunSuite:

  test("describes expected behaviour") {
    // Arrange
    val input = createTestData()

    // Act
    val result = functionUnderTest(input)

    // Assert
    assertEquals(result, expected)
  }
```

**Effectful tests (cats-effect):**
```scala
class FeatureSpec extends munit.CatsEffectSuite:

  test("async operation succeeds") {
    for
      result <- asyncOperation()
    yield assertEquals(result, expected)
  }
```

**Assertions:**
- `assertEquals(obtained, expected)`
- `assertNotEquals(obtained, unexpected)`
- `assert(condition)`
- `interceptMessage[ExceptionType]("message") { code }`

## Error Handling Patterns

```scala
// Prefer Either over exceptions
def parse(input: String): Either[ParseError, Value] = ???

// Use EitherT for effectful errors
def fetchAndParse(id: Id): EitherT[IO, AppError, Value] = ???

// MonadError for generic error handling
def process[F[_]: MonadError[*[_], Throwable]](input: Input): F[Output] = ???
```
# Disallowed

## Avoid Option.get and Option.get

Never use Option.get or Either.get. Use fold instead. If you are certain that the get() method call will succeed, then
something previously must have ensured that and that is the place to do the fold or pattern match. If you don't know how to achieve
the result without using get, ask for help.

# Preferences

## Avoid Pattern Matches on Option and Either

Prefer Option.fold and Either.fold instead.

Alternatively, use Option.map(...).getOrElse(...), or Either.bimap(..., ...).merge instead.