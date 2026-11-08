# Backend — Java/Gradle

> Reference document for the Java backend. Not auto-loaded — read on demand when working on backend code.

## Build Commands

```bash
# Build without tests
./gradlew clean assemble

# Run tests only
./gradlew test

# Run backend tests only (excludes frontend tests)
./gradlew test -x :frontend:test

# Run a specific backend test class
./gradlew :test --tests "com.example.event.EventServiceTest"

# Run Cucumber tests by layer
./gradlew cucumber --tags="@unit" -Ddatabase=in-memory
./gradlew cucumber --tags="@integration" -Ddatabase=in-memory

# Check for dependency updates
./gradlew checkUpdates
```

## Backend Structure (`src/main/java/com/example/`)

- **Entry point:** `Main.java` - starts Javalin, schedules jobs
- **Routes:** `Routes.java` - all HTTP endpoint definitions, plus the filters and exception handling
- **Pattern per feature package:**
    - Services and repositories live at the feature root
    - Controllers live in versioned subpackages (`v1/`, `v2/`, etc.)
    - Repositories follow the pattern: interface `FooRepository`, production implementation `SqlFooRepository`, test
      implementation `InMemoryFooRepository` (in test directory)
  - A class name says what the class does inside its package, and nothing the package already says. A controller in
    `admin/inventory/v1` is `GetInventoryController`, not `GetAdminInventoryInventoryController`: the
    package carries the feature and the version, the name carries the HTTP verb, the domain thing, and the role
    suffix (`Controller`, `Request`, `Response`, `Service`). `Routes` imports the classes it routes to and
    uses their simple names; a name that collides with another import is referenced by its complete name inline
    instead.

Example structure:

```
src/main/java/.../orders/
├── OrderService.java           # Service at root
├── OrderRepository.java        # Repository interface at root
├── SqlOrderRepository.java     # SQL implementation
├── v1/
│   └── GetOrdersController.java
└── v2/
    └── GetOrdersController.java

src/test/java/.../orders/
└── InMemoryOrderRepository.java # Test implementation
```

Key feature packages: one per business feature, e.g. `orders/`, `admin/`

## Routes, controllers, and ids (canonical references)

These patterns barely change between features. Mimic the referenced file rather than re-deriving the shape.

### A typical controller

**Constructor**

- Declare the permission with `requirePermission(permission)`. Some endpoints legitimately require none — login,
  public endpoints, and cases decided deliberately. When a controller declares no permission, say why in a comment.
- Declare every **mandatory** path parameter with `requireRouteParam(":id")`, or several at once
  with `requireRouteParams(...)`. The base class then rejects a missing one with a 400 before `handle` runs. An
  optional parameter is not declared; it is read with `getRouteParam(name)`. A mandatory parameter that is read but
  never declared is a silent defect. `requireQueryParam(...)` does the same for query parameters.

**handle**

- **Transform the arguments** into the form the service wants: `payload.toModel()` and
  `routeParams.getParam(":id", SomeId.class)`.
- **Call the service.**
- **Transform the answer** into what the frontend reads — `Response.fromModel(...)`.

Nothing else. No business logic, no figures, no branching on data.

Two exceptions to the three steps:

- A response that is not 200 sets it explicitly with `setStatus(...)`, for example `setStatus(HttpStatus.CREATED_201)`
  after a create. Everything else is 200 by default.
- A controller that returns bytes rather than JSON is invoked with `handleFile(ctx)` instead of `handle(ctx)`.

**Errors**

Throw, and let the abstract controller deal with it. It gives us, once, for every endpoint:

- the permission check, from what the constructor declared
- status mapping — 401 unknown user, 403 permission denied, 400 bad request, 500 otherwise
- logging of the failure and of the request

Signal a client error by throwing `InvalidRequestException`. Never map a status by hand.

**Its tests**

- `getRequiredPermissions()` is asserted — always, including when it is empty.
- The request record's `toModel()` has its own test, however trivial.
- Security and wiring only; figures and edge cases belong to the service and response tests. See
  [A controller test covers security and wiring](#mandatory-a-controller-test-covers-security-and-wiring).

### Route registration

All HTTP endpoints are registered in `src/main/java/com/example/Routes.java`, one entry per route. The
controller is constructed from the `Services` passed into `routes(...)`, then `.handle(ctx)` is called. Path
parameters use Javalin's `{name}` syntax in the route:

```java
app.get("/api/v1/orders/tree", ctx -> {
  OrderService orderService = services.getOrderService();
  UserService userService = services.getUserService();
  GetOrderTreeController controller = new GetOrderTreeController(orderService, userService);
  controller.handle(ctx);
});

app.post("/api/v1/orders/", ctx -> new CreateOrderController(...).handle(ctx));

app.delete("/api/v1/orders/{orderId}", ctx -> new DeleteOrderController(...).handle(ctx));
```

The controller still reads the parameter as `:orderId`. The web adapter puts the colon back on, so
`requireRouteParam(":orderId")` and `routeParams.getParam(":orderId")` are unchanged.

Add new endpoints next to their siblings (e.g. the order routes near `/api/v1/orders/tree`). A controller that
needs a new collaborator gets a matching `getFooService()` added to `Services`.

### Create controller returning a generated id

Canonical: `orders/v1/CreateOrderController.java`. The controller extends
`BaseController<RequestRecord, ResponseRecord>`, declares its permission in the constructor, converts the request to a
domain command with `payload.toModel()`, calls the service, and returns `ResponseRecord.fromModel(...)`:

```java
public class CreateOrderController extends BaseController<CreateOrderRequest, OrderResponse> {
  public CreateOrderController(OrderService orderService, UserService userService) {
    super(userService);
    this.orderService = orderService;

    Permission permission = new Permission("manage-orders");
    requirePermission(permission);
  }

  @Override
  public OrderResponse handle(User user, CreateOrderRequest payload) {
    CreateOrder createOrder = payload.toModel();
    Order order = orderService.createOrder(createOrder);
    setStatus(HttpStatus.CREATED_201);
    return OrderResponse.fromModel(order);
  }
}
```

The request record carries a `toModel()`, the response record a static `fromModel(...)` (see Conventions).

### Path parameters (`:id`)

Canonical: `orders/v1/DeleteOrderController.java`. Declare the parameter in the constructor and
read it as a typed value object in `handle`:

```java
requireRouteParam(":id");   // in the constructor

@Override
public String handle(User user, RouteParams routeParams) {
  OrderId id = routeParams.getParam(":id", OrderId.class);
  service.delete(id);
  return Json.ok();
}
```

`getParameter(":id", SomeId.class)` constructs the id via its single-`String` constructor, so the id type must accept a
`String`.

### System-generated ids

Ids are value-object records wrapping a `String` or `Integer`, validating non-null in the compact constructor:

```java
public record TaskId(String id) {
  public TaskId {
    if (id == null) {
      throw new IllegalArgumentException("Id is mandatory");
    }
  }
}
```

For a fresh URL-safe id, generate a UUID via `infra/IdGenerator.java` (`newId()`). Integer/identity ids
(e.g. `OrderId`) are minted by the database `GENERATED ALWAYS AS IDENTITY` column and read back; string/UUID ids are
generated in code before the insert.

## MANDATORY: API Changes Must Work for Both the Old and the New Frontend

The same window exists one level up, between the backend and the frontend, and it is wider: after a push the old
frontend keeps running on some instances while the new one rolls out to the rest, and the two serve traffic side by
side until every instance is updated. A release is only safe when it works with both versions on the other side of
the seam:

- A field added to a response is safe, since the old frontend ignores what it does not read. A field removed or
  renamed from a response breaks the old frontend reading it, and ships only after every deployed frontend has
  stopped reading it.
- Backend validation or enforcement must not reject requests the currently deployed frontend legitimately sends.
  If the new backend rejects something the old frontend still posts, the rejection must be an explicit, agreed
  release decision with a risk note, never a side effect of the validation being new.
- A new frontend that depends on new backend behaviour rolls out only after the backend it depends on has reached
  every instance. Until then the frontend must degrade to the previous behaviour, see the response-field rule in
  [frontend.md](frontend.md).

Compatibility code written for a rollout is debt with no natural expiry. When the fallback is written, plan its
removal: once every instance runs the new release on both sides of the seam, a cleaning deploy drops the tolerant
behaviour and tightens the contract, so the next reader does not have to guess which fallbacks are still load-bearing.
If the tolerant behaviour is worth keeping permanently, that is a decision recorded in the code or the plan, not a
default.

## Database

- **Production:** PostgreSQL
- **Development/Test:** H2 in-memory
- **ORM:** JDBI 3 (SQL Object API)
- **Migrations:** Flyway (`src/main/resources/db/migration/V*.sql`)

### Migration files and DDL

Migrations are named `V<N>__<description>.sql`, numbered sequentially. Check the highest existing `V*` and use the next
number. A simple table (canonical: `V10__create_tags.sql`) looks like:

```sql
CREATE TABLE tags
(
  tag_key     VARCHAR(64)   NOT NULL,
  description VARCHAR(1024) NOT NULL,
  color       VARCHAR(50)   NOT NULL,
  PRIMARY KEY (tag_key)
);

CREATE TABLE tag_options
(
  tag_key    VARCHAR(64) NOT NULL,
  option_key VARCHAR(50) NOT NULL,
  PRIMARY KEY (tag_key, option_key),
  CONSTRAINT fk_tag_option_tag_key FOREIGN KEY (tag_key)
    REFERENCES tags (tag_key)
);
```

String/UUID ids are `VARCHAR(36)` columns populated in code. Honor the named-constraint rule below for every
constraint.

### Wiping the database in tests

`src/test/java/com/example/infra/TestDatabase.java` has a single `clear(DaoProvider)` that `DELETE`s every table
inside one transaction, child rows before parent rows. **Every new table must be added to this list**, or the SQL
repository tests flake intermittently as leftover rows from other tests collide with unique/foreign keys. Add the new
`DELETE FROM <table>` ahead of any table it references by foreign key.

### MANDATORY: Named Constraints and Indexes

All constraints (UNIQUE, FOREIGN KEY, CHECK) and indexes must have explicit names. Unnamed constraints cannot be
reliably dropped in migrations, especially on H2. For example, write
`CONSTRAINT uq_tags_owner UNIQUE (tag_id, owner_id)` not `UNIQUE (tag_id, owner_id)`.

### MANDATORY: Schema Changes Must Work for Both Old and New Code

During every deploy there is a window where the previous release still serves traffic against the migrated schema:
the new release runs the Flyway migrations when it boots, while the platform keeps the old instances serving until
the new release is up. Every schema change must therefore be written so that both the old code and the new code work
against it at every moment of the deploy. Concretely:

- A migration that adds a table or column must be deployed **before, or in the same release as,** the code that
  references it. Code must never reference a table or column whose migration ships in a later release.
- Dropping or renaming is dangerous and needs extra care. The old code must be able to handle the change just as
  well as the new code. That means a drop or rename ships **one release after** the last code that referenced the
  table or column was removed. Never drop something in the same release where the code stops using it; the old
  instances still query it during the deploy window and the app is broken until they die.

When in doubt, split the work: ship the code change first, ship the destructive migration in the next release.

### No Cascading Deletes

Never use `ON DELETE CASCADE` or `ON UPDATE CASCADE` in schema definitions. A seemingly simple delete must not silently
wipe related rows across the database. Instead, perform multi-table deletes explicitly inside a transaction: delete
child rows first, then the parent row, each as a separate statement. This keeps the blast radius visible in the code and
impossible to trigger by accident.

### MANDATORY: Transactions for Multi-Statement Writes

Every repository method that performs more than one write statement (INSERT, UPDATE, DELETE) **must** wrap all
statements in a JDBI transaction using `jdbi.useTransaction(handle -> { ... })` (void) or
`jdbi.inTransaction(handle -> { ... })` (returns a value). A method that issues two or more writes without a transaction
is a bug — partial writes corrupt data silently and are nearly impossible to diagnose after the fact.

This rule has no exceptions. "It's only two statements" or "they'll usually both succeed" is not a justification for
skipping the transaction.

When summarizing completed work, explicitly list any new or modified repository methods that use transactions so the
user can verify the transaction boundaries are correct.

### MANDATORY: Rollback Tests for Every Transaction

Every repository method that uses a transaction **must** have a corresponding rollback test that proves the transaction
actually rolls back all changes on failure. This test **must** live in the abstract contract test class so it runs
against both the in-memory and the SQL implementation — proving they are functionally equivalent.

The rollback test pattern is:

1. Set up valid state
2. Trigger the transactional method in a way that causes it to fail partway through (e.g., via a spy that throws on the
   second write)
3. Assert that **all** state remains unchanged — no partial writes survived

If the in-memory implementation does not roll back identically to the SQL implementation, it is a bug in the in-memory
implementation, not a test to skip.

**How to inject the failure (house technique).** Pass a Mockito `spy(...)` of a *domain input* whose accessor throws,
and let the production method fail on it mid-transaction. Mockito here can spy records, so an id or value object works:

```java

@Test
void should_roll_back_when_a_batch_delete_fails_partway() {
  SomeId second = new SomeId("second");
  SomeId poisoned = spy(second);
  RuntimeException failure = new RuntimeException("Oops");
  when(poisoned.id()).thenThrow(failure);
  SomeId first = new SomeId("first");
  List<SomeId> batch = List.of(first, poisoned);   // first write commits, second throws

  assertThatThrownBy(() ->
          repository.delete(batch)
  )
          .hasMessageContaining("Oops");

  List<Some> actual = repository.findAll();
  assertThat(actual).containsExactlyInAnyOrder(/* everything still there */);
}
```

Canonical: `orders/OrderRepositoryTest#should_roll_back_when_a_batch_delete_fails_partway`.
This only works if the production method reads the poisoned accessor *while iterating inside the transaction* — a method
that pre-computes all its bind values before `useTransaction` has no poisonable mid-point, so structure it to iterate
the inputs inside the transaction (read `id.id()` per item in the loop) when a rollback test must be possible.

### MANDATORY: SQL Repository DAO Access

Every JDBI DAO must be registered as a `get…Dao()` method on `DaoProvider`, implemented in
`JdbiDaoProvider` via `onDemand(...)`. Single-statement reads access the on-demand DAO directly:

```java
SomeDao dao = daoProvider.getSomeDao();
return dao.findAll();
```

Multi-statement writes wrap the work in a transaction and attach the DAO to the handle inside the
lambda:

```java
jdbi.useTransaction(handle -> {
    SomeDao dao = handle.attach(SomeDao.class);
    dao.firstWrite();
    dao.secondWrite();
});
```

Do not invent ad-hoc `jdbi.withHandle(handle -> handle.attach(...))` wrappers for reads. Register
the DAO in `DaoProvider` and access it on-demand.

### MANDATORY: Guard @BindList Against Empty Lists

Every repository method that passes a list to a JDBI `@BindList` DAO call must guard against an empty list before
calling the DAO. JDBI's `@BindList` throws `IllegalArgumentException` on an empty list by default. Return an empty
collection (or zero, for count methods) early, before acquiring the DAO:

```java
List<String> itemNames = items.stream()
        .map(ItemName::getName)
        .toList();

if (itemNames.isEmpty()) {
    return List.of();
}

ItemDAO dao = daoProvider.getItemDAO();
return dao.getCountsPerItem(itemNames);
```

The guard belongs in the repository implementation (`SqlFooRepository`), not the DAO or the service. This is the same
boundary where domain types are converted to the strings/ints the DAO expects. Canonical examples:
`SqlOrderRepository`, `SqlInvoiceRepository`.

A contract test for the empty-list case belongs in the abstract repository test so it runs against both the in-memory
and SQL implementations.

### SQL Compatibility

All SQL must work on both H2 (tests) and PostgreSQL (production). There are no exceptions to this rule.

When proposing or writing SQL:

- Never use H2-only syntax (e.g., `MERGE INTO ... KEY ...`).
- Never use PostgreSQL-only syntax (e.g., `DISTINCT ON`).
- Always verify that the chosen SQL dialect is supported by both databases.
- When in doubt, prefer standard SQL or `INSERT ... ON CONFLICT` (supported by both H2 in PostgreSQL mode and
  PostgreSQL).

### Nullable numbers in row mappers

`ResultSet.getLong` and `getInt` answer `0` for a NULL column, and `getBoolean` answers `false`. When the column is
nullable and zero is also a real value, call `wasNull()` straight after the read and map the NULL to `null`:

```java
long amount = r.getLong("discount_amount");
boolean missing = r.wasNull();
if (missing) {
  return null;
}
return new Discount(amount);
```

An in-memory repository double cannot show this, since it hands back the object it was given. A round-trip test for the
missing value belongs in the shared abstract repository test, where the SQL implementation has to satisfy it too.

## Code Style

- When a method needs to exist for the code to compile but its implementation is not yet known (whether demanded by an
  interface, a caller, or any other dependency), the method body must throw `RuntimeException("not yet implemented")`.
  This lets the code compile while ensuring the missing implementation is never silently ignored at runtime
- Never swallow exceptions. If a catch block cannot handle the exception meaningfully, wrap it in a RuntimeException and
  rethrow. Silent `catch` blocks hide bugs
- Single abstraction level per method; extract low-level details into private helpers
- Always use braces around `if` statement bodies, even for one-liners
- Prefer conditional returns over switch statements
- Never use ternary operators; use if/else blocks instead — zero tolerance, no exceptions
- Never inline `new` expressions or method calls as constructor or method arguments. Assign them to a variable first,
  then pass the variable. This makes intermediate values visible when debugging, and it applies to all code, production
  and test. For example, write
  `OrderRepository orderRepository = new InMemoryOrderRepository(); new Service(orderRepository);` not
  `new Service(new InMemoryOrderRepository())`, and
  `FieldName name = eventPayload.getName(); Checksum checksum = eventPayload.getChecksum();`, then
  `new FieldWithChecksum(name, checksum);`, not
  `new FieldWithChecksum(eventPayload.getName(), eventPayload.getChecksum())`
- Do not abbreviate variable, parameter, or method names. Readability matters more than brevity — typing is not the
  bottleneck, reading is. For example, write `orderService` not `svc`, `orderRepository` not `repo`
- Use text blocks (`"""`) for SQL strings. For example, write `"""
  SELECT * FROM users
  WHERE id = :id
  """` not `"SELECT * FROM users WHERE id = :id"`
- Never use `System.out.println` or `System.err.println` in production sources (`src/main/`). Use SLF4J:
  `private static final Logger logger = LoggerFactory.getLogger(MyClass.class);` and pick the level: `debug` for noisy
  diagnostics, `info` for boot or lifecycle events, `warn` for recoverable problems, `error` for failures. Never log
  PII (full names, email addresses, full payloads, raw user input) or raw SQL fragments.
- Log the exception, not its message: `logger.warn("Failed to purge export {}", exportId, e)`, never
  `logger.warn(e.getMessage())`. Database exceptions can carry the SQL and the bound values in their message, and
  redacting them by exception type only works when the exception itself is passed. Do not log an exception and rethrow
  it either; it gets logged where it is finally handled.

### Virtual Threads by Default

Always use virtual threads (`Thread.ofVirtual()`) when creating threads. Platform threads (`new Thread(...)` or
`Thread.ofPlatform()`) are exceptions that require explicit justification. If you find yourself reaching for a platform
thread, question why and confirm with the team before proceeding.

### No Reflection

Reflection (`java.lang.reflect`, `Class.forName`, `Method.invoke`, etc.) is **never allowed** in backend code. If a
circular dependency exists, fix the dependency graph — extract an interface, introduce a new service, or restructure
the packages. Reflection is never an acceptable workaround for a design problem.

## Conventions

- Declare variables using the interface type, not the concrete class. For example, use
  `OrderRepository orderRepository = InMemoryOrderRepository.inMemoryOrderRepository(userService);` not
  `InMemoryOrderRepository orderRepository = ...`
- Controllers should extend BaseController to keep the web framework out of business logic
- Services never accept or return bare language types on their public API. Every **public** parameter and return type
  must be a domain value object (a record wrapping the data). "Bare language type" means primitives (`int`, `double`,
  `boolean`, …), **their boxed wrappers (`Integer`, `Long`, `Double`, `Boolean`, …)**, and `String` — none of these
  count as a domain type. For example, a service method should accept `ExternalApiUrl` not `String apiUrl`, and return
  `ReserveDays` not `int`. Private helpers and locals inside the method may use bare types when it is simpler; domain
  types there are welcome where they clarify, but not required.
- Domain model records must not contain primitives (`String`, `int`, `boolean`, etc.). Every field must be a domain
  value object (a record wrapping the primitive). For example, use `CourseName` not `String name`, `Rating`
  not `int value`, `AttendanceDate` (wrapping `LocalDate`) not `String date`. This applies to all model records — the
  only place primitives appear is inside the leaf value-object records themselves
- Services never return request or response objects. Services return domain model objects (records with domain types).
  Response/request objects are controller concerns — the controller converts between domain objects and HTTP DTOs
- Request records must have a `toModel()` method that converts to the corresponding domain object. Response records must
  have a static `fromModel(DomainType)` factory method that converts from the domain object. Controllers use these
  methods exclusively — never construct domain objects or response records by hand in the controller
- Services call other services, not repositories directly (except their own)
- Controllers live in a versioned package, v1, v2, etc.
- Response classes don't need visibility outside their package
- Services live in the root package of the specific functionality
- Domain objects (records, value objects) live in the feature's root package — they are the model
- Immutable data uses Java records

## Testing Strategy

- JUnit 5 unit tests
- Cucumber BDD tests, tagged per layer (e.g. `@unit`, `@integration`)
- Feature files in `src/test/resources/com/example/*.feature`
- In-memory repository implementations for fast isolated tests

Outside-in and the mock endstate live in [tdd.md](tdd.md).

### Test conventions

- **AAA formatting:** Separate Arrange / Act / Assert with blank lines. Name the asserted variable `actual`, never
  `result`:
  ```java
  @Test
  void should_return_empty_list_when_no_users() {
      User user = anExistingUser();

      CandidateUsersResponse response = controller.handle(user);

      List<CandidateUser> actual = response.users();
      assertThat(actual).isEmpty();
  }
  ```
- Test naming: `should_<action>_<scenario>`
- Use AssertJ for assertions (`assertThat`)
- When asserting exceptions, use `assertThatThrownBy` with the lambda on a separate line so the executed statement is
  clearly visible, and always verify the exception message:
  ```java
  assertThatThrownBy(() ->
          orderService.cancel(orderId)
  )
          .isInstanceOf(OrderAlreadyCancelledException.class)
          .hasMessageContaining("already cancelled");
  ```
  Match the project formatter's output (IntelliJ IDEA's default), so write it this way from the start.
  This also applies when the exception is only the route to a state assertion, as in rollback tests: never
  `catchThrowable` or a try/catch in a test. Assert the throw with `assertThatThrownBy`, then assert the state after it.
- When proposing Gherkin scenarios, always present the Java step definitions alongside the feature file. The reviewer
  must be able to see where each step connects to the system under test (which service, repository, or adapter is
  called). A Gherkin scenario without visible step wiring is not reviewable.

### MANDATORY: Parameterised Value-Object Validation

Value-object validation tests must cover the full set of invalid inputs, not a single representative case. Use
`@ParameterizedTest` with `@NullAndEmptySource` and `@ValueSource` for blank/null/empty/whitespace rejection, rather
than asserting on one example. A blank-rejection test should exercise `null`, `""`, and whitespace (`"  "`, `"\t"`) in
one parameterized method. Length caps stay as explicit boundary `@Test`s (just over the limit, and — when it matters —
exactly at the limit). This is the concrete realization of the [testing.md](testing.md) "missing or null inputs, empty
strings" edge-case rule at the value-object boundary.

```java
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"   ", "\t"})
void should_reject_a_blank_name(String blank) {
    assertThatThrownBy(() ->
            new FolderName(blank)
    )
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("mandatory");
}
```

### MANDATORY: Test Models and Fixtures

Production models live in `src/main/resources/com/example/model/`. `ModelCatalogue` finds them by
scanning the classpath, so a new model needs no registration. Never add a hand-written list of models.

Test models live in two folders under `src/test/resources/com/example/model/`, one for each way a test uses a
model:

- `testmodel/` holds test models. A test that needs a model by its `ModelId` puts its model here and gets it from a
  plain `new ModelService()`. Production code that creates its own `ModelService` or `ModelRepository` finds it too.
  A test model must parse, its file name must be its declared name with dashes as dots plus the version
  (`annual-report` 0 is `annual.report.0.json`), and no other model in either folder may declare the same name and
  version. The catalogue fails every test that touches it if two models do.
- `fixture/` holds files a test parses by name: deliberately broken models, models that share a name, models that do
  not match their file name. Read them with `ModelUtil`. Production code never sees them.

Never mock `ModelService` only to make a model findable, and never put a test model in `model/`.

A test model that cannot be found usually fails with an `IllegalStateException` from the "not found" fallback in
`ModelService`, not with an error naming the model. Check the folder and the file name first.

### In-memory repository pattern

Every repository interface should have:

- A production implementation (e.g. SqlUserRepository)
- An InMemoryUserRepository used in all feature/unit tests
- A single integration test that runs the same contract tests against the real implementation to prove they behave
  identically

In-memory stubs must be functionally equivalent to the SQL implementation. If the SQL implementation supports
transactions (rollback on failure), the in-memory implementation must too — e.g., take a backup before the operation and
restore it on exception. Transaction rollback tests belong in the abstract contract test, not just the SQL subclass.

### Contract-test skeleton

Canonical: `orders/OrderRepositoryTest.java` with its `InMemory…` and `Sql…` subclasses. The
abstract class holds the `repository` field (declared as the interface type) and every `@Test`. Behaviour that genuinely
differs between the two backings is exposed as a `protected abstract` template-method hook the subclass implements (for
example, force-ageing a row, or holding a row lock). Each subclass assigns `repository` in its constructor; the SQL
subclass wipes first:

```java
public abstract class FooRepositoryTest {
  FooRepository repository;

  @Test
  void should_save_and_find_a_foo() {
    Foo foo = aFoo();

    repository.save(foo);

    FooId id = foo.id();
    Foo actual = repository.findById(id);
    assertThat(actual).isEqualTo(foo);
  }

  // Only when behaviour must differ between in-memory and SQL:
  protected abstract void forceSomeBackingSpecificState(FooId id);
}

public class InMemoryFooRepositoryTest extends FooRepositoryTest {
  public InMemoryFooRepositoryTest() {
    repository = new InMemoryFooRepository();
  }
}

public class SqlFooRepositoryTest extends FooRepositoryTest {
  public SqlFooRepositoryTest() {
    JdbiDaoProvider daoProvider = new JdbiDaoProvider();
    TestDatabase.clear(daoProvider);
    repository = new SqlFooRepository(daoProvider);
  }
}
```

Build the in-memory backing and the abstract tests first; the SQL subclass then becomes mechanical — assign the real
repository and implement any hooks. Do not duplicate the `@Test` methods per subclass.

### Seeding a permission in a controller test

A controller declares its permission with `requirePermission(permission)`. The matching test seeds that permission
through `PermissionService.create(...)` and asserts on `controller.getRequiredPermissions()`:

```java
Permission permission = new Permission("manage-orders");
permissionService.create(permission);
```

Every controller test must verify the required permissions via `getRequiredPermissions()` (see the existing
`GetOrderTreeControllerTest`).

### MANDATORY: A controller test covers security and wiring

A controller test covers security and wiring. That means the permissions it requires, refusals that must reach the
caller, and one happy path proving the controller reaches the service.

Wiring is whatever the controller does itself, on either side of the service call: reading path and query parameters,
the request record's `toModel()`, the status code the controller sets, and file handling. Those are tested at the
controller.

Figures, empty cases and edge cases are not. They go in the service test for what the model holds, and in the response
test for what `fromModel` does with it. When a behaviour spans both, **write both halves**. A behaviour that has only
ever been proven through the controller loses its only test the moment that test is split, so the service half and the
response half both have to exist before the controller test is deleted.

"The failure only shows in the transform" is a reason for a response test, not a reason for a controller test.
