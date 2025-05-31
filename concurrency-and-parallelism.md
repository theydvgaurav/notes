# 🧠 Concurrency, Parallelism, Context Switching & Python Async — Deep Dive Notes

This document summarizes key insights and code explanations from an in-depth discussion about how concurrency and parallelism work in Python, including `asyncio`, threading, and system-level context switching.

---

## 📚 Concepts Covered

### ✅ Program vs Process
- **Program**: Passive code on disk.
- **Process**: Active instance of a program running in memory.
- A program may spawn **multiple processes** (e.g., Chrome tabs, Python scripts).

---

### 🧩 Process vs Thread
| Feature           | Process             | Thread               |
|-------------------|---------------------|----------------------|
| Memory Space      | Separate            | Shared               |
| Communication     | Slower (IPC)        | Faster (Shared memory)|
| Failure Isolation | Safer               | Riskier              |
| Switching Cost    | High                | Lower                |
| Python Example    | `multiprocessing`    | `threading`          |

---

### 🔄 Context Switching
- **Definition**: Switching the CPU from one task to another.
- **Cost**:
  - Process switch: 🔵 High
  - Thread switch: 🟨 Medium
  - Coroutine (`await`): 🟩 Very Low
- **Too many switches** can reduce performance. Efficient scheduling is crucial.
- **Context switching happens for both processes and threads**:
  - **Process** context switch involves saving/restoring the entire process state (memory, registers, etc).
  - **Thread** context switch is lighter but still involves register and stack management.

---

### ⬆️ Concurrency vs Parallelism
| Term          | Meaning                                   | Example               |
|---------------|-------------------------------------------|-----------------------|
| Concurrency   | Tasks appear to run at the same time (shared CPU) | `asyncio`, threads     |
| Parallelism   | Tasks run *at the same* time on multiple cores     | multiprocessing       |

- **Single-core** CPUs only support **concurrency**, via context switching.
- **Multi-core** CPUs support **true parallelism** (plus concurrency within each core).

---

## ⚙️ `asyncio`: Cooperative Concurrency in Python

### 🔹 How It Works:
- `async def` functions return coroutines.
- `await` pauses the coroutine and **yields control** to the event loop.
- The event loop resumes other ready tasks while waiting.

### 🔹 Key Point:
> **Coroutines don’t run in parallel. They interleave using `await`.**

---

### 🍪 Example Breakdown

```python
async def async_fun1():
    time.sleep(3)                # ❌ Blocks event loop
    await asyncio.sleep(1)       # ✅ Yields

async def async_fun2():
    await asyncio.sleep(2)       # ✅ Yields

async def async_fun3():
    await asyncio.sleep(1)       # ✅ Yields
```

### 🔍 Timeline (Executed via `asyncio.gather(...)`):
| Time | Event                                           |
|-------|------------------------------------------------|
| 0s    | `async_fun1` runs `time.sleep(3)` → blocks event loop |
| 3s    | `async_fun1` hits `await`, yields              |
| 3s    | `async_fun2` starts, hits `await`, yields      |
| 3s    | `async_fun3` starts, hits `await`, yields      |
| 4s    | `async_fun3` resumes and finishes               |
| 5s    | `async_fun1` and `async_fun2` resume and finish|

🕒 **Total time: 5 seconds**

---
```python
import asyncio
import time
from datetime import datetime
from threading import Thread


def fun1():
    start = datetime.now()
    print("Starting fun1 Execution")
    time.sleep(6)
    end = datetime.now()
    print(f"Time taken for execution of fun1 :: {(end - start).seconds} second(s)")


def fun2():
    start = datetime.now()
    print("Starting fun2 Execution")
    time.sleep(2)
    end = datetime.now()
    print(f"Time taken for execution of fun2 :: {(end - start).seconds} second(s)")


def fun3():
    start = datetime.now()
    print("Starting fun3 Execution")
    time.sleep(3)
    end = datetime.now()
    print(f"Time taken for execution of fun3 :: {(end - start).seconds} second(s)")


async def async_fun1():
    start = datetime.now()
    print("Starting async_fun1 Execution")
    time.sleep(3)
    await asyncio.sleep(2)
    time.sleep(1)
    end = datetime.now()
    print(f"Time taken for execution of async_fun1 :: {(end - start).seconds} second(s)")


async def async_fun2():
    start = datetime.now()
    print("Starting async_fun2 Execution")
    await asyncio.sleep(2)
    end = datetime.now()
    print(f"Time taken for execution of async_fun2 :: {(end - start).seconds} second(s)")


async def async_fun3():
    start = datetime.now()
    print("Starting async_fun3 Execution")
    time.sleep(1)
    await asyncio.sleep(2)
    end = datetime.now()
    print(f"Time taken for execution of async_fun3 :: {(end - start).seconds} second(s)")


def sync_calls():
    start = datetime.now()
    print("Starting Sync Execution of fns")
    fun1()
    fun2()
    fun3()
    end = datetime.now()
    print(f"Time taken for sync execution of fns :: {(end - start).seconds} second(s)")


async def async_calls():
    start = datetime.now()
    print("Starting async Execution of fns")
    tasks = [async_fun1(), async_fun2(), async_fun3()]
    await asyncio.gather(*tasks)
    end = datetime.now()
    print(f"Time taken for async execution of fns :: {(end - start).seconds} second(s)")


def multi_threaded():
    start = datetime.now()
    print("Starting Multi-threaded Execution of fns")
    t1 = Thread(target=fun1)
    t2 = Thread(target=fun2)
    t3 = Thread(target=fun3)
    t1.start()
    t2.start()
    t3.start()
    # t1.join()
    # t2.join()
    # t3.join()
    end = datetime.now()
    print(f"Time taken for multi-threaded execution of fns :: {(end - start).seconds} second(s)")


def runner():
    sync_calls()
    print("*********************************")
    asyncio.run(async_calls())
    print("*********************************")
    multi_threaded()
```

## 🧵 Threading vs Async

| Feature             | `threading`           | `asyncio`            |
|---------------------|-----------------------|----------------------|
| Uses OS threads?     | ✅ Yes                | ❌ No                |
| True parallelism?    | ❌ Not with CPython (GIL) | ❌ Single-threaded     |
| Blocking?           | Can be                | Avoided              |
| Context Switch Type  | Preemptive (by OS)    | Cooperative (via `await`) |

> `threading` is useful for **blocking I/O**.
> `asyncio` is best for **high-concurrency I/O-bound tasks**.

---

## 🤔 What Does “Yield Control” Mean?

> A coroutine **pauses itself** at `await`, letting the event loop run something else.

It’s:
- **Voluntary**
- **Fast**
- Done in **user space** (not kernel)
- The essence of **cooperative multitasking**

---

## 🧩 Threads — Join and Daemon Behavior

### Multi-threading with `join()`

- When you spawn threads (e.g., t1, t2, t3) alongside the main thread, these threads run concurrently (interleaved execution) but not in true parallelism due to the GIL. They share one CPU core for Python bytecode execution, and context switching happens between threads..
- If you **call `join()` on each thread**, the main thread waits for them to finish.
- The total elapsed time is approximately the **maximum time** among all threads (because they run concurrently).
- Without `join()`, the main thread finishes early, possibly before child threads complete, so time measured in main thread is very small.

### Daemon Threads

- **Daemon threads** run in the background.
- When the **main thread exits**, all daemon threads are **killed immediately**, even if they haven't finished their work.
- If you make child threads daemon and **do not `join()`**, those threads may **not complete their tasks** because they terminate as soon as the main thread ends.
- Normal (non-daemon) threads keep the program alive until they finish.

### Summary Table

| Thread Type  | Main Thread Exit Behavior                        | Need to use `join()`?                |
|--------------|------------------------------------------------|------------------------------------|
| Non-Daemon   | Program waits for threads to finish             | Optional (to explicitly wait)      |
| Daemon       | Program exits and kills daemon threads immediately | Required if you want threads to finish |

---

## 📦 Bonus: Execution Time Summary

| Approach                 | Total Time | Notes                                    |
|--------------------------|------------|------------------------------------------|
| `sync_calls()`           | 11s        | All blocking, runs in main thread         |
| `async_calls()` (with bad `time.sleep`) | 6s         | Event loop blocked for 3s                 |
| `async_calls()` (fully async) | 2s         | Efficient!                               |
| `multi_threaded()` (with join) | ~6s        | Threads overlap, but GIL limits parallelism |

---
