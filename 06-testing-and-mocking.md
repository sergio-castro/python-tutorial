# Testing and Mocking

Python's testing story is unusual in one respect: the tool everyone uses is *not* the one in
the standard library. This chapter covers what modern Python projects actually do.

## The Landscape

| Tool | What it is | Should you use it? |
|-------------------|-------------------------------------------------|-------------------------|
| `unittest` | Stdlib, class-based, a direct JUnit descendant | Only in legacy code |
| **`pytest`** | Third-party, function-based, the de facto standard | **Yes** |
| `nose` / `nose2` | Older runners | No — unmaintained |

`unittest` is not deprecated and never will be — it ships with Python. But essentially every
modern project uses **pytest**, because it needs far less ceremony. Conveniently, pytest also
*runs* `unittest` test classes, so a migration can be gradual.

For mocking, the situation is the reverse: the standard library's `unittest.mock` **is** the
answer. If you are coming from Java looking for "the Mockito of Python," this is it. There is
no dominant third-party mocking framework — instead there is `pytest-mock`, a thin wrapper
that makes `unittest.mock` pleasant to use from pytest.

## Setup

```bash
uv add --dev pytest pytest-mock pytest-cov
```

This writes to `[dependency-groups]` — the contributor-only section from
[Project Configuration](./03-project-configuration.md), not an extra, because consumers of
your package should never be forced to install your test tools:

```toml
[dependency-groups]
dev = ["pytest>=8.0", "pytest-mock>=3.14", "pytest-cov>=6.0"]
```

Configure pytest in the same `pyproject.toml`. This is another `[tool.*]` block, read only by
pytest:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --import-mode=importlib"
```

- **`testpaths`** — where to look, so a bare `pytest` doesn't crawl your whole tree.
- **`-ra`** — print a summary of everything that wasn't a plain pass.
- **`--strict-markers`** — fail on unknown `@pytest.mark.foo`, so typo'd markers don't
  silently do nothing.
- **`--import-mode=importlib`** — the modern import mode. Recommended for `src/` layouts and
  it removes the need for `__init__.py` files in your test directories.

## Layout

Tests live in a top-level `tests/` directory, outside `src/`:

```
my-app/
├── src/
│   └── my_app/
│       ├── __init__.py
│       ├── orders.py
│       └── pricing.py
├── tests/
│   ├── conftest.py          # shared fixtures (auto-discovered, never imported)
│   └── test_orders.py
└── pyproject.toml
```

Because of the `src/` layout, your tests import the **installed** package
(`from my_app.orders import ...`), not the files next door. `uv sync` installs your project
in editable mode, so your edits are picked up immediately — but anything you forgot to
package will fail loudly in tests rather than silently working.

Discovery is convention-based: files named `test_*.py`, functions named `test_*`, and classes
named `Test*` (with no `__init__` method).

## Your First Test

Two modules to test against, used throughout this chapter:

```python
# src/my_app/pricing.py
import requests

def fetch_rate(currency: str) -> float:
    response = requests.get(f"https://api.example.com/rates/{currency}", timeout=10)
    response.raise_for_status()
    return response.json()["rate"]
```

```python
# src/my_app/orders.py
from my_app.pricing import fetch_rate

def line_total(unit_price: int, quantity: int) -> int:
    if quantity < 0:
        raise ValueError("quantity cannot be negative")
    return unit_price * quantity

def order_total(amount: float, currency: str) -> float:
    return round(amount * fetch_rate(currency), 2)
```

And the test:

```python
# tests/test_orders.py
from my_app.orders import line_total

def test_line_total_multiplies_price_by_quantity():
    assert line_total(250, 3) == 750
```

No class, no inheritance, no `self`, no `assertEqual`. Just a function and the plain `assert`
statement.

### Why Plain `assert` Is Enough

In JUnit you need `assertEquals(expected, actual)` because a bare `assert` would only tell you
*that* it failed. pytest rewrites your test module's bytecode at import time to instrument
every assertion, so it can reconstruct the whole expression on failure:

```
    def test_line_total_multiplies_price_by_quantity():
>       assert line_total(250, 3) == 700
E       assert 750 == 700
E        +  where 750 = line_total(250, 3)
```

This is why Python has no equivalent of AssertJ or Hamcrest in common use — the language's
own operators plus this rewriting cover it.

### Expecting Exceptions

```python
import pytest
from my_app.orders import line_total

def test_line_total_rejects_negative_quantity():
    with pytest.raises(ValueError, match="cannot be negative"):
        line_total(250, -1)
```

`match` is a **regular expression** searched against the message — a substring works, but
remember that characters like `.`, `(`, and `$` are regex metacharacters.

For floating-point comparisons, use `pytest.approx`:

```python
assert order_total(10.0, "EUR") == pytest.approx(11.53)
```

## Fixtures: Setup Without Inheritance

A fixture is a function that produces something a test needs. Tests request fixtures **by
naming them as parameters**:

```python
import pytest

@pytest.fixture
def order():
    return {"amount": 100.0, "currency": "EUR"}

def test_order_has_currency(order):        # requests the fixture by name
    assert order["currency"] == "EUR"
```

The mental shift from JUnit: `@BeforeEach` runs for *every* test in the class whether it needs
the setup or not. A pytest fixture runs only for tests that ask for it. Each test declares its
own dependencies — it's dependency injection, not inheritance.

For teardown, `yield` instead of `return`. Everything after the `yield` runs when the test
finishes, pass or fail:

```python
@pytest.fixture
def db_connection():
    conn = connect("postgres://localhost/test")
    yield conn
    conn.close()                            # teardown
```

Fixtures can request other fixtures, and `scope` controls how often they rebuild:

```python
@pytest.fixture(scope="session")            # once per test run
def docker_postgres():
    ...
```

Scopes are `function` (the default), `class`, `module`, `package`, and `session`. Widen the
scope only for genuinely expensive setup — shared mutable state between tests is a classic
source of order-dependent failures.

Fixtures defined in `tests/conftest.py` are available to every test file in that directory
and below, with no import. Do not import `conftest.py` yourself; pytest loads it.

### Built-in Fixtures Worth Knowing

| Fixture | What it gives you |
|--------------|--------------------------------------------------------------------|
| `tmp_path` | A unique temporary `pathlib.Path` directory, cleaned up for you |
| `monkeypatch`| Set env vars / attributes, auto-reverted after the test |
| `capsys` | Captures `stdout` / `stderr` so you can assert on printed output |
| `caplog` | Captures log records |
| `request` | Metadata about the running test (used for parameterized fixtures) |

## Parameterized Tests

One test, many inputs — the equivalent of JUnit's `@ParameterizedTest`:

```python
@pytest.mark.parametrize(
    "unit_price,quantity,expected",
    [
        (250, 3, 750),
        (0, 5, 0),
        (250, 0, 0),
    ],
)
def test_line_total(unit_price, quantity, expected):
    assert line_total(unit_price, quantity) == expected
```

That reports as three separate tests. Use `pytest.param` when you want a readable id or a
per-case marker:

```python
@pytest.mark.parametrize(
    "currency",
    [
        "EUR",
        pytest.param("XXX", marks=pytest.mark.xfail(reason="unsupported currency")),
    ],
    ids=["euro", "unknown"],
)
def test_currencies(currency): ...
```

Prefer this over a `for` loop inside one test: a loop stops at the first failure and reports
one result, while parametrize runs every case and names the ones that broke.

## Testing Classes

Everything so far applies to functions. The [previous chapter](./05-object-oriented-python.md)
covered Python's object model — here is how those features change the way you write tests.

### Dataclasses Make Assertions One Line

A `@dataclass` generates `__eq__` that compares **field values**, not identity. That turns a
pile of field-by-field assertions into a single one:

```python
from dataclasses import dataclass

@dataclass
class Receipt:
    order_id: str
    total: int
    currency: str
```

```python
def test_receipt():
    got = build_receipt(order_id="A-1", amount=750)

    # Instead of three assertions that stop at the first failure:
    assert got == Receipt(order_id="A-1", total=750, currency="EUR")
```

When it fails, pytest diffs the two objects and shows you exactly which field differs — you
get the full picture in one run rather than fixing one field at a time.

This is worth knowing precisely because the opposite is a silent trap. A **plain** class
without `__eq__` falls back to identity comparison, so two objects with identical contents are
never equal:

```python
class Receipt:                                  # no @dataclass
    def __init__(self, order_id, total):
        self.order_id, self.total = order_id, total

assert Receipt("A-1", 750) == Receipt("A-1", 750)   # fails — different objects
```

The failure message is baffling the first time, because both sides print the same values. If
you want value equality, use `@dataclass` (or write `__eq__`).

### Build Variants with `dataclasses.replace`

Rather than a fixture per scenario, define one valid baseline and derive from it. `replace`
returns a **new** object with some fields swapped, which works on `frozen=True` dataclasses
too:

```python
from dataclasses import replace

@pytest.fixture
def config():
    return ConnectionConfig(endpoint="https://test", username="admin", password="secret")

def test_rejects_empty_username(config):
    with pytest.raises(ValueError):
        connect(replace(config, username=""))
```

The test states only what makes it different from a valid baseline, so adding a required field
to `ConnectionConfig` later means updating one fixture instead of thirty literals.

### Grouping Tests in Classes

pytest collects methods on classes named `Test*`. This is purely organizational — no
inheritance, no base class, and **no `__init__` method** (pytest skips classes that define
one):

```python
class TestGraphService:
    @pytest.fixture
    def service(self):
        return GraphService(endpoint="https://test")

    def test_queries_the_endpoint(self, service):
        assert "https://test" in service.get_graph("q")

    def test_rejects_an_empty_query(self, service):
        with pytest.raises(ValueError):
            service.get_graph("")
```

A fixture defined inside the class is visible only to that class — useful for narrowing scope
without adding it to `conftest.py`. Don't store state on `self` between test methods, though:
pytest creates a **new instance for every test**, so anything you set in one method is gone in
the next. Share setup through fixtures, not attributes.

## Mocking

### The Vocabulary

`unittest.mock` gives you a handful of pieces:

| Object | Purpose |
|-------------------|-----------------------------------------------------------------|
| `Mock` | Records calls; returns a new `Mock` for any attribute you touch |
| `MagicMock` | A `Mock` that also supports dunder protocols (`len()`, `in`, …) |
| `AsyncMock` | A `Mock` whose calls return awaitables |
| `patch` | Temporarily replaces an object at a given import path |
| `create_autospec`| Builds a mock shaped like a real object's API |

The default `Mock` is extremely permissive. Every attribute access invents a new child mock,
and every call is accepted:

```python
m = Mock()
m.fetch_rate("EUR", "nonsense", bogus=True)   # accepted
m.fetch_raet("EUR")                            # typo — also accepted
```

That permissiveness is the single biggest source of tests that pass while the code is broken.
The fix is `autospec`, covered below.

### Use the `mocker` Fixture

`pytest-mock` exposes everything through a `mocker` fixture that undoes every patch when the
test ends:

```python
from my_app.orders import order_total

def test_order_total_applies_the_rate(mocker):
    fetch = mocker.patch("my_app.orders.fetch_rate", return_value=1.5)

    assert order_total(10.0, "EUR") == 15.0
    fetch.assert_called_once_with("EUR")
```

The stdlib alternative is a decorator, and it has a trap — stacked `@patch` decorators inject
arguments **bottom-up**:

```python
@patch("my_app.orders.log_event")          # ← second argument
@patch("my_app.orders.fetch_rate")         # ← first argument
def test_order_total(mock_fetch_rate, mock_log_event):
    ...
```

Get that order backwards and you will silently assert against the wrong mock. `mocker` has no
such ordering, so prefer it in pytest projects.

### The Golden Rule: Patch Where It's *Used*

This is the mistake everyone makes once. `orders.py` does
`from my_app.pricing import fetch_rate`, which **copies the reference** into the `orders`
module namespace at import time. Patching the original module rebinds
`pricing.fetch_rate`, but `orders.fetch_rate` still points at the function object it grabbed
earlier:

```python
mocker.patch("my_app.pricing.fetch_rate", return_value=1.5)   # ✗ no effect on orders
mocker.patch("my_app.orders.fetch_rate",  return_value=1.5)   # ✓ correct
```

The rule follows from *how* the code under test imports the dependency:

| Import style in `orders.py` | What to patch |
|--------------------------------------------|----------------------------------|
| `from my_app.pricing import fetch_rate` | `my_app.orders.fetch_rate` |
| `from my_app import pricing` → `pricing.fetch_rate(...)` | `my_app.pricing.fetch_rate` |
| `import requests` → `requests.get(...)` | `my_app.pricing.requests.get` |

In the second and third cases the attribute is looked up on the module object *at call time*,
so patching the source module works. In the first case the lookup already happened.

A useful diagnostic: if your mock reports zero calls but the real code clearly ran, you almost
certainly patched the definition instead of the usage.

### Always Autospec

`autospec=True` shapes the mock like the real object: the signature is enforced and
non-existent attributes raise `AttributeError`.

```python
def test_signature_is_enforced(mocker):
    fetch = mocker.patch("my_app.orders.fetch_rate", autospec=True, return_value=1.5)

    order_total(10.0, "EUR")

    fetch.assert_called_once_with("EUR")
```

Now the mistakes from earlier fail loudly:

```python
fetch("EUR", "nonsense")    # TypeError: too many positional arguments
fetch.fetch_raet            # AttributeError
```

This matters most over time. Without autospec, renaming a parameter in `fetch_rate` leaves
every test green while production breaks — the mock happily accepts the old call. With
autospec, the tests fail the moment the real signature moves. Treat `autospec=True` as the
default and omitting it as the thing that needs a justification.

`mocker.patch.object(SomeClass, "method", autospec=True)` does the same for a single
attribute, and `create_autospec(SomeClass)` builds a standalone specced mock without patching
anything.

One caveat: autospec inspects the class, so attributes created dynamically in `__init__` are
not part of the spec. For those, use `mocker.patch.object` on the instance.

### Mocking Classes and Their Members

Patching a single method is the common case, and `patch.object` targets it directly:

```python
def test_get_graph_is_stubbed(mocker):
    mocker.patch.object(GraphService, "get_graph", autospec=True, return_value="stubbed")

    assert GraphService("https://real").get_graph("q") == "stubbed"
```

With `autospec=True` the mock receives `self` as its first argument, exactly like the real
method — so `assert_called_once_with(service, "q")` includes the instance. Without autospec it
doesn't, which is a frequent source of confusing assertion failures.

#### Patching a Whole Class

When the code under test constructs its own collaborator, you patch the class. The thing to
internalize: **the object your code receives is `MockClass.return_value`**, because that is
what calling the class returns.

```python
# src/my_app/reports.py
from my_app.services import GraphService

def run(endpoint: str) -> str:
    return GraphService(endpoint).get_graph("q")
```

```python
def test_run(mocker):
    MockService = mocker.patch("my_app.reports.GraphService", autospec=True)
    MockService.return_value.get_graph.return_value = "faked"

    assert run("https://x") == "faked"

    MockService.assert_called_once_with("https://x")           # the constructor call
    MockService.return_value.get_graph.assert_called_once_with("q")   # the method call
```

Setting `MockService.return_value` gets you the instance; asserting on `MockService` itself
checks how it was **constructed**. Mixing the two up is the usual reason a test insists a
method was never called.

#### Properties

A `@property` is a class-level descriptor, so assigning to it on a real instance fails. Patch
it on the class with `PropertyMock`:

```python
def test_endpoint(mocker):
    endpoint = mocker.patch.object(
        GraphService, "endpoint", new_callable=mocker.PropertyMock
    )
    endpoint.return_value = "https://faked"

    assert GraphService("https://real").endpoint == "https://faked"
```

On an autospec **instance** mock the rule is different and simpler — the property is just an
attribute there, so plain assignment works:

```python
service = mocker.create_autospec(GraphService, instance=True)
service.endpoint = "https://faked"
```

#### Standalone Doubles with `create_autospec`

You don't have to patch anything to get a correctly shaped object. `create_autospec` builds a
double you can pass in, which pairs well with dependency injection:

```python
service = mocker.create_autospec(GraphService, instance=True)
service.get_graph.return_value = "stubbed"

assert build_report(service) == "stubbed"
service.get_graph.assert_called_once_with("q")
```

`instance=True` says "this is an instance, not the class," so calling it like a constructor
raises `TypeError` instead of quietly handing back another mock.

This also solves abstract base classes. `BaseService(ABC)` can't be instantiated, and writing a
throwaway subclass per test is noise:

```python
double = mocker.create_autospec(BaseService, instance=True)
double.execute.return_value = "stubbed"
```

Add `spec_set=True` when you want assignment locked down too — it rejects setting attributes
the real class doesn't have, catching typos like `service.get_grpah = ...` that a normal mock
would happily accept.

#### Context Managers and Other Dunders

`Mock` does not implement dunder methods; `MagicMock` does. That is the whole practical
difference between them, and it matters as soon as your code uses `with`, `len()`, `in`, or
iteration:

```python
def test_uses_a_connection(mocker):
    conn = mocker.MagicMock()
    conn.__enter__.return_value.execute.return_value = ["row"]

    assert fetch_rows(conn) == ["row"]
```

`mocker.patch` gives you a `MagicMock` by default, so this usually just works — but a bare
`Mock()` you construct yourself will fail with a `TypeError` about the context manager
protocol.

### Asserting on Calls

```python
fetch.assert_called_once_with("EUR")
fetch.assert_called_with("EUR")            # the *most recent* call only
fetch.assert_any_call("USD")
fetch.assert_not_called()
fetch.assert_has_calls([call("EUR"), call("USD")])

assert fetch.call_count == 2
assert fetch.call_args.args == ("EUR",)
assert fetch.call_args.kwargs == {}
assert fetch.call_args_list == [call("EUR"), call("USD")]
```

Modern Python guards against the most dangerous typo here: any attribute beginning with
`assert`, `assret`, `asert`, `aseert`, or `assrt` raises `AttributeError` rather than
inventing a child mock that is always truthy. `assert_called_onec_with(...)` therefore fails
properly. That guard only covers the `assert` prefix, though — `fetch.was_called_with(...)`
still silently passes.

### Async Code

`patch` detects `async def` targets and gives you a mock that must be awaited — no extra
configuration, and it works with `autospec=True` too:

```python
import pytest

@pytest.mark.asyncio                       # requires pytest-asyncio
async def test_async_fetch(mocker):
    fetch = mocker.patch("my_app.orders.fetch_rate_async", autospec=True, return_value=1.5)

    assert await order_total_async(10.0, "EUR") == 15.0
    fetch.assert_awaited_once_with("EUR")
```

Note `assert_awaited_once_with` — `assert_called_once_with` only proves the coroutine was
*created*, not that anything awaited it.

### `monkeypatch` for Values, Mocks for Behaviour

When you just need to set something rather than observe calls, pytest's built-in
`monkeypatch` fixture is simpler and reverts automatically:

```python
def test_reads_api_key(monkeypatch, tmp_path):
    monkeypatch.setenv("CLAUDE_API_KEY", "test-key")
    monkeypatch.delenv("HTTP_PROXY", raising=False)
    monkeypatch.chdir(tmp_path)

    assert load_config().api_key == "test-key"
```

Rule of thumb: `monkeypatch` for environment and configuration, `mocker` when you need to
assert on how something was called.

## When *Not* to Mock

Every mock is a duplicate of an assumption about how a collaborator behaves, and duplicates
drift. State-of-the-art practice is to mock as little as possible.

**1. Pure functions need no mocks.** Push logic out of the I/O path and test it directly.
`line_total` above needs no test doubles at all; `order_total` does only because it reaches
out to the network.

**2. Prefer injection and fakes over patching.** Patching reaches into another module's
namespace — it works, but it couples the test to the code's import structure. Passing the
dependency in is cleaner:

```python
# src/my_app/orders.py
from typing import Protocol

class RateSource(Protocol):
    def fetch(self, currency: str) -> float: ...

def order_total(amount: float, currency: str, rates: RateSource) -> float:
    return round(amount * rates.fetch(currency), 2)
```

```python
# tests/test_orders.py
class FakeRates:
    def __init__(self, table): self._table = table
    def fetch(self, currency): return self._table[currency]

def test_order_total_applies_the_rate():
    assert order_total(10.0, "EUR", FakeRates({"EUR": 1.5})) == 15.0
```

No patching, no import-path strings, and the test survives any refactor that keeps the
behaviour.

The [OO chapter](./05-object-oriented-python.md) noted that Python has no `interface` keyword —
you use abstract classes or duck typing. `Protocol` is the third option, and the best one for
test seams: it names a shape without requiring anyone to inherit from it. `FakeRates` satisfies
`RateSource` purely by having a matching `fetch` method, so `mypy` checks the contract while
your test class stays a plain object. An `ABC` would force both the real implementation and
every fake to subclass it.

**3. Don't mock what you don't own.** Wrapping a third-party client in your own thin adapter
and faking the adapter means a library upgrade breaks one class, not fifty tests.

**4. Mock at the boundary, and prefer the real boundary.** Rather than patching your own
`fetch_rate`, intercept the HTTP call itself — that way your request-building code is actually
exercised:

```python
import responses

@responses.activate
def test_fetch_rate_parses_the_payload():
    responses.get("https://api.example.com/rates/EUR", json={"rate": 1.5})

    assert fetch_rate("EUR") == 1.5
```

The patched version proves nothing about the URL, the timeout, or the JSON shape. This version
proves all three.

## The Wider Toolbox

| Need | Reach for |
|-----------------------------------|--------------------------------------------|
| HTTP mocking (`requests`) | `responses`, `requests-mock` |
| HTTP mocking (`httpx`) | `respx`, `pytest-httpx` |
| Record & replay real traffic | `vcrpy` |
| Freezing / travelling in time | `time-machine` (fast), `freezegun` |
| Real Postgres, Kafka, Redis, … | `testcontainers` |
| Building test objects | `polyfactory`, `factory-boy` |
| Property-based testing | `hypothesis` |
| Snapshot testing | `syrupy` |
| Async tests | `pytest-asyncio`, `anyio` |
| Coverage | `pytest-cov` |
| Running tests in parallel | `pytest-xdist` |

Two are worth singling out. **`hypothesis`** generates inputs instead of you enumerating
them — you assert a property ("the total is never negative") and it hunts for a counterexample,
then shrinks it to the smallest failing case. **`testcontainers`** boots a real database in
Docker for the test session, which is frequently more honest than mocking a database driver.

## Running Tests

```bash
uv run pytest                                   # everything
uv run pytest tests/test_orders.py              # one file
uv run pytest tests/test_orders.py::test_total  # one test
uv run pytest -k "rate and not async"           # match by name expression
uv run pytest -x                                # stop at first failure
uv run pytest --lf                              # re-run last failures only
uv run pytest -q                                # quieter output
uv run pytest -n auto                           # parallel (needs pytest-xdist)
uv run pytest --cov=my_app --cov-report=term-missing
```

`--cov-report=term-missing` lists the line numbers that were never executed, which is the only
part of a coverage report that tells you what to do next. Treat coverage as a smoke detector:
a sharp drop is worth investigating, but a percentage target mostly teaches people to write
tests that execute code without asserting anything about it.

## Coming From Another Language

| Concept | Java | JavaScript (Jest) | Python |
|---------------------|--------------------------|---------------------|----------------------------------|
| Test runner | JUnit | Jest / Vitest | `pytest` |
| Assertions | `assertEquals`, AssertJ | `expect(...)` | plain `assert` |
| Setup / teardown | `@BeforeEach` | `beforeEach` | fixtures (`yield` for teardown) |
| Parameterized tests | `@ParameterizedTest` | `test.each` | `@pytest.mark.parametrize` |
| Mocking | Mockito | `jest.mock` | `unittest.mock` via `mocker` |
| Strict stubbing | Mockito strictness | — | `autospec=True` |
| Test discovery | annotations | `*.test.js` | `test_*.py` / `test_*` |

## Cheat Sheet

```bash
uv add --dev pytest pytest-mock pytest-cov    # install the toolchain
uv run pytest                                 # run everything
uv run pytest -x --lf                         # fix-a-failure loop
```

```python
import pytest

def test_something():                          # discovery: test_*.py, test_*
    assert value == expected                   # plain assert; pytest explains failures

with pytest.raises(ValueError, match="regex"): ...   # expected exception
assert result == pytest.approx(1.23)                 # float comparison

@pytest.fixture                                # setup; yield for teardown
def thing(): yield make_thing()

@pytest.mark.parametrize("a,expected", [(1, 2)])     # table-driven tests
def test_table(a, expected): ...

def test_with_mock(mocker):
    m = mocker.patch("module.where.it.is_USED", autospec=True, return_value=1.5)
    ...
    m.assert_called_once_with("EUR")

# --- classes ---
assert got == Receipt("A-1", 750)                     # dataclass __eq__ compares fields
mocker.patch.object(Cls, "method", autospec=True)     # one method
MockCls = mocker.patch("mod.Cls", autospec=True)      # whole class...
MockCls.return_value.method.return_value = "x"        # ...instance is .return_value
mocker.create_autospec(Cls, instance=True)            # standalone double, no patching
```

Three rules to carry away: **patch where the name is used, not where it's defined**; **always
pass `autospec=True`**; and **prefer a fake you inject over a mock you patch**.

---

Previous: [Object-Oriented Python](./05-object-oriented-python.md) |
Next: [Packaging and Deployment](./07-packaging-and-deployment.md)
