# 20 — Async & asyncio

> Week 2 · Python Advanced

This is the lesson that finally makes "why is FastAPI so fast?" click. You already write web code; here you learn how Python does **concurrency** without threads-everywhere, why the famous **GIL** exists, and exactly when `async`/`await` helps (spoiler: I/O, not number-crunching).

## What you will learn

- The difference between **concurrency** and **parallelism** (they are not the same thing).
- **CPU-bound** vs **I/O-bound** work, and which kind async actually speeds up.
- What the **GIL** (Global Interpreter Lock) is and why threads do *not* give you CPU parallelism in Python.
- **Coroutines**: writing `async def` functions and `await`ing them.
- The **event loop** and `asyncio.run()` as the single entry point.
- Running things **concurrently** with `asyncio.gather` and `asyncio.create_task`.
- **`async with`** and **`async for`** for async resources and streams.
- Using `asyncio.sleep` as a stand-in for real I/O.
- The classic beginner traps: calling a coroutine without `await`, and blocking the loop with synchronous calls.
- Why this matters for **FastAPI**, **ASGI**, and async endpoints.
- A decision table: **threading vs asyncio vs multiprocessing**.

## PHP/Laravel bridge

In standard PHP-FPM, each HTTP request runs in its own process and code is **synchronous and blocking**: when you call `Http::get()` or run a DB query, that worker just sits and waits, and concurrency comes entirely from running many FPM workers. Python's `asyncio` brings a different model — a *single* thread can juggle thousands of in-flight I/O operations by suspending a coroutine whenever it would block and resuming it when the data is ready. The closest things you've touched in the PHP world are **Laravel Octane** (Swoole/RoadRunner), **ReactPHP**, or **Amp** — those bolt an event loop onto PHP much like `asyncio` is built into Python. The mental shift: instead of "spin up more workers to wait in parallel," it's "one worker that never wastes time waiting."

---

## Core concept: concurrency vs parallelism

These two words get used interchangeably in casual conversation, but in an interview you must keep them distinct.

| Term | Meaning | Kitchen analogy |
|------|---------|-----------------|
| **Concurrency** | *Dealing with* many things at once by interleaving them. Progress on multiple tasks overlaps in time, but not necessarily at the same instant. | One cook starting the rice, then chopping veg *while the rice boils*, then stirring the sauce. One person, lots of overlap. |
| **Parallelism** | *Doing* many things at literally the same instant, on multiple CPU cores. | Three cooks, each on their own dish, simultaneously. |

`asyncio` gives you **concurrency on a single thread**. It does not, by itself, give you parallelism. One coroutine runs at a time; the trick is that when a coroutine is *waiting* (for a network response, a disk read, a DB row), it yields control so another coroutine can make progress. Nothing actually runs on two cores at once.

> Rob Pike's line is worth memorising: **"Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once."**

## Core concept: CPU-bound vs I/O-bound

The single most important question before reaching for async:

- **I/O-bound** work spends most of its time *waiting* — for the network, the database, the disk, another service. The CPU is idle during the wait. Examples: calling 50 HTTP APIs, querying Postgres, reading files.
- **CPU-bound** work spends most of its time *computing* — hashing, image resizing, parsing huge JSON, number crunching, machine learning math. The CPU is pinned at 100%.

**Async helps I/O-bound work and does nothing for CPU-bound work.** When you `await` a network call, the event loop reclaims the idle waiting time and spends it on other coroutines. But if a coroutine is doing pure computation, there is no idle time to reclaim — it just hogs the single thread and *blocks every other coroutine*. For CPU-bound work you need real parallelism: **multiprocessing**.

```python
# I/O-bound: 100 requests that each wait ~200ms.
# Synchronous: ~20 seconds (100 x 0.2s, one after another).
# Async:       ~0.2 seconds (all waiting happens concurrently).

# CPU-bound: summing a billion numbers.
# Async gives you ZERO speedup — there's no waiting to overlap.
```

## Core concept: the GIL (and why threads don't parallelise CPU work)

The **GIL — Global Interpreter Lock** — is a mutex inside CPython (the standard Python interpreter) that allows **only one thread to execute Python bytecode at a time**. Even on a 16-core machine, two Python threads cannot run Python code simultaneously.

Why does it exist? It makes CPython's memory management (reference counting) simple and fast for the common single-threaded case, and it makes a lot of C extension code safe by default. The tradeoff: no free CPU parallelism from threads.

Consequences you must be able to recite:

- **CPU-bound + threads = no speedup** (often *slower*, due to lock contention). Use **multiprocessing** — separate processes, each with its own interpreter and its own GIL, running on different cores.
- **I/O-bound + threads = real benefit**, because a thread *releases the GIL while it waits* on blocking I/O (and many C calls). So another thread runs during the wait.
- **`asyncio`** sidesteps the whole debate for I/O: it gets the same "use the waiting time" benefit as threads, but with one thread, no OS thread overhead, and cooperative (predictable) switching instead of preemptive switching.

```python
# Demonstration: two CPU-bound threads do NOT run faster than one,
# because the GIL serialises bytecode execution.
import time
from threading import Thread

def burn(n: int) -> None:
    while n > 0:
        n -= 1

COUNT = 50_000_000

start = time.perf_counter()
burn(COUNT)
burn(COUNT)
print(f"sequential: {time.perf_counter() - start:.2f}s")
# => sequential: ~2.0s

start = time.perf_counter()
t1 = Thread(target=burn, args=(COUNT,))
t2 = Thread(target=burn, args=(COUNT,))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"two threads: {time.perf_counter() - start:.2f}s")
# => two threads: ~2.0s (or slightly slower) — NOT 1.0s. The GIL is why.
```

> Note for currency: recent CPython (3.13+) ships an experimental **free-threaded / "no-GIL" build** behind a build flag. It is opt-in and not the default in 3.12, so for interviews treat "CPython has a GIL" as the correct baseline answer, and mention the free-threaded build as the forward-looking caveat.

## Core concept: coroutines — `async def` and `await`

A function defined with **`async def`** is a **coroutine function**. Calling it does **not run the body** — it returns a **coroutine object** that is "paused at the start." You must `await` it (or hand it to the event loop) to actually run it.

```python
import asyncio

async def greet(name: str) -> str:
    print(f"start {name}")
    await asyncio.sleep(1)        # pretend this is a network/DB call
    print(f"done  {name}")
    return f"Hello, {name}"

# Calling it does NOT execute the body:
coro = greet("Ada")
print(type(coro))                # => <class 'coroutine'>
# RuntimeWarning: coroutine 'greet' was never awaited  <-- the #1 beginner bug

# To actually run it, you need an event loop:
result = asyncio.run(greet("Ada"))
# => start Ada
# (1 second pause)
# => done  Ada
print(result)                    # => Hello, Ada
```

**`await`** does two things: it runs the awaited coroutine, and — crucially — it marks a point where *this* coroutine is allowed to suspend so the event loop can do other work. You can only use `await` inside an `async def`.

What can you `await`? Any **awaitable**: another coroutine, a `Task`, or a `Future`. In practice that means async library functions like `await session.get(url)` (httpx), `await conn.fetch(...)` (asyncpg), `await asyncio.sleep(...)`.

## Core concept: the event loop and `asyncio.run()`

The **event loop** is the engine. It keeps a queue of ready-to-run coroutines, runs one until it hits an `await` that suspends, then picks the next ready one, and so on. When an awaited I/O operation completes, the loop wakes the coroutine that was waiting on it.

You almost never manage the loop by hand. The modern, correct entry point is **`asyncio.run(coro)`**:

```python
import asyncio

async def main() -> None:
    print("in the event loop")
    await asyncio.sleep(0.1)
    print("loop still running")

asyncio.run(main())   # creates a loop, runs main() to completion, closes the loop
```

Rules of thumb:

- Call `asyncio.run()` **once**, at the top level of your program (the `main()` pattern). Do not call it inside an already-running loop.
- In a web framework like FastAPI, **you do not call `asyncio.run()` yourself** — the ASGI server (uvicorn) owns the loop and simply `await`s your handlers. You just write `async def` endpoints.

## Core concept: doing things concurrently

Here's the part people get wrong. Putting `await` in front of two calls *in sequence* does **not** make them concurrent — it just waits for the first to finish before starting the second.

```python
import asyncio, time

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)      # stand-in for an I/O call
    return f"{name} ({delay}s)"

async def sequential() -> None:
    start = time.perf_counter()
    a = await fetch("A", 1)         # wait 1s ...
    b = await fetch("B", 1)         # ... THEN wait another 1s
    print(a, b, f"{time.perf_counter()-start:.2f}s")
    # => A (1s) B (1s) 2.00s   <-- total ~2s, NOT concurrent

asyncio.run(sequential())
```

### `asyncio.gather` — run a batch and collect results

`gather` schedules every awaitable to run concurrently and returns their results **in the order you passed them** (not the order they finished).

```python
import asyncio, time

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"{name} ({delay}s)"

async def concurrent() -> None:
    start = time.perf_counter()
    results = await asyncio.gather(
        fetch("A", 1),
        fetch("B", 1),
        fetch("C", 2),
    )
    print(results, f"{time.perf_counter()-start:.2f}s")
    # => ['A (1s)', 'B (1s)', 'C (2s)'] 2.00s   <-- ~2s, not 4s!

asyncio.run(concurrent())
```

By default, if one awaitable raises, `gather` propagates that exception and the others keep running in the background. Pass `return_exceptions=True` to get the exceptions back as values instead of raising:

```python
import asyncio

async def ok() -> str:
    return "fine"

async def boom() -> str:
    raise ValueError("kaboom")

async def main() -> None:
    results = await asyncio.gather(ok(), boom(), return_exceptions=True)
    print(results)
    # => ['fine', ValueError('kaboom')]

asyncio.run(main())
```

### `asyncio.create_task` — start now, await later

`create_task` schedules a coroutine to start running **immediately in the background** and hands you a `Task` handle. This is the tool when you want a coroutine to make progress *while you do other work*, then collect it.

```python
import asyncio, time

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return name

async def main() -> None:
    start = time.perf_counter()
    # Both tasks start running here, immediately and concurrently:
    t1 = asyncio.create_task(fetch("A", 2))
    t2 = asyncio.create_task(fetch("B", 1))

    # ... you could do other awaitable work here ...

    print(await t1, await t2, f"{time.perf_counter()-start:.2f}s")
    # => A B 2.00s   <-- overlapped, total = the longest one

asyncio.run(main())
```

> Gotcha: a bare `create_task` whose handle you discard can be **garbage-collected mid-flight**. Keep a reference (e.g. add it to a set) or use a structured group.

### `asyncio.TaskGroup` — the modern, safe way (3.11+)

`TaskGroup` (an `async with` context manager) is the recommended pattern in modern Python. It waits for every task on exit, and if any task fails it cancels the siblings and raises — no silently-lost tasks.

```python
import asyncio

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return name

async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch("A", 1))
        t2 = tg.create_task(fetch("B", 2))
    # On exit, both tasks are guaranteed done.
    print(t1.result(), t2.result())   # => A B

asyncio.run(main())
```

### Timeouts

Don't let one slow dependency hang forever. `asyncio.timeout` (3.11+) is the clean way:

```python
import asyncio

async def slow() -> str:
    await asyncio.sleep(10)
    return "eventually"

async def main() -> None:
    try:
        async with asyncio.timeout(1.0):
            await slow()
    except TimeoutError:
        print("gave up after 1s")   # => gave up after 1s

asyncio.run(main())
```

## Core concept: `async with` and `async for`

Async resources need async setup/teardown and async iteration. Two new keywords cover this.

**`async with`** drives an *asynchronous context manager* — its enter/exit can `await` (e.g. opening a pooled DB connection, an HTTP client session). This is exactly how you use `httpx.AsyncClient`:

```python
import asyncio
import httpx   # pip install httpx

async def get_status(url: str) -> int:
    async with httpx.AsyncClient() as client:   # async setup/teardown
        resp = await client.get(url)            # the awaited I/O
        return resp.status_code

async def main() -> None:
    code = await get_status("https://example.com")
    print(code)   # => 200

asyncio.run(main())
```

**`async for`** iterates an *asynchronous iterator*, awaiting between items — perfect for streaming responses, paginated APIs, or Kafka-style consumers where each "next item" requires I/O.

```python
import asyncio

async def stream_rows(n: int):
    """An async generator: yields values, awaiting (I/O) between each."""
    for i in range(n):
        await asyncio.sleep(0.1)   # pretend each row comes from the network
        yield {"id": i}

async def main() -> None:
    async for row in stream_rows(3):
        print(row)
    # => {'id': 0}
    # => {'id': 1}
    # => {'id': 2}

asyncio.run(main())
```

## Core concept: `asyncio.sleep` as a stand-in for I/O

Throughout this lesson `asyncio.sleep(seconds)` plays the role of "a real I/O call." That's deliberate: `await asyncio.sleep(1)` suspends the coroutine and **yields control to the loop** for one second, exactly the way `await client.get(...)` yields while the network round-trip happens. It is the *async-friendly* sleep.

Contrast it with the blocking `time.sleep(1)`, which freezes the **entire thread and event loop** — every other coroutine is stuck. This is the heart of the "don't block the loop" rule below.

```python
import asyncio, time

async def good() -> None:
    await asyncio.sleep(1)   # loop free to run other coroutines

async def bad() -> None:
    time.sleep(1)            # BLOCKS the whole event loop for 1 second
```

## Core concept: the cardinal sin — blocking the event loop

Because everything shares **one thread**, a single synchronous/blocking call stalls *all* coroutines. The loop can't switch away from a coroutine that never `await`s. Common offenders:

- `time.sleep()` — use `await asyncio.sleep()`.
- `requests.get()` (the sync library) — use `httpx.AsyncClient` or `aiohttp`.
- A synchronous DB driver (`psycopg2` in blocking mode) — use `asyncpg` / async SQLAlchemy.
- Heavy CPU work (parsing, hashing, image processing) — it has no `await`, so it monopolises the loop.

When you *must* call blocking code from async, offload it to a thread or process pool so the loop stays responsive:

```python
import asyncio
import time

def blocking_io() -> str:
    time.sleep(2)            # legacy blocking call we can't change
    return "done"

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))   # pure CPU

async def main() -> None:
    # Offload blocking I/O to a thread (releases the loop):
    result = await asyncio.to_thread(blocking_io)
    print(result)            # => done

    # Offload CPU work to a PROCESS pool (true parallelism, dodges the GIL):
    loop = asyncio.get_running_loop()
    import concurrent.futures
    with concurrent.futures.ProcessPoolExecutor() as pool:
        total = await loop.run_in_executor(pool, cpu_heavy, 10_000_000)
    print(total)

asyncio.run(main())
```

## Core concept: why async matters for web frameworks (FastAPI & ASGI)

Traditional Python web apps run on **WSGI** (the interface behind Gunicorn + Flask/Django-sync). WSGI is synchronous: one worker handles one request at a time, blocking while it waits on the DB or an upstream API. **ASGI** (Asynchronous Server Gateway Interface) is the async successor — the spec behind **uvicorn**, **FastAPI**, **Starlette**, and async Django. ASGI lets a single worker hold thousands of concurrent connections, because while one request is awaiting its DB row, the worker serves others.

In **FastAPI** you opt in simply by declaring the endpoint `async def`:

```python
# app.py  —  run with:  uvicorn app:app --reload
import asyncio
from fastapi import FastAPI
import httpx

app = FastAPI()

@app.get("/slow-io")
async def slow_io():
    # While this awaits, the worker serves OTHER requests concurrently.
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://httpbin.org/delay/1")
    return {"upstream_status": resp.status_code}

@app.get("/fanout")
async def fanout():
    # Call three upstreams concurrently instead of one-by-one:
    async with httpx.AsyncClient() as client:
        urls = ["https://httpbin.org/delay/1"] * 3
        responses = await asyncio.gather(*(client.get(u) for u in urls))
    return {"count": len(responses)}   # ~1s total, not ~3s
```

Two FastAPI specifics worth knowing for interviews:

1. **You can mix sync and async endpoints.** A plain `def` endpoint is run by FastAPI in an external **thread pool**, so a blocking call there won't freeze the loop. But an `async def` endpoint that secretly does blocking work (e.g. `time.sleep`, `requests.get`) **will** freeze the loop and tank throughput. Rule: if your handler is `async`, every I/O call inside it must be `await`ed/async.
2. **Async only helps if the whole chain is async.** An async endpoint calling a synchronous ORM still blocks. Use async DB drivers (asyncpg, async SQLAlchemy 2.x) to get the benefit end-to-end.

## Common mistakes & gotchas

- **Calling a coroutine without `await`.** `greet("Ada")` just builds a coroutine object and runs nothing; you get a `RuntimeWarning: coroutine '...' was never awaited`. Always `await` it (or wrap in a task).
- **Sequential `await`s when you wanted concurrency.** `a = await x(); b = await y()` runs them one after another. Use `gather`/`TaskGroup` to overlap.
- **Blocking the loop** with `time.sleep`, `requests`, `psycopg2`, or heavy CPU inside an `async def`. One offender stalls every coroutine. Offload with `asyncio.to_thread` (I/O) or a process pool (CPU).
- **Expecting CPU parallelism from async or threads.** Neither beats the GIL for compute. Use **multiprocessing**.
- **`asyncio.run()` inside a running loop** (e.g. in a Jupyter notebook or a FastAPI handler) → `RuntimeError: asyncio.run() cannot be called from a running event loop`. Just `await` instead.
- **Fire-and-forget `create_task` whose handle is discarded** — it can be garbage-collected before finishing. Keep a reference or use `TaskGroup`.
- **`gather` reorders? No — results keep input order**, even though tasks finish out of order. A common misconception.
- **Mixing the loop across threads.** Loops are per-thread; don't share one event loop object between OS threads.

## Best practices

- Have exactly **one** `asyncio.run(main())` entry point; let it own the loop.
- Reach for **`asyncio.TaskGroup`** over bare `create_task` for structured, leak-free concurrency (3.11+).
- Wrap risky awaits in **`asyncio.timeout`** so one slow dependency can't hang a request.
- Keep coroutines **async all the way down** — async client (httpx/aiohttp), async DB driver (asyncpg / SQLAlchemy async). A single sync call poisons the chain.
- Use **`asyncio.to_thread`** for unavoidable blocking I/O and a **`ProcessPoolExecutor`** for CPU-bound work.
- Decide with the rule: **I/O-bound → asyncio (or threads); CPU-bound → multiprocessing.**
- In FastAPI, only mark a handler `async def` if you can keep its I/O async; otherwise leave it plain `def` and let FastAPI thread-pool it.
- Add type hints: a coroutine function returning `str` is typed `async def f() -> str`.

## Interview Q&A

**Q: What's the difference between concurrency and parallelism?**
**A:** Concurrency is *structuring* a program so multiple tasks make progress by interleaving (overlapping in time) — it can happen on a single core. Parallelism is *literally* executing multiple tasks at the same instant on multiple cores. `asyncio` provides concurrency on one thread; multiprocessing provides parallelism.

**Q: When does async actually help, and when is it useless?**
**A:** It helps **I/O-bound** workloads — anything that spends time *waiting* on the network, disk, or DB — because the event loop reclaims that idle wait time for other coroutines. It does **nothing** for **CPU-bound** work: there's no waiting to overlap, so a compute-heavy coroutine just monopolises the single thread. CPU-bound work needs multiprocessing.

**Q: What is the GIL and what does it mean for threads?**
**A:** The Global Interpreter Lock is a CPython mutex that lets only one thread execute Python bytecode at a time. So threads give **no CPU parallelism** — two CPU-bound threads don't run faster than one. Threads *do* help I/O-bound work, because a thread releases the GIL while blocked on I/O. For CPU parallelism you use multiprocessing (each process has its own interpreter and GIL).

**Q: What does calling an `async def` function return?**
**A:** A **coroutine object** — the body does not run yet. You must `await` it, schedule it with `asyncio.create_task`, or pass it to `asyncio.run`/`gather`. Forgetting this is the classic "coroutine was never awaited" warning.

**Q: What exactly does `await` do?**
**A:** It runs the awaited awaitable and marks a **suspension point**: while that awaitable is pending (e.g. network I/O), the current coroutine yields control back to the event loop, which runs other ready coroutines. When the awaited operation completes, the loop resumes the coroutine.

**Q: Difference between `asyncio.gather` and `asyncio.create_task`?**
**A:** `create_task` schedules a single coroutine to start running in the background immediately and returns a `Task` handle you `await` later. `gather` takes several awaitables, runs them concurrently, and returns all results together (in input order). `gather` is convenient for "run this batch and collect"; `create_task`/`TaskGroup` is for finer control or fire-and-collect-later patterns.

**Q: Does `await x(); await y()` run x and y concurrently?**
**A:** No. That awaits `x` to completion, *then* starts `y` — fully sequential. To overlap them use `await asyncio.gather(x(), y())` or a `TaskGroup`.

**Q: What is "blocking the event loop" and how do you avoid it?**
**A:** Because all coroutines share one thread, any synchronous/blocking call (`time.sleep`, `requests.get`, a sync DB driver, heavy CPU) prevents the loop from switching, stalling every coroutine. Avoid it by using async libraries and `await asyncio.sleep`, and offload unavoidable blocking work with `asyncio.to_thread` (I/O) or a `ProcessPoolExecutor` (CPU).

**Q: WSGI vs ASGI — why does FastAPI need ASGI?**
**A:** WSGI is a synchronous interface: one request per worker, blocking on I/O. ASGI is the async-capable successor, allowing a single worker to handle many concurrent connections by awaiting I/O. FastAPI/Starlette are ASGI apps served by uvicorn, which is what enables `async def` endpoints and high concurrency.

**Q: In FastAPI, when should an endpoint be `async def` vs `def`?**
**A:** Use `async def` when the handler's I/O is async-aware (httpx, asyncpg, async SQLAlchemy) so it cooperates with the loop. Use plain `def` for blocking/legacy code — FastAPI runs sync handlers in a thread pool so they don't freeze the loop. The dangerous combo is `async def` containing blocking calls (`time.sleep`, `requests`), which stalls the entire loop.

**Q: How do you run blocking or CPU-bound code from inside async code?**
**A:** For blocking I/O, `await asyncio.to_thread(func, *args)` — it runs in a thread that releases the GIL during the wait. For CPU-bound work, offload to a process pool via `loop.run_in_executor(ProcessPoolExecutor(), func, *args)` to get true parallelism past the GIL.

**Q: What's the modern preferred way to run a group of tasks safely?**
**A:** `asyncio.TaskGroup` (Python 3.11+) used as `async with`. It awaits all child tasks on exit and, if any fails, cancels the siblings and raises an `ExceptionGroup` — preventing silently-lost tasks that bare `create_task` can cause.

## Practice exercises

Try these before peeking. Run each with `python file.py`.

1. **Spot the bug.** This program prints a warning and no greeting. Fix it.
   ```python
   import asyncio
   async def hello():
       await asyncio.sleep(0.5)
       return "hi"
   def main():
       print(hello())
   main()
   ```

2. **Make it concurrent.** Three `await asyncio.sleep(1)` calls run sequentially (3s total). Rewrite so the total time is ~1s, using `asyncio.gather`.
   ```python
   import asyncio, time
   async def work(n): 
       await asyncio.sleep(1)
       return n
   async def main():
       start = time.perf_counter()
       a = await work(1); b = await work(2); c = await work(3)
       print(a, b, c, f"{time.perf_counter()-start:.2f}s")
   asyncio.run(main())
   ```

3. **Don't block the loop.** The endpoint below uses `time.sleep`, freezing the event loop. Make it loop-friendly.
   ```python
   import time
   async def handler():
       time.sleep(2)
       return {"ok": True}
   ```

4. **Classify the workload.** For each, say *I/O-bound or CPU-bound* and *asyncio, threads, or multiprocessing*: (a) downloading 500 images from a CDN; (b) resizing those 500 images; (c) running 1000 SQL queries against Postgres; (d) computing SHA-256 of a 10 GB file.

5. **TaskGroup + timeout.** Launch two coroutines with a `TaskGroup`; one takes 0.5s, one takes 5s. Wrap the group in a 2-second `asyncio.timeout` so the slow one is cancelled. Print whichever results you got.

6. **Async generator.** Write an async generator `countdown(n)` that yields `n, n-1, ..., 1`, sleeping 0.2s between each, and consume it with `async for`.

<details><summary>Solutions</summary>

**1.** Coroutines must be awaited and the program needs an event loop:
```python
import asyncio

async def hello():
    await asyncio.sleep(0.5)
    return "hi"

async def main():
    print(await hello())   # => hi

asyncio.run(main())
```

**2.** Use `gather` so the three sleeps overlap:
```python
import asyncio, time

async def work(n):
    await asyncio.sleep(1)
    return n

async def main():
    start = time.perf_counter()
    a, b, c = await asyncio.gather(work(1), work(2), work(3))
    print(a, b, c, f"{time.perf_counter()-start:.2f}s")
    # => 1 2 3 1.00s

asyncio.run(main())
```

**3.** Either await the async sleep, or offload real blocking work to a thread:
```python
import asyncio

async def handler():
    await asyncio.sleep(2)          # loop-friendly
    return {"ok": True}

# If the 2s were unavoidable blocking code:
import time
async def handler_offload():
    await asyncio.to_thread(time.sleep, 2)
    return {"ok": True}
```

**4.**
- (a) Downloading images → **I/O-bound** → **asyncio** (or threads). Lots of network waiting.
- (b) Resizing images → **CPU-bound** → **multiprocessing**. Pure computation, GIL-bound.
- (c) 1000 SQL queries → **I/O-bound** → **asyncio** with an async driver (asyncpg).
- (d) SHA-256 of 10 GB → mostly **CPU-bound** hashing → **multiprocessing** (or split the file across processes).

**5.**
```python
import asyncio

async def quick():
    await asyncio.sleep(0.5)
    return "quick done"

async def slow():
    await asyncio.sleep(5)
    return "slow done"

async def main():
    try:
        async with asyncio.timeout(2):
            async with asyncio.TaskGroup() as tg:
                t_quick = tg.create_task(quick())
                t_slow = tg.create_task(slow())
        print(t_quick.result(), t_slow.result())
    except TimeoutError:
        print("timed out; the slow task was cancelled")
        # => timed out; the slow task was cancelled

asyncio.run(main())
```

**6.**
```python
import asyncio

async def countdown(n):
    while n > 0:
        await asyncio.sleep(0.2)
        yield n
        n -= 1

async def main():
    async for value in countdown(3):
        print(value)
    # => 3
    # => 2
    # => 1

asyncio.run(main())
```

</details>

## Cheat sheet

| Concept | Syntax / call | Notes |
|---------|---------------|-------|
| Define coroutine | `async def f(): ...` | Calling it returns a coroutine; doesn't run yet |
| Run it (top level) | `asyncio.run(f())` | One per program; owns the loop |
| Await one thing | `x = await f()` | Only inside `async def` |
| Run many, collect | `await asyncio.gather(a(), b())` | Results in input order |
| Start in background | `t = asyncio.create_task(f())` then `await t` | Keep the reference |
| Structured group | `async with asyncio.TaskGroup() as tg: tg.create_task(...)` | 3.11+, leak-free, cancels on error |
| Timeout | `async with asyncio.timeout(2): ...` | Raises `TimeoutError` |
| Async resource | `async with client as c: ...` | Awaitable setup/teardown |
| Async iteration | `async for item in stream(): ...` | Awaits between items |
| Non-blocking sleep | `await asyncio.sleep(s)` | Stand-in for I/O; yields loop |
| Offload blocking I/O | `await asyncio.to_thread(fn, *args)` | Keeps loop responsive |
| Offload CPU work | `loop.run_in_executor(ProcessPoolExecutor(), fn, ...)` | True parallelism past GIL |

**Decision rule:** I/O-bound → `asyncio` (or threads). CPU-bound → `multiprocessing`. Never block the loop.

### threading vs asyncio vs multiprocessing

| | **threading** | **asyncio** | **multiprocessing** |
|---|---|---|---|
| Best for | I/O-bound | I/O-bound (high concurrency) | CPU-bound |
| Parallelism (CPU) | No (GIL) | No (single thread) | **Yes** (separate processes) |
| Switching | Preemptive (OS) | Cooperative (at `await`) | OS processes |
| Overhead per unit | OS thread (heavy-ish) | Coroutine (very light) | Process (heavy) |
| Shared memory | Yes (locks needed) | Yes (one thread, fewer races) | No (IPC / pickling) |
| Beats the GIL? | No | N/A | **Yes** |
| Typical use | Legacy blocking I/O, mixed code | Web servers, API fan-out, scraping | Image/ML/number crunching |

**One-line recall:** *Coroutine = pausable function; `await` = "I might wait here, run someone else"; event loop = the scheduler; gather/TaskGroup = overlap the waiting; multiprocessing = the only real CPU parallelism in CPython.*
