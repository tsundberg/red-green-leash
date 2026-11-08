# TDD Workflow

This project uses Test-Driven Development. The unit of work is **one test**, taken through a full red-green-refactor
cycle, followed by a look back at the test list, before any other test is proposed or written:

1. **Red** — Write the failing test, run it, show the failure.
2. **Green** — Write minimal production code, run the test, show it pass.
3. **Refactor** — Improve code quality with tests green.
4. **Revise** — Feed what the cycle taught back into the test list (see "Revise the List After Every Cycle").

Test N+1 is not proposed until test N has completed all four phases. Two or more tests in a red state at once is a
process violation, not a style preference.

Never write or modify test code without explicit user approval. Before any test creation or modification, present the
test with justification and wait for user consent before proceeding. Production code changes that follow from approved
tests do not need to be shown or approved — just make the tests pass and move on. Do not present production code for
review. A brief summary of what was done at the end of a session is welcome, but do not walk through the code during the
TDD cycle.

Production code must never exist without a failing test demanding it. Temporary test compilation failures during red
phases are acceptable.

## Where to begin

### MANDATORY: Outside-In TDD (London School)

Always start from the outermost layer and work inward. Never jump straight to the inner layers.

**Backend:** Begin with the controller or acceptance test, mock the collaborators that don't exist yet, and let the
tests drive the interface design. Move one layer inward at a time.

The controller's first test is its happy path, driving the wiring. Behaviour tests start at the service. Outside-in says
where to *begin*, not where every test ends up living — see "A controller test covers security and wiring" in
[backend.md](backend.md) for what stays at the controller and what moves inward.

**Frontend:** Begin with the component the user sees. Render it, assert what's on screen, and mock backend calls (e.g.,
MSW handlers) or child components that don't exist yet. Work inward toward utilities and services.

Mocks are stepping stones, not endpoints. Replace them with real implementations or hand-rolled in-memory doubles as
soon as the interface is stable. Before a feature is considered done, all mocks must be replaced — even if commits have
happened in between. The final test suite should contain no framework mocks except at true external system boundaries.

The lifecycle of a collaborator:

1. Introduce it as a mock to drive the interface design
2. Once the interface is stable, build a hand-rolled in-memory implementation
3. Replace the mock with the in-memory double
4. The real implementation is driven out last, tested with a focused integration test against the real infrastructure

Acceptable permanent test doubles are in-memory implementations of storage/repository interfaces and simple hand-rolled
stubs for external systems (e.g. a fake email sender).

## Proposing a test

### MANDATORY: Two-Phase Test Presentation

When proposing tests, first present a high-level skeleton (test names + one-line descriptions). After the user approves
the overall plan, present each test's full implementation one at a time for individual approval before writing it.

One message contains at most one complete test implementation. Stacking several finished tests in a single message —
three or four with all their code, even offered for approval one by one — is forbidden: every test after the first is
proposed while the cycles before it are unfinished. The rough plan lives in the skeleton's one-line descriptions; a full
test body appears only when its turn has come.

The skeleton marks its commit points: the tests after which the work is in a state the user can commit. They are what
the per-test indicator counts down to (see "Show the Distance to the Next Commit").

### MANDATORY: Justify Every Proposed Test

When presenting a test for approval, include a short justification explaining what the test drives, what production
change it forces, and any non-obvious choices (assertion shape, fixtures, what's deliberately not tested). One or two
sentences is usually enough. The justification is what lets the reviewer say "yes, write it" without having to
reverse-engineer the intent from the code.

The justification includes a prediction of the red: where the test will fail, and the expected and actual values, in
the same three parts a red is reported with (see Pause at Red). If the test should pass on write, say so, and predict
the red of the deliberate break instead. Tag the prediction with how sure you are when it depends on behaviour you
have not checked, such as a framework's fallback.

When the red is reported, compare it with the prediction. **If the raw failure trace does not match your exact
prediction, STOP.** A red that fails somewhere else or with other values than predicted is not the red you proposed:
say so, explain the difference, and do not go green until the user has explicitly judged it. A red that matches
perfectly is evidence the test fails for the reason it was written.

### MANDATORY: Show the Distance to the Next Commit

A session can run for an hour or two, and the tiring part is not knowing how far away the next stopping point is. So
every test proposal opens with one line that places the test against the nearest commit point, for example
`Test 11 of 17. Commit point after test 13, three tests away counting this one.` The numbers follow the list as last
revised, so a total that grew or shrank is visible without comparing two lists.

That line is an indicator, not a recap. Do not restate the list with it. The list is restated only when it changes (see
"Revise the List After Every Cycle") and at the start of a session that picks a plan up midway, where the remaining
tests are listed with their positions and commit points before anything is proposed.

## Red

### MANDATORY: Red Must Be at the Assertion

A test that fails before reaching its assertion is a draft, not a red. Examples: `vi.spyOn` errors because the spied
method doesn't exist; an import doesn't resolve; render throws because a required class is missing; a Java test fails to
compile because the production class isn't there yet. When this happens, add the minimum scaffolding (declare the spied
method as an empty stub, create the missing class, fix the import) without yet implementing the behavior the test
asserts. Re-run. The test must now reach its assertion line and fail there. That second run is the real red — show the
user that one.

Self-check: look at the stack trace. If the top frame points at `expect`, `assertThat`, `verify`, or similar, the
failure is at the assertion. If it points at `spyOn`, an import line, a constructor, or any setup helper, you've shown a
draft. Add the missing piece and re-run before pausing.

### MANDATORY: Pause at Red

Whenever a test fails — initial red phase, deliberate breaks to verify catch quality, accidental breakage during
refactor, anything — stop and show the user the failure output. Wait for the user to confirm the failure is the right
shape before proceeding (to green, to restore, to anything else). Never skip this confirmation. Seeing the right failure
is as important as seeing the right pass, and the user is in charge of judging whether the failure proves what we wanted
to prove.

Report every red, backend and frontend, deliberate breaks included, with three things written out in the message itself:

- **Where:** the file and line of the failing assertion, e.g. `OrderServiceTest.java:142` or
  `UserProfileCard.test.tsx:477`.
- **Expected:** what the assertion wanted.
- **Actual:** what it got. When the failure carries no value, say what was there instead: for a `verify` that found no
  call, the calls that were made; for a timeout on a query such as `findByText`, what the DOM showed.

Do not leave the reader to dig them out of pasted runner output, and do not paraphrase them in prose. A bare
one-line summary like "expected 2 to be 1" is not enough when the values need context to read.

### MANDATORY: No Queued Reds

Never propose, write, or run a new test while an earlier test is still failing. The full cycle — red, user confirms the
failure shape, green, refactor, revise the list — must complete for the current test before the next test is even
discussed.

Warning signs that this rule is being broken:

- The previous message showed a red, and this message proposes another test.
- Two tests' implementations are presented back-to-back.
- A new test is being drafted while production code for the previous test has not been written.

If any of these is true, stop and go green on the current test. Then refactor — either do the cleanup, or say explicitly
"nothing to refactor". Then revise the list — either present the changes, or say explicitly "list unchanged". Only then
move to the next test.

### MANDATORY: Never Trust a Test You Haven't Seen Fail

Never trust a test you haven't seen fail for the right reason. If a new test passes immediately because the production
code already satisfies it, deliberately break the production code with a plausible defect (see "The Deliberate Break
Must Be a Plausible Defect") to force a red phase. Only after confirming the test fails for the expected reason,
restore it and go green. This applies to both frontend and backend tests.

When a new test passes on write, ask why before breaking anything. If the production code exists because an earlier
green phase wrote ahead of its test, remove that early code — the removal is the red, and it restores the natural
order; go green only when the new test demands the code. The deliberate break is for code that a genuinely earlier
test demanded or that predates the feature. Sabotage is the last resort, not the default remedy.

### MANDATORY: See the Test Fail for the Right Reason

A test that has only ever been seen as green proves nothing about its ability to catch a broken implementation.

Compilation or type errors aren't "red" — they're just things to fix. Get the code building and the test reaching its
assertion, then look for a meaningful failure.

The interesting failure is the one where the code builds, the production code looks like it should work, and the test
fails with a clear assertion mismatch (or expected exception) that pinpoints the missing behavior. That failure is
what proves the test would catch a regression. Sometimes you reach it after a single stub; sometimes after a partial
implementation that's one trivial change away from green. The number of iterations doesn't matter — the *quality* of
the failure does.

A failure that says "expected X, got Y" where Y is exactly what a half-built implementation would produce is the right
kind of red. A crash from a missing field or unmocked dependency (`NullPointerException`, `Cannot read properties of
undefined`) is usually the wrong kind — it doesn't demonstrate that the assertion would catch a real-world bug.

### MANDATORY: The Deliberate Break Must Be a Plausible Defect

Seeing a test fail only counts if the failure comes from a defect a real change could introduce. If the only way to make
a test red is to sabotage the exact line it asserts (a getter returning its field, a constructor storing its argument, a
passthrough returning its input), the test proves nothing beyond "the code is the code."

When you break to confirm a test can fail, the break must be a realistic mutation: remove a wiring call, drop a branch,
return the wrong collaborator, the kind of edit a developer might actually make. Returning null from a getter or
hardcoding the expected value is vandalism, not a valid red.

If no plausible defect can make the test fail, the behaviour isn't yours to test in isolation. It belongs to the
framework or is a trivial passthrough. Assert observable behaviour reached through a real collaborator instead, or drop
the test.

## Green

### MANDATORY: Minimalist Green Phase

When writing production code to pass a new test, write the absolute simplest, most literal implementation possible. If a
hardcoded value or a single `if` statement satisfies the current test, use it. Do not anticipate future test cases or
add architectural scaffolding the current test does not demand.

This rule restricts **generalization**, not **refactoring**. After going green, the refactor phase (see "Refactor and
revise") remains mandatory: extract helpers, rename for clarity, remove duplication, improve structure. What you must
not do during the first green is replace a literal implementation with a general algorithm before a second test forces
it. The path from `return 42` to `return a + b` is triangulation, not refactoring.

**Mandatory safety patterns are not scaffolding.** Codebase rules that mark a pattern MANDATORY (the double-POST guard
in frontend.md, transaction wrapping in backend.md, named constraints on schema, no cascading deletes, etc.) must be
present from the first green, even if the current test does not exercise the failure mode they protect. These are
load-bearing constraints on every instance of the construct, not speculative architecture waiting for a future test.

**"Simplest" operates within conventions.** The simplest implementation still follows the interface contracts the
codebase demands: services return domain value objects, not bare primitives; frontend components are class components;
variables are extracted before being passed as arguments. "Literal" refers to the logic the test exercises, not the
types or structure the conventions require.

Before writing any piece of production code in a green phase, ask: does the current red test fail — or not compile —
without it? If the test would still pass, the piece belongs to a later test and must not be written yet. This includes
pieces the plan has already settled, like a controller's permission declaration or a response field: the plan settles
the design, the test list as last revised settles the order. A security declaration, a validation — nothing is exempt;
if the current test cannot tell the difference, it is early. The one exception is what "Mandatory safety patterns are
not scaffolding" names: per-construct safety patterns are present from the first green by their own rule.

### MANDATORY: Triangulate Before Generalizing

A single test that passes against a hardcoded constant proves nothing. The production code could `return 42` and go
green. Before generalizing from a constant to an algorithm, write a second test with a different input that forces the
constant out. The first test fixes the shape; the second forces the implementation.

Only after two examples agree on the underlying logic should you replace the special cases with the general solution. If
the second example passes against the same hardcoded constant, the test pair is too narrow — pick an input that breaks
the shortcut.

## Refactor and revise

### MANDATORY: Plans Must Include a Refactoring Step

Every implementation plan MUST include an explicit final step for reviewing and refactoring the production code after
all tests pass. The refactor phase is not optional — it is the third phase of red-green-refactor. Omitting it from a
plan means the plan is incomplete. This step should look for duplication, dead code, unused imports, extracted-method
opportunities, missing concepts (a long parameter list, or values that travel together, see
[code-style.md](code-style.md)), and any other cleanup that the changes demand. The look covers each touched file as a
whole, not only the changed lines (the boy scout rule in [code-style.md](code-style.md)).

### MANDATORY: Refactor the Tests Too

The refactor phase covers test code as well as production code. After green, look for duplicated test setup, dead helper
methods, extractable test fixtures, and inconsistent naming across tests in the same class. Tests are read far more
often than they are written, and clarity there pays the same dividend as clarity in production code.

Do not extract shared setup so aggressively that individual tests stop being readable in isolation. A test helper that
hides what the test arranges is a liability; a helper that removes noise the reader does not need is a win. Aim for each
test to be understandable on its own, with setup that is obvious or named clearly enough to skip.

### MANDATORY: Revise the List After Every Cycle

The test list is approved when we know the least about the feature. Every cycle teaches something, and that has to
reach the list before the next red, or it evaporates between tests. So the refactor phase ends with a look at the list,
not at the code. Before test N+1 is proposed:

1. **Strike** tests the implementation made redundant, with a one-line reason each.
2. **Add** tests the code revealed: a branch the skeleton did not foresee, a triangulation partner, a fixture that
   turned out to matter, an edge that only became visible once the code existed.
3. **Reorder** when a discovery changes which test should come next.
4. **Say "list unchanged"** when nothing moved. Silence is not an answer; the reader cannot tell a considered no from a
   skipped step.

A changed list is presented the way the skeleton was: names and one-line descriptions, with strikes, additions and
commit points marked, and it needs the user's approval before the next test is proposed. If the change moves the next
commit point, say so in the same message. An added test still gets its own justification and approval when its turn
comes; adding it to the list and writing it are two separate steps.

This phase moves the list, not the plan. Design decisions the plan settled stay settled. If a discovery contradicts one
of them, say so and stop for a decision rather than quietly re-planning through the list.

## Test design

### MANDATORY: One Behavior Per Test

Each test should assert one behavior. When a test fails, the person reading the output should be able to name the broken
behavior from the test name alone, without inspecting the body. A test that asserts three unrelated things fails as a
unit, and the reader has to split the assertions apart to find which one went wrong.

If a behavior naturally produces several observable consequences, either assert them in one test with a single
conceptual name, or split into multiple tests. The test name, not the assertion count, is the test's unit.

Cases whose expected shapes differ — one answering `null` and another answering a value, one throwing and another
returning — are separate behaviors, not multiple assertions in one body, even when they share a conceptual name. A
parameterized test fits exactly when every case expects the same shape; the moment one case expects a different one,
it becomes its own test.

### MANDATORY: No Logic in Test Bodies

Test bodies should contain no conditionals (`if`, `switch`, ternary) and no loops (`for`, `while`, `forEach`). Each run
of a test should exercise exactly one path, so a green result means every assertion executed. A branch or loop inside a
test means some assertions may silently not run, and a green is meaningless.

If you find yourself reaching for a loop over inputs, write each case as a separate test instead. If the cases are
genuinely the same behavior with different data, use a parameterized test where the framework drives the iteration — the
body of each case still has no logic of its own.

### MANDATORY: Minimal Fixtures

A test should contain only the setup the behavior under test actually exercises. Extra fixtures — unused fields,
superfluous collaborators, broad builders that construct more than the test reads — obscure what matters, slow the
suite, and make tests harder to change. When a fixture is not needed for this test, leave it out. Add it back when a
test that actually needs it arrives.

If a shared `@BeforeEach` or `setUp` builds state that some tests in the class do not use, split the class or move the
shared setup into a named helper that each test calls explicitly. A reader should be able to look at a test in isolation
and see everything that is relevant to it, without guessing which parts of a blanket setup are load-bearing.

### MANDATORY: Hard-to-Write Test Is a Design Signal

When a test is painful to set up — long fixture chains, deep mocking, awkward constructor arguments, reaching through
several layers to get at the behavior — treat that as feedback from the design, not as a problem with the test. The
production code is telling you it is coupled to too many collaborators or that the unit under test has too many
responsibilities.

Before adding more setup, ask whether the production code can be simplified: extract a collaborator, split a class, move
logic closer to the data. The right response to a hard-to-write test is often a production code change, not a more
elaborate test fixture.

## Changing existing code

### MANDATORY: Test Safety Net Before Any Production Code Change

**This rule is unconditional and applies to ALL production code changes — no exceptions.**

Before modifying, refactoring, or deleting any production code, you MUST:

1. **Verify existing test coverage.** Search for tests that exercise the code being changed. Run them and confirm they
   pass.
2. **If no tests exist, write characterization tests first.** These tests lock in the current behavior so that any
   unintended change is caught immediately. Characterization tests MUST be presented to the user and approved before
   proceeding.
3. **All new or modified tests require explicit user review.** Present the test code with a clear justification. Do NOT
   proceed with production code changes until the user has approved the tests.
4. **Never change production code without a passing safety net.** If you cannot demonstrate that tests cover the code
   you are about to change, you are not allowed to change it. Stop and add tests first.

Changing production code without verified test coverage is **strictly forbidden**. This applies equally to new features,
bug fixes, refactoring, renaming, moving code, and deleting code.

### MANDATORY: Safe Deletion Order

When removing a method, constructor, or class, never delete it before all callers have been migrated away. The safe
order is:

1. Update all callers to stop using the target method/constructor
2. Build and verify everything compiles and tests pass
3. Only then remove the now-unused method/constructor
4. Build again to confirm

Deleting first and fixing callers after breaks the build in between and risks missing callers.
