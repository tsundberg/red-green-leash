# Code Style

Cross-cutting style rules. Stack-specific style lives in [backend.md](backend.md) and [frontend.md](frontend.md).

## MANDATORY: No Inline Method Calls as Arguments

Never embed method calls as arguments to other method calls. Extract them to named local variables first. This applies
to both backend and frontend code. See [backend.md](backend.md) and [frontend.md](frontend.md) for language-specific
examples.

## MANDATORY: No console output in production code

Never write to the console in production sources (`System.out.println`, `console.log`, `printStackTrace`, etc.). Use the
project's logging framework instead. Never log PII or raw SQL. See [backend.md](backend.md) for the full rule and
logging-level guidance.

## Keep methods and functions short

Methods and functions should try to stay under 15 lines; refactor longer ones, 16 lines may be acceptable.

## Prefer positive logic over negation

Phrase conditionals positively. Filter chains, guards, and helper names should describe what something *is* rather than
what it *isn't*.

- Combine `if (!a) continue; if (!b) continue;` chains into one positive predicate or a stream of positive
  `.filter(...)`
  calls.
- Rename or wrap helpers so the call site reads positive: `isEligible(...)` beats `!isExcluded(...)` at the filter line.
- It is fine to keep a single `!` inside a small named helper if the underlying check is naturally phrased as "has X"
  and the caller cares about "lacks X". Wrap the negation, don't expose it.

Why: stacked `!`s read like puzzles, especially when combined with `&&` or `||`. Positive predicates compose better and
survive future edits without flipping the meaning by accident.

## A long parameter list is a missing concept

When a constructor, method, function or record takes more than four parameters, stop and look for the concept they
share before adding another. Four is a prompt to look, not a limit to stay under. Record components count.

Other signals point the same way, whatever the count:

- The same group of values travels together through several signatures.
- Fields are set together, or are null together. Three nullable fields that encode one state are one value.
- A new parameter is added only to pass it along.

Name the concept as a record in the backend and an interface in the frontend, and move the behaviour that belongs to
it there. Value objects for each field are not enough: a 14-component order record still hid an outcome that was
pending, completed or failed, spread over three nullable fields.

A signature that a framework dictates, and a record that really is one flat thing, may stay long. Say why in a
comment. Existing long lists are fixed when they are touched, not swept on the side.

## Leave every file you touch better than you found it

The boy scout rule, from Robert C. Martin's *Clean Code*. When a change touches a file, look at the whole file, not
only the lines you changed, and fix what is worse than it should be. Any improvement counts, not one kind of
improvement: duplicated setup, a helper or field in the wrong place, a misleading name, a test without the AAA layout
or with its asserted value called `result`, a call or a `new` inlined as an argument, a concrete type where an
interface exists, a long parameter list, a line over 120 characters. The rules in this directory are the checklist.

Keep it reviewable:

- Only the files the change touches. Do not sweep the package on the side.
- Refactoring only. Behaviour stays the same and the tests stay green. An improvement that needs a new test, or changes
  what the code does, is feature work: note it and do it under the TDD rules.
- A small cleanup goes into the change's own commit. A larger one gets a commit of its own next to it, with its own
  full build, so the feature diff stays readable. A cleanup too large to review in one sitting is too large: split it,
  or ask first.
- A cleanup left undone is named, with the reason. "Pre-existing, left alone" is not a reason.

Do not narrow the rule to one item on the list. Wrapping the long lines in a test file and calling that the boy
scout commit missed the point: the same setup was written out a dozen times in that file, and that was the bigger
problem.
