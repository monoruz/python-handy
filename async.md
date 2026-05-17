# Python Async
## basic coroutine
```python
import asyncio

async def say_hello():
    print("hello")
    await asyncio.sleep(1)
    print("world")

asyncio.run(say_hello())
```
## gather (run concurrently)
```python
import asyncio

async def fetch(url, delay):
    await asyncio.sleep(delay)
    return f"result from {url}"

async def main():
    results = await asyncio.gather(
        fetch("/api/a", 2),
        fetch("/api/b", 1),
        fetch("/api/c", 3),
    )
    print(results)  # all three finish in ~3s, not 6s

asyncio.run(main())
```
## create_task (fire and forget scheduling)
```python
import asyncio

async def background_job(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} done")

async def main():
    task1 = asyncio.create_task(background_job("A", 2))
    task2 = asyncio.create_task(background_job("B", 1))
    # tasks are already running, await collects their results
    await task1
    await task2

asyncio.run(main())
```
## TaskGroup (Python 3.11+)
```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)
    return f"{name}: ok"

async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch("A", 1))
        t2 = tg.create_task(fetch("B", 2))
    # if any task raises, all others are cancelled and an ExceptionGroup is raised
    print(t1.result(), t2.result())

asyncio.run(main())
```
## timeout
```python
import asyncio

async def slow():
    await asyncio.sleep(10)

async def main():
    try:
        async with asyncio.timeout(2):
            await slow()
    except TimeoutError:
        print("timed out!")

asyncio.run(main())
```
## async generator
```python
import asyncio

async def ticker(to, delay):
    for i in range(to):
        yield i
        await asyncio.sleep(delay)

async def main():
    async for val in ticker(5, 0.5):
        print(val)

asyncio.run(main())
```
## async context manager
```python
import asyncio

class AsyncConnection:
    async def __aenter__(self):
        print("connecting...")
        await asyncio.sleep(0.5)
        return self

    async def __aexit__(self, exc_type, exc, tb):
        print("disconnecting...")
        await asyncio.sleep(0.5)

    async def query(self, q):
        await asyncio.sleep(0.1)
        return f"result for {q}"

async def main():
    async with AsyncConnection() as conn:
        result = await conn.query("SELECT 1")
        print(result)

asyncio.run(main())
```
## semaphore (limit concurrency)
```python
import asyncio

async def fetch(sem, url):
    async with sem:
        print(f"fetching {url}")
        await asyncio.sleep(1)
        return url

async def main():
    sem = asyncio.Semaphore(3)  # max 3 concurrent
    urls = [f"https://example.com/{i}" for i in range(10)]
    results = await asyncio.gather(*(fetch(sem, u) for u in urls))
    print(len(results))

asyncio.run(main())
```
## async queue (producer/consumer)
```python
import asyncio

async def producer(queue):
    for i in range(5):
        await asyncio.sleep(0.5)
        await queue.put(i)
        print(f"produced {i}")
    await queue.put(None)  # sentinel

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"consumed {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=2)
    await asyncio.gather(producer(queue), consumer(queue))

asyncio.run(main())
```
## Event (signal between coroutines)
```python
import asyncio

async def waiter(event):
    print("waiting for event...")
    await event.wait()
    print("event fired!")

async def setter(event):
    await asyncio.sleep(2)
    event.set()

async def main():
    event = asyncio.Event()
    await asyncio.gather(waiter(event), setter(event))

asyncio.run(main())
```
## Lock
```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:
        temp = counter
        await asyncio.sleep(0.01)
        counter = temp + 1

async def main():
    await asyncio.gather(*(increment() for _ in range(100)))
    print(counter)  # 100 (without lock this would be wrong)

asyncio.run(main())
```
## run blocking code in executor
```python
import asyncio
import time

def blocking_io():
    time.sleep(2)
    return "done"

async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_io)
    print(result)

asyncio.run(main())
```
## async comprehension
```python
import asyncio

async def get_value(x):
    await asyncio.sleep(0.1)
    return x * 2

async def main():
    results = [await get_value(i) for i in range(5)]
    print(results)  # [0, 2, 4, 6, 8]
    # NOTE: this runs sequentially! for concurrent use gather instead

asyncio.run(main())
```
## as_completed (process fastest first)
```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)
    return name

async def main():
    coros = [fetch("slow", 3), fetch("fast", 1), fetch("mid", 2)]
    for coro in asyncio.as_completed(coros):
        result = await coro
        print(result)  # prints: fast, mid, slow

asyncio.run(main())
```
## wait (fine-grained control)
```python
import asyncio

async def flaky(n):
    await asyncio.sleep(n)
    if n == 2:
        raise ValueError("boom")
    return n

async def main():
    tasks = [asyncio.create_task(flaky(i)) for i in [1, 2, 3]]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_EXCEPTION)
    for t in done:
        if t.exception():
            print(f"failed: {t.exception()}")
        else:
            print(f"ok: {t.result()}")
    for t in pending:
        t.cancel()

asyncio.run(main())
```
## shield (protect from cancellation)
```python
import asyncio

async def critical_operation():
    await asyncio.sleep(2)
    print("critical work done")
    return 42

async def main():
    try:
        result = await asyncio.wait_for(
            asyncio.shield(critical_operation()), timeout=1
        )
    except TimeoutError:
        print("outer timed out, but shielded task keeps running")
        await asyncio.sleep(2)  # give it time to finish

asyncio.run(main())
```
---
# Common Async Mistakes
## mistake 1: calling coroutine without await
```python
# WRONG - returns a coroutine object, never executes
async def fetch_data():
    await asyncio.sleep(1)
    return "data"

async def main():
    result = fetch_data()  # forgot await!
    print(result)  # <coroutine object fetch_data at 0x...>
    print(type(result))  # <class 'coroutine'>
```
```python
# CORRECT
async def main():
    result = await fetch_data()
    print(result)  # "data"
```
## mistake 2: using time.sleep instead of asyncio.sleep
```python
# WRONG - blocks the entire event loop
import time

async def handler(name):
    print(f"{name} start")
    time.sleep(2)  # blocks everything!
    print(f"{name} end")

async def main():
    await asyncio.gather(handler("A"), handler("B"))
    # takes 4 seconds instead of 2!
```
```python
# CORRECT
async def handler(name):
    print(f"{name} start")
    await asyncio.sleep(2)
    print(f"{name} end")

async def main():
    await asyncio.gather(handler("A"), handler("B"))
    # takes 2 seconds total
```
## mistake 3: creating tasks but not awaiting them
```python
# WRONG - tasks may be garbage collected before completing
async def main():
    asyncio.create_task(background_work())  # not saved or awaited!
    # function returns, task may never finish
```
```python
# CORRECT
async def main():
    task = asyncio.create_task(background_work())
    # ... do other stuff ...
    await task  # make sure it completes
```
## mistake 4: sequential awaits when you want concurrency
```python
# WRONG - runs one after another (6 seconds total)
async def main():
    a = await fetch("/api/a", 2)
    b = await fetch("/api/b", 2)
    c = await fetch("/api/c", 2)
```
```python
# CORRECT - runs all at once (2 seconds total)
async def main():
    a, b, c = await asyncio.gather(
        fetch("/api/a", 2),
        fetch("/api/b", 2),
        fetch("/api/c", 2),
    )
```
## mistake 5: using asyncio.run() inside an already running loop
```python
# WRONG - raises RuntimeError
async def inner():
    return 42

async def outer():
    result = asyncio.run(inner())  # RuntimeError: cannot be called from a running event loop
```
```python
# CORRECT
async def outer():
    result = await inner()  # just await it directly
```
## mistake 6: not handling exceptions in gathered tasks
```python
# WRONG - one failure cancels everything, others are lost
async def flaky(n):
    if n == 2:
        raise ValueError("boom")
    return n

async def main():
    results = await asyncio.gather(flaky(1), flaky(2), flaky(3))
    # ValueError propagates, results from 1 and 3 are lost
```
```python
# CORRECT - return_exceptions=True captures errors as values
async def main():
    results = await asyncio.gather(
        flaky(1), flaky(2), flaky(3),
        return_exceptions=True
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"error: {r}")
        else:
            print(f"ok: {r}")
    # ok: 1
    # error: boom
    # ok: 3
```
## mistake 7: sharing mutable state without a lock
```python
# WRONG - race condition
counter = 0

async def increment():
    global counter
    temp = counter
    await asyncio.sleep(0.01)  # context switch happens here
    counter = temp + 1

async def main():
    await asyncio.gather(*(increment() for _ in range(100)))
    print(counter)  # much less than 100!
```
```python
# CORRECT - use asyncio.Lock
lock = asyncio.Lock()
counter = 0

async def increment():
    global counter
    async with lock:
        temp = counter
        await asyncio.sleep(0.01)
        counter = temp + 1

async def main():
    await asyncio.gather(*(increment() for _ in range(100)))
    print(counter)  # 100
```
## mistake 8: blocking the event loop with CPU-bound work
```python
# WRONG - starves other coroutines
import hashlib

async def hash_data(data):
    for _ in range(1_000_000):
        data = hashlib.sha256(data).digest()  # CPU-bound, never yields
    return data
```
```python
# CORRECT - offload to a thread or process pool
import hashlib
from concurrent.futures import ProcessPoolExecutor

def _hash_sync(data):
    for _ in range(1_000_000):
        data = hashlib.sha256(data).digest()
    return data

async def hash_data(data):
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, _hash_sync, data)
    return result
```
## mistake 9: forgetting to close async resources
```python
# WRONG - connection/session leak
import aiohttp

async def fetch(url):
    session = aiohttp.ClientSession()
    resp = await session.get(url)
    return await resp.text()
    # session never closed!
```
```python
# CORRECT - use async with
import aiohttp

async def fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()
```
## mistake 10: assuming async is always faster
```python
# async does NOT help with CPU-bound work
# these both take the same time:

# synchronous
def compute():
    return sum(i * i for i in range(10_000_000))

# async (no benefit - nothing to await)
async def compute_async():
    return sum(i * i for i in range(10_000_000))

# async shines when waiting on I/O (network, disk, database)
# not when crunching numbers
```
