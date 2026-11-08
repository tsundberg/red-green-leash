# Testing

Conventions for what a good test looks like. The red-green-refactor workflow lives in [tdd.md](tdd.md).

## MANDATORY: Every Test Must Assert

Every test must contain at least one explicit assertion (`assertThat`, `assertEquals`, `verify`, etc.). A test that
merely calls production code and passes because no exception was thrown is not a real test — it proves nothing about
correctness. If the only thing you can think of is "it doesn't throw," think harder about what observable effect the
code should have and assert on that. If you still cannot identify a meaningful assertion, ask the user for guidance
rather than writing an assertionless test.

## MANDATORY: Arrange, Act, Assert, and the asserted value is `actual`

Every test has three blocks in this order, separated by a blank line:

1. **Arrange:** the inputs and the expected values, each in a named variable. A literal the test is about (the
   typed text, the id, the amount) is arranged here, not written inline in the act call.
2. **Act:** the one call the test is about.
3. **Assert:** each value that is asserted is first read into a variable named `actual`, or `actualX` when there
   are several, and then asserted. Never `result`, and never a name for what the thing is (`valid`, `field`,
   `message`): the name says it is the outcome under test. Never fold a lookup into the assertion
   (`expect(screen.getByText(...))`, `assertThat(response.status())`). When the act returns the outcome, that
   returned value is `actual`; compare it with an arranged `expected` where you can.

Values the test only uses on the way (a field that gets typed into, a button that gets clicked) keep descriptive
names. Before presenting a test, read its body against two questions: is there an arrange block, and is every
asserted value named `actual`? This applies to backend and frontend tests alike.

A backend test that expects an exception is the one shape without an `actual`. It uses the `assertThatThrownBy(() ->`
form shown in [backend.md](backend.md): the call on its own line inside the lambda is the act, and the assertions are
chained after it, with a check of the message. Do not reach for `catchThrowable` to get an `actual` variable.

## MANDATORY: A test name describes behaviour, not code

Name a test after what can be observed from outside the code under test: what a user sees or does in a component
test, what a caller gets in a service or controller test. Leave out method, prop, class and variable names:
`Picks the item that is clicked`, not `Calls onPick with the item that was clicked`. The body may check
a callback, the name does not mention it.

The exception is a test of a contract the outside world relies on, where the name is the behaviour: a permission, a
path parameter, a route (`should_require_the_manage_orders_permission`).

## MANDATORY: A test lives as close as possible to what it tests

A behaviour is tested against the smallest unit that owns it, and the test goes in that unit's own test file. Every
component and service with behaviour of its own has its own test file. A parent's tests cover only what the parent adds:
what it passes down and what it does with the answer.

Outside-in says where to begin, not where a test ends up. When a green extracts or creates a component, the behaviour
tests go with it in the same change. The full tree is still rendered, so a component's test includes its children; only
the starting point moves.

## MANDATORY: Propose Negative and Edge-Case Tests

When proposing a set of tests, do NOT stop at happy-path coverage. Always include a section listing potential negative
and edge-case tests. Think hard about what can go wrong — be thorough and creative. Consider at minimum:

- **Duplicate operations** — What if the same action runs twice? (e.g., inserting the same primary key, processing the
  same event twice, concurrent instances doing the same work)
- **Boundary/interval edges** — What happens at the start, end, or just outside an interval? Off-by-one? Empty ranges?
  Zero? Negative values? Maximum values?
- **Missing or null inputs** — What if expected data is absent? Empty strings? Empty collections?
- **Untrusted input** — Is user-typed data used for anything beyond raw storage? If the value is rendered back, parsed,
  used in a query or path, or forwarded to another system, consider what surprising or malicious content could do.
  Strings the user controls are hostile until proven otherwise. Defense belongs on the frontend (catch it early, tell
  the user) and again on the backend (request objects, path parameters, query parameters), since the backend cannot
  trust the frontend.
- **Ordering and concurrency** — What if operations arrive out of order, or two run simultaneously?
- **Constraint violations** — What if uniqueness, foreign key, or other constraints are hit?
- **State transitions** — What if the entity is in the wrong state for the operation? (e.g., cancelling something
  already cancelled, modifying something that's been finalized)
- **Authorization and ownership** — What if the caller doesn't own the resource? What if they lack the required role?
- **Partial failure** — What if a multi-step operation fails halfway? Is the system left in a consistent state?
- **Stale or outdated data** — What if the data changed between reading and writing? Race conditions?

This list is a starting point, not a ceiling. If you see risks specific to the feature that don't fit these categories,
add them. Present these as a short list of risks with the proposed tests. Ask the user: "Are you worried about any of
these?" Let the user pick which negative tests to include before writing them. The happy path is great; the sad path
saves us from embarrassing bugs.

## MANDATORY: Acceptance Tests Drive the App Like a User

Playwright acceptance tests must exercise the application only through the affordances a real user touches:
locators, clicks, typed input, `setInputFiles`, and reading what is on screen. They must NOT reach around the UI with
`page.evaluate` calling `fetch`, nor hit API endpoints directly, to make an action happen or to observe its result. If
the user clicks a link to download a file, the test clicks that link — it does not re-issue the download with a
hand-rolled request.

Why: reaching around the UI reimplements production behavior inside the test and then asserts that the reimplementation
works, which proves nothing about the real interaction and is the same anti-pattern as leaving a mock in place instead
of the real thing. The whole point of driving the app like a user is to prove the user-visible path.

When a technical obstacle makes the user path awkward to observe (e.g. a `window.open(blob)` download with no download
event, whose response body Playwright cannot re-read afterwards), that is a signal to find the right observation
technique — intercept the request the click fires with `page.route`, proxy it with `route.fetch()`, capture the bytes,
and `route.fulfill` so the app still works — NOT a license to bypass the click. `page.evaluate` is still fine for pure
observation that has no UI affordance (reading `sessionStorage`, `window.location`), never to trigger an action or fetch
application data.

Navigating to a direct URL is an allowed user action. Users type and paste deep links into the address bar, and
deep links are a supported way to reach a page, so a test may open a view with `page.navigate(url)`
at its deep-link URL. What is not allowed is driving navigation by executing JavaScript in the page, such as
`page.evaluate(() => window.location.hash = ...)`, since no real user runs JavaScript to move around the app. When the
test is about the menu, click the menu; when the destination rather than the path to it is what matters, a deep-link
navigate is fine.

## MANDATORY: Manual Test Instructions

At the end of a completed feature or bug fix, include step-by-step instructions for manually testing at least the happy
path. These should be concrete and specific — URLs to visit, buttons to click, data to enter, expected results to see.
The instructions will be used for demos and manual verification.
