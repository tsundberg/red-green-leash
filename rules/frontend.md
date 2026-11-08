# Frontend — React/TypeScript

> Reference document for the React frontend. Not auto-loaded — read on demand when working on frontend code.

## Build Commands

Project-specific: the test, dev-server, and build commands vary with each project. Document them here.

### MANDATORY: The test runner does not type-check — run the full build before "done"

A green test run does NOT mean the code compiles. Run the full build (see [project.md](project.md)) before considering
any frontend change finished. Watch for the strict-null gaps the test runner ignores but the compiler enforces:
indexing a `.find()` result after `expect(x).toBeDefined()` (which does not narrow the type — use `filter()` into a
typed array and index that, or a guard), unchecked optional access, and unchecked indexing.

## Frontend Structure (`frontend/src/`)

- **Framework:** React with TypeScript, built with Vite
- **Main component:** `App.tsx` (main app)
- **Backend communication:** `infra/ApiClient.ts`
- **Component organization:** `components/` directory mirrors backend features

## MANDATORY: Frontend components are class components

We are old-school, backend-minded developers. React class components match the way we think: explicit state, explicit
lifecycle, no magic. Every new production component in `frontend/src/` must extend `React.Component<props, state>`. Do
not write functional components. Do not use `useState`, `useEffect`, or other hooks in production code. Consumer
components subscribe to a `React.Context` via `static contextType = SomeContext` and read `this.context`.

Test probes and test-only harnesses may stay functional, since they are not production code. A narrow helper like a
`useX()` convenience wrapper around `useContext` is fine only if the surrounding production code is still a class
component using `static contextType`.

Before writing a new `.tsx` component, scan one or two existing neighbors to confirm the pattern.

## MANDATORY: Missing response fields fall back to their defaults

When a component reads a field that a newer backend response carries, a missing field must fall back to the behaviour
the previous release had. Never break the render, never treat `undefined` as `false` by accident and call it a day:
write the fallback explicitly, so the component works both before and after the backend it talks to is updated
anywhere in the rollout. See the API contract rule in [backend.md](backend.md) for the other side of the seam.

The fallback is also what makes a frontend rollout safe ahead of the backend: a component that reads a missing field
as its default degrades to today's behaviour instead of failing on an instance whose backend has not caught up.

The fallback is temporary. Once every instance runs the backend that serves the field, a cleaning deploy removes the
fallback code, or a decision is recorded to keep it as permanent robustness. A fallback left in place unmarked is debt
the next reader has to reverse-engineer, see the API contract rule in [backend.md](backend.md).

## Code Style

- Never swallow exceptions. If a catch block cannot handle the error meaningfully, rethrow it or throw a new `Error`
  with context. For example, write `throw new Error("Failed to fetch", { cause: e })` not `return null`. Silent `catch`
  blocks hide bugs
- Always use TypeScript with explicit type annotations on variables, function parameters, and return types. Never omit
  type annotations
- Avoid `any` — use proper types. If a type is complex, define an interface
- Define `props` and `state` interfaces for every React component, even if empty. Use lowercase names (
  `interface props`, `interface state`), not PascalCase
- Use `const` by default — only use `let` when reassignment is necessary
- Never use ternary operators; use `if`/`else` blocks instead — zero tolerance, no exceptions. Never use nullish
  coalescing (`??`) or optional chaining (`?.`) — use explicit `if`/`else` blocks instead
- Extract method calls into variables before passing them as function arguments. This simplifies debugging
  by making intermediate values visible. For example, write
  `const items = fetchItems(userId); const filtered = filterByStatus(items); setItems(filtered);`
  not `setItems(filterByStatus(fetchItems(userId)))`
- Never use conditional rendering that changes the number or type of DOM children (e.g. `null`, fragments, or swapping
  element types). Always render the element and control visibility through empty content or CSS. React's DOM
  reconciliation conflicts with direct DOM manipulation when child nodes appear or disappear
- Single abstraction level per function; extract low-level details into separate functions
- Always use braces around `if` statement bodies, even for one-liners
- Import order: external libraries first, local imports second, CSS imports last
- CSS class names use kebab-case, prefixed with component or feature name to avoid collisions (e.g.,
  `user-overview-card`, `save-button`)
- Plain CSS files per component — no CSS modules, no inline styles
- When encountering JavaScript components (`.js`/`.jsx`), refactor them to TypeScript (`.ts`/`.tsx`)
- File names are PascalCase for component, service, and helper module files (`OrderOpener.ts`,
  `ApiClient.ts`, `OrderMaintenance.tsx`), matching the exported component or primary symbol. Test files
  mirror the file they cover with `.test.tsx` / `.test.ts`. This is distinct from the lowercase `interface props`/
  `interface state` and kebab-case CSS-class rules above — those govern identifiers and CSS classes, not file names.

### MANDATORY: Guard async submit handlers against double-POST

Every async handler that performs a state-changing request (`POST`, `PUT`, `DELETE`, `PATCH`) — typically a click
handler that calls `this.context.post(...)` — MUST be guarded against duplicate submission. Two POSTs from a single
user intent cause duplicate records server-side. This is a class of bug we treat as critical.

The goal is to prevent double-POST. How you achieve it is up to you, but you MUST close both of these windows:

1. **Synchronous double-click.** A user double-taps; both clicks run before React commits the next render. This window
   is closed by a synchronous check at the top of the handler — typically a `private submitting` instance variable set
   before any `await`. State-only flags (e.g. `setState({spinner: true})`) DO NOT close this window: `setState` is
   async, both invocations of the handler pass the check.
2. **In-flight async window.** The first POST is awaiting a response (often hundreds of milliseconds). A click during
   this window must not fire a second POST. This is closed by disabling the submit button
   (`disabled={... || this.state.submitting}`).

Prefer the shared `submitOnce` helper (`infra/SubmitOnce`) for the handler body instead of re-deriving the
begin/`try`/`finally`/`stayOpen` flow by hand. It encodes the flow once (bail-if-submitting → begin → perform →
onSuccess/onFailure → reset only if the modal stays open) and is unit-tested, so the guard cannot silently drift
between modals. You still MUST disable the submit button via `state.submitting` (the in-flight window); the helper
manages that flag through the `lock` but not the binding.

Build the lock with `StateSubmitLock` from the same file instead of writing the object by hand. It keeps the instance
flag that closes the synchronous window, and passes every change to a callback, where the component sets the state its
submit button is disabled by. Hold one lock per component in a field. A component that resets more state when a
submit begins, such as an error message, does that in the same callback. Call it as:

```tsx
private readonly lock: SubmitLock = new StateSubmitLock((submitting: boolean) => {
  this.setState({submitting: submitting});
  if (submitting) {
    this.setState({saveFailed: false});
  }
});

await submitOnce(this.lock, () => this.context.post(url, payload, message),
  async () => { await this.props.onSaved(); this.props.hideModal(); },
  (result) => this.handleFailure(result));
```

The hand-rolled equivalent, for understanding the mechanics or when the helper does not fit:

```tsx
private submitting: boolean = false;

private async submit() {
  if (this.submitting) { return; }
  this.submitting = true;
  this.setState({submitting: true});
  try {
    await this.context.post(url, payload, ...);
    // ... close modal, reload, etc.
  } finally {
    this.submitting = false;
    this.setState({submitting: false});
  }
}
```

Both the instance variable and the state field are mandatory. A `try { ... } finally { ... }` around the body that
resets the lock is required so a thrown error doesn't leave the component permanently locked.

## Testing Strategy

- Vitest with React Testing Library
- Mock the backend at the network boundary (a mocked API handler), not internal modules
- Test files: `*.test.ts`, `*.test.tsx`

### MANDATORY: Test Names Are Plain Sentences

Test names must be plain sentences, not snake_case. Example: `'Disable Save button on invalid number'`, not
`'should_disable_save_button_on_invalid_number'`.

### MANDATORY: Black-Box Tests by Default

Every frontend test MUST be a black-box test unless it is physically impossible. "Difficult" or "inconvenient" is not a
valid reason to break this rule — only true impossibility is.

**What black-box means in practice:**

- **Interact through the UI, not through implementation.** Use `screen.getByRole`, `screen.getByText`,
  `screen.getByLabelText`, and `userEvent.click` / `userEvent.type`. Never reach into component internals, call
  component methods directly, or assert on internal state.
- **Assert on what the user sees.** Check rendered text, visible elements, navigation changes, and DOM structure — not
  props passed between components, internal state variables, or hook return values.
- **Never import or assert on internal functions, hooks, or state.** If a helper function needs testing, test it through
  the component that uses it. If that is truly impossible (pure utility with no UI surface), it may be tested directly —
  but justify why in a comment.
- **Mock at the network boundary, not at internal module boundaries.** Mock the backend API calls at the network/handler
  level. Do not mock internal services, hooks, or child components unless the child has its own integration with an
  external system.
- **No shallow rendering.** Always render the full component tree. Shallow rendering tests implementation structure, not
  behavior.
- **No snapshot tests.** Snapshots test implementation details (markup structure) and break on irrelevant changes. Test
  behavior instead.
- **Selectors must be resilient to refactoring.** Prefer accessible queries (`getByRole`, `getByLabelText`) over
  test-IDs, and test-IDs over CSS class selectors. Never query by component name or internal class.

**When you believe a test cannot be black-box:** Stop and explain why to the user before writing a white-box test. The
user decides whether the exception is justified.

### MANDATORY: Test the double-POST guard

Every async submit handler MUST have a black-box test that:

1. Renders the component with the API handler mocked to answer with a deferred (manually-resolvable) promise.
2. Clicks the submit button twice, synchronously, before resolving the promise.
3. Asserts that the handler received exactly one request.

Removing the guard, seeing this test fail, and then restoring the guard is the canonical way to confirm the test
proves the lock works (per "never trust a test you haven't seen fail").

**That test can pass for the wrong reason.** If the submit button is disabled from `state.submitting`, React flushes the
state update at the end of the first click and the DOM never dispatches the second one, so the handler runs once
regardless of what the instance variable does. A green double-submit test can leave the synchronous bail unexecuted,
which coverage reveals and the test never would. Use `submitOnce` or `withSubmitLock` from `infra/SubmitOnce` instead of
hand-rolling the flow, and the synchronous bail is proven once in `SubmitOnce.test.ts`, not per component.

When reviewing frontend code, every async submit handler that lacks both the instance-variable guard and the test
above is a finding. Flag it.
