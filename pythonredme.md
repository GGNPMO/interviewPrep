# Python Interview Questions

A practical question bank with concise answers, grouped from fundamentals to advanced and design-level topics.

### Priority Guide

- **Critical**: Must-answer topics for almost every Python interview.
- **High**: Frequently tested topics, especially for backend and senior roles.
- **Medium**: Role-dependent topics that commonly distinguish stronger candidates.
- **Low**: Specialized or less frequently asked topics; review when relevant to the role.

## Table of Contents

1. [Core Language](#1-core-language)
2. [Data Structures](#2-data-structures)
3. [Functions and Functional Features](#3-functions-and-functional-features)
4. [Object-Oriented Python](#4-object-oriented-python)
5. [Iterators, Generators and Context Managers](#5-iterators-generators-and-context-managers)
6. [Memory, GIL and Concurrency](#6-memory-gil-and-concurrency)
7. [Error Handling and Testing](#7-error-handling-and-testing)
8. [Standard Library and Tooling](#8-standard-library-and-tooling)
9. [Performance](#9-performance)
10. [Architecture and Design (Senior Level)](#10-architecture-and-design-senior-level)
11. [Production Scenarios and Current Topics](#11-production-scenarios-and-current-topics)
12. [Coding Exercises](#12-coding-exercises)

---

## 1. Core Language

**Q1. [Critical] What is the difference between a list and a tuple?**
Lists are mutable; tuples are immutable and hashable (if their elements are), so tuples can be dict keys or set members. Tuples are slightly faster and signal "fixed record" intent.

**Q2. [Critical] What is the difference between `is` and `==`?**
`==` compares values (calls `__eq__`); `is` compares object identity. Use `is` for `None` checks: `if x is None`.

**Q3. [Critical] What are mutable default arguments and why are they dangerous?**
Defaults are evaluated once at function definition time.
```python
def add(item, bucket=[]):   # bug: shared across calls
    bucket.append(item)
    return bucket

def add(item, bucket=None): # fix
    bucket = [] if bucket is None else bucket
    bucket.append(item)
    return bucket
```

**Q4. [High] Explain Python's scoping rules (LEGB).**
Name lookup order: Local, Enclosing, Global, Built-in. Use `global` / `nonlocal` to rebind names in outer scopes.

**Q5. [High] What is the difference between shallow and deep copy?**
`copy.copy` copies the container but shares nested objects; `copy.deepcopy` recursively copies everything.

**Q6. [Critical] What are `*args` and `**kwargs`?**
`*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. They also unpack on calls: `f(*seq, **mapping)`.

**Q7. [Medium] What is the walrus operator?**
`:=` assigns within an expression (3.8+): `if (n := len(data)) > 10: ...`

**Q8. [High] How does Python handle integers and floats?**
Integers have arbitrary precision. Floats are IEEE 754 doubles, so `0.1 + 0.2 != 0.3`. Use `decimal.Decimal` or `fractions.Fraction` when exactness matters.

---

## 2. Data Structures

**Q9. [Critical] How do dicts work internally?**
Hash tables with open addressing. Average O(1) lookup/insert. Since 3.7, insertion order is guaranteed.

**Q10. [High] What makes an object hashable?**
It implements `__hash__` and `__eq__` consistently, and its hash never changes during its lifetime. Equal objects must have equal hashes.

**Q11. [High] List vs. `collections.deque`?**
`deque` gives O(1) appends/pops at both ends; list is O(n) for `insert(0, x)` / `pop(0)`.

**Q12. [High] When would you use `defaultdict`, `Counter`, `namedtuple`, `OrderedDict`?**
- `defaultdict`: grouping/accumulating without key checks
- `Counter`: frequency counting
- `namedtuple` / `dataclass`: lightweight records
- `OrderedDict`: order-sensitive equality or `move_to_end`

**Q13. [Critical] List comprehension vs. generator expression?**
`[x*x for x in data]` builds the whole list in memory; `(x*x for x in data)` is lazy and uses constant memory.

**Q14. [High] How do you remove duplicates while preserving order?**
```python
list(dict.fromkeys(items))
```

---

## 3. Functions and Functional Features

**Q15. [Medium] What are first-class functions?**
Functions are objects: they can be assigned, passed as arguments, returned, and stored in collections.

**Q16. [High] What is a closure?**
A function that retains references to variables from its enclosing scope after that scope has finished.

**Q17. [Critical] What is a decorator? Write one.**
```python
import functools, time

def timed(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            print(f"{fn.__name__} took {time.perf_counter() - start:.4f}s")
    return wrapper
```
Mention `functools.wraps` to preserve metadata.

**Q18. [High] What is `functools.lru_cache`?**
Memoization decorator with a bounded least-recently-used cache. Arguments must be hashable. `functools.cache` is the unbounded variant.

**Q19. [Medium] `map`, `filter`, `reduce` vs. comprehensions?**
Comprehensions are generally preferred for readability. `reduce` lives in `functools`.

**Q20. [High] What are positional-only and keyword-only parameters?**
```python
def f(a, b, /, c, *, d): ...
```
`a`, `b` positional-only; `d` keyword-only.

---

## 4. Object-Oriented Python

**Q21. [High] What are dunder methods? Name important ones.**
`__init__`, `__repr__`, `__str__`, `__eq__`, `__hash__`, `__len__`, `__iter__`, `__next__`, `__enter__`/`__exit__`, `__call__`, `__getitem__`.

**Q22. [Medium] `__new__` vs. `__init__`?**
`__new__` creates the instance (a static method returning the object); `__init__` initializes it.

**Q23. [High] What is MRO?**
Method Resolution Order, computed with C3 linearization. Inspect with `Class.__mro__`. `super()` follows the MRO, which is what makes cooperative multiple inheritance work.

**Q24. [Critical] `@staticmethod` vs. `@classmethod` vs. instance method?**
Instance methods receive `self`; class methods receive `cls` (good for alternative constructors); static methods receive neither.

**Q25. [High] What are `@property` and descriptors?**
`property` is a built-in descriptor providing managed attribute access. Descriptors implement `__get__`, `__set__`, `__delete__`.

**Q26. [Critical] What are `dataclasses`? When prefer them over a plain class?**
Auto-generate `__init__`, `__repr__`, `__eq__` (optionally ordering, immutability via `frozen=True`, memory savings via `slots=True`).

**Q27. [Medium] What are `__slots__`?**
Fixed attribute set stored without a per-instance `__dict__`, reducing memory and speeding attribute access.

**Q28. [High] ABC vs. Protocol?**
`abc.ABC` enforces nominal inheritance. `typing.Protocol` enables structural subtyping (duck typing checked statically).

**Q29. [Low] What is a metaclass?**
The class of a class (`type` by default). Used for class-creation hooks, registries, ORMs. Often replaceable by `__init_subclass__` or class decorators.

---

## 5. Iterators, Generators and Context Managers

**Q30. [Critical] Iterable vs. iterator?**
Iterable implements `__iter__`; iterator implements `__iter__` and `__next__` and is exhausted once consumed.

**Q31. [Critical] What does `yield` do?**
Turns a function into a generator that suspends and resumes, preserving local state. Supports `send()`, `throw()`, `close()` and `yield from`.

**Q32. [High] How do you write a context manager?**
```python
from contextlib import contextmanager

@contextmanager
def opened(path):
    f = open(path)
    try:
        yield f
    finally:
        f.close()
```
Or implement `__enter__` / `__exit__`.

**Q33. [Critical] What is `async`/`await`?**
Coroutines scheduled cooperatively on an event loop (`asyncio`). Best for I/O-bound concurrency with many connections.

---

## 6. Memory, GIL and Concurrency

**Q34. [High] How does Python manage memory?**
Reference counting plus a cyclic garbage collector (generational). Objects live in private heaps managed by the interpreter.

**Q35. [Critical] What is the GIL?**
The Global Interpreter Lock lets only one thread execute Python bytecode at a time in CPython. It limits CPU-bound threading but not I/O-bound threading. Recent CPython versions offer an optional free-threaded build; verify support for your version and dependencies.

**Q36. [Critical] Threads vs. processes vs. asyncio: when to use which?**
- I/O-bound, few tasks: `threading`
- I/O-bound, many connections: `asyncio`
- CPU-bound: `multiprocessing` / `concurrent.futures.ProcessPoolExecutor`, or native extensions (NumPy, Cython, Rust)

**Q37. [Low] What are weak references?**
`weakref` lets you reference an object without increasing its refcount; useful for caches and avoiding cycles.

**Q38. [High] How do you detect a memory leak?**
`tracemalloc`, `objgraph`, `gc.get_objects()`, heap profilers (memray). Common causes: unbounded caches, global registries, lingering references in closures, cycles with `__del__`.

---

## 7. Error Handling and Testing

**Q39. [Critical] Explain `try / except / else / finally`.**
`else` runs if no exception occurred; `finally` always runs. Catch specific exceptions; avoid bare `except:`.

**Q40. [High] How do you create custom exceptions and chain them?**
Subclass `Exception`. Use `raise NewError(...) from original` to preserve causality.

**Q41. [Critical] `pytest` vs. `unittest`?**
`pytest`: plain asserts, fixtures, parametrization, rich plugin ecosystem. `unittest`: stdlib, xUnit style.

**Q42. [High] How do you mock dependencies?**
`unittest.mock.patch` / `MagicMock`, or `pytest-mock`. Patch where the name is *looked up*, not where it is defined.

**Q43. [Critical] What is type hinting and how do you enforce it?**
Annotations are not enforced at runtime. Use `mypy` / `pyright` in CI. Runtime validation is done by libraries like Pydantic.

---

## 8. Standard Library and Tooling

**Q44. [Critical] Virtual environments and dependency management?**
`venv`, `pip`, `pip-tools`, Poetry, `uv`, with a lock file for reproducible builds.

**Q45. [High] What does `if __name__ == "__main__":` do?**
Runs code only when the file is executed directly, not when imported.

**Q46. [High] `pathlib` vs. `os.path`?**
`pathlib` is object-oriented and more readable: `Path("a") / "b.txt"`.

**Q47. [High] How do you handle JSON, CSV, dates?**
`json`, `csv`, `datetime` (prefer timezone-aware datetimes; `zoneinfo` for time zones).

**Q48. [Critical] Logging best practices?**
Use the `logging` module with module-level loggers, structured output, appropriate levels, and no `print` in libraries.

---

## 9. Performance

**Q49. [High] How do you profile Python code?**
`cProfile`, `py-spy`, `line_profiler`, `timeit`. Measure first, then optimize hot paths.

**Q50. [High] Common optimization techniques?**
Better algorithms and data structures, caching, generators, local variable binding in hot loops, vectorization (NumPy), moving hot code to C/Rust/Cython, concurrency suited to the workload.

**Q51. [Medium] `str.join` vs. `+=` in loops?**
`"".join(parts)` is linear; repeated concatenation can be quadratic.

---

## 10. Architecture and Design (Senior Level)

**Q52. [Critical] How would you structure a large Python service?**
Layered or hexagonal architecture: domain logic isolated from frameworks, dependency injection at the edges, clear package boundaries, typed interfaces, configuration via environment.

**Q53. [High] FastAPI vs. Django vs. Flask?**
- Django: batteries included, ORM, admin; great for CRUD-heavy products
- FastAPI: async, type-driven validation, OpenAPI out of the box; great for APIs/microservices
- Flask: minimal and flexible; assemble your own stack

**Q54. [Critical] How do you design for scalability in Python services?**
Stateless workers behind a load balancer, horizontal scaling, task queues (Celery, RQ, Dramatiq), caching (Redis), async I/O where appropriate, DB connection pooling, observability.

**Q55. [High] How do you manage dependencies and releases across many services?**
Pinned lock files, internal package index, semantic versioning, automated dependency updates, CI with linting, type checks, tests, and security scanning.

**Q56. [Critical] How would you secure a Python application?**
Input validation, parameterized queries, secrets management, dependency auditing (`pip-audit`), least-privilege access, safe deserialization (avoid `pickle` on untrusted data), HTTPS everywhere.

**Q57. [Medium] How do you integrate Python with other stacks (e.g. .NET, Java)?**
REST/gRPC contracts with OpenAPI/Protobuf schemas, message brokers (Kafka, RabbitMQ, Azure Service Bus), shared event schemas, and containerized deployments.

**Q58. [Critical] How do you approach observability?**
Structured logging, metrics (Prometheus), distributed tracing (OpenTelemetry), health endpoints, and correlation IDs across services.

---

## 11. Production Scenarios and Current Topics

**Q59. [Medium] What is structural pattern matching, and when would you use it?**
`match`/`case` (Python 3.10+) matches the shape and values of data, including mappings, sequences and classes. It is useful for parsing commands or event payloads, but ordinary `if`/`elif` is often clearer for simple conditions.

**Q60. [Medium] What are `ExceptionGroup` and `except*`?**
They allow multiple independent exceptions to be raised and handled together, especially by concurrent tasks. `except* SomeError` handles matching members of the group; it is different from regular `except`, which handles one exception flow.

**Q61. [High] What are `TypedDict`, `Literal`, `TypeGuard` and `TypeVar` used for?**
They express more precise contracts for dictionaries, allowed values, user-defined type narrowing and reusable generic code. Static checkers use these annotations; Python generally does not enforce them at runtime.

**Q62. [Critical] How do you make an async service reliable?**
Use timeouts, bounded concurrency, cancellation-safe cleanup, retries only for transient failures, exponential backoff with jitter, and idempotency keys. Do not call blocking libraries directly on the event-loop thread; use an async client or offload the work.

**Q63. [High] What is the difference between `asyncio.gather`, `TaskGroup` and `create_task`?**
`create_task` schedules one task. `gather` waits for multiple awaitables and has configurable exception behavior. `TaskGroup` (3.11+) provides structured concurrency: child tasks share a lifetime and failures cancel siblings and are reported after cleanup.

**Q64. [High] How do you prevent race conditions in threaded Python code?**
Protect shared mutable state with locks or queues, keep critical sections small, prefer immutable data or message passing, and test under realistic concurrency. The GIL does not make multi-step operations or application invariants automatically safe.

**Q65. [Critical] How would you investigate a slow production endpoint?**
Confirm the symptom with latency percentiles, traces and logs; separate application, database and downstream time; inspect query plans and connection pools; reproduce with representative load; then make one measured change and compare p95/p99, error rate and resource usage.

**Q66. [Critical] How do you design a safe retry policy?**
Retry only transient, idempotent operations; set a maximum attempt count and total deadline; use exponential backoff with jitter; honor `Retry-After`; and use a circuit breaker or bulkhead to avoid amplifying an outage.

**Q67. [Critical] How do you handle database transactions in a Python service?**
Keep transaction scope short, commit explicitly at the service boundary, roll back on failure, use parameterized queries, and choose isolation levels deliberately. For an external side effect, discuss an outbox, idempotency key or compensating action rather than assuming a distributed transaction.

**Q68. [High] How would you evolve a public API without breaking clients?**
Prefer additive, backward-compatible changes; validate and version schemas; support a deprecation window; publish examples and contract tests; and use feature flags or content negotiation for migrations.

**Q69. [High] What belongs in a production-ready Python package?**
A `pyproject.toml`, declared runtime and optional dependencies, a reproducible lock/build process, tests, documentation, typed public APIs where useful, lint/format/type-check configuration, and CI that builds and installs the artifact in a clean environment.

**Q70. [Critical] What security issues are common in Python applications?**
Unsafe deserialization (`pickle`), SQL injection, command injection, SSRF, path traversal, leaked secrets, vulnerable dependencies and weak authorization. Validate at trust boundaries, use allowlists and parameterized APIs, apply least privilege, and audit dependencies.

**Q71. [High] How do you test code that calls an external service?**
Keep unit tests deterministic by injecting the client and mocking at the lookup site. Add contract tests against the agreed schema, integration tests against a disposable or sandbox service, and a small number of end-to-end tests. Test timeouts, retries, malformed responses and rate limits.

**Q72. [High] What is the difference between liveness, readiness and health checks?**
Liveness answers whether the process should be restarted. Readiness answers whether it should receive traffic. A health or diagnostic endpoint can report dependencies, but it should not make a transient downstream failure cause unnecessary process restarts.

**Q73. [High] How do you handle graceful shutdown?**
Stop accepting new work, signal workers, allow in-flight requests a deadline to finish, cancel remaining tasks safely, close pools and flush telemetry, then exit. The design must tolerate repeated signals and partial shutdown.

**Q74. [High] Describe a production incident you handled.**
Use a concise situation-action-result structure: impact and detection, hypothesis and mitigation, communication, root cause, and the follow-up that reduced recurrence. Strong answers include metrics and what you learned, without blaming individuals.

**Q75. [High] How would you review a Python pull request?**
Check correctness and edge cases first, then API clarity, failure handling, security, performance, observability, tests and maintainability. Separate blocking defects from style preferences and explain the risk behind each requested change.

## 12. Coding Exercises

1. **[Critical]** Reverse a string without slicing.
2. **[Critical]** Check whether two strings are anagrams.
3. **[High]** Find the first non-repeating character.
4. **[High]** Implement an LRU cache using `OrderedDict`, then without it.
5. **[High]** Flatten an arbitrarily nested list using a generator.
6. **[High]** Merge overlapping intervals.
7. **[Medium]** Implement a thread-safe singleton.
8. **[Critical]** Write a retry decorator with exponential backoff.
9. **[High]** Implement a rate limiter (token bucket).
10. **[Critical]** Read a multi-GB file and compute word frequencies with constant memory.
11. **[High]** Build an async scraper that limits concurrency with `asyncio.Semaphore`.
12. **[Medium]** Design a simple plugin system using entry points or a registry decorator.
13. **[High]** Implement a bounded worker pool with backpressure and graceful shutdown.
14. **[Critical]** Build an async API client with timeout, cancellation, retry and rate-limit handling.
15. **[High]** Implement an idempotent request handler using an idempotency key and a storage interface.
16. **[High]** Write a log parser that streams a large file and produces a time-windowed error summary.
17. **[High]** Implement an in-memory TTL cache with thread-safe reads, writes and eviction.
18. **[Medium]** Given service latency samples, compute p50, p95 and p99 without sorting the full input.
19. **[High]** Design a paginated API endpoint and write tests for validation, authorization and cursor expiry.

### Example: Flatten nested lists
```python
def flatten(items):
    for item in items:
        if isinstance(item, (list, tuple)):
            yield from flatten(item)
        else:
            yield item

print(list(flatten([1, [2, [3, 4]], 5])))  # [1, 2, 3, 4, 5]
```

### Example: Retry with backoff
```python
import functools, random, time

def retry(times=3, base=0.5, exceptions=(Exception,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw):
            for attempt in range(times):
                try:
                    return fn(*a, **kw)
                except exceptions:
                    if attempt == times - 1:
                        raise
                    time.sleep(base * 2 ** attempt + random.random() * 0.1)
        return wrapper
    return deco
```

---

## Tips for the Interview

- Clarify requirements and constraints before coding.
- State time and space complexity.
- Talk through trade-offs, not just the final answer.
- Write tests or at least walk through edge cases (empty input, `None`, large input).
- For senior roles, tie answers back to maintainability, scalability and team impact.
