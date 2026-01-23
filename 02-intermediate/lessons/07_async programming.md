# Python Async Programming: A Detailed Guide

## Table of Contents
1. [Introduction to Asynchronous Programming](#introduction-to-asynchronous-programming)
2. [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
3. [Coroutines](#coroutines)
4. [The Event Loop](#the-event-loop)
5. [async and await Keywords](#async-and-await-keywords)
6. [Running Async Code](#running-async-code)
7. [Concurrent Execution](#concurrent-execution)
8. [Async Context Managers](#async-context-managers)
9. [Async Iterators and Generators](#async-iterators-and-generators)
10. [Error Handling](#error-handling)
11. [When to Use Async](#when-to-use-async)
12. [Practical Examples](#practical-examples)

---

## Introduction to Asynchronous Programming

Asynchronous programming allows a program to handle multiple operations concurrently without using multiple threads. It's particularly useful for I/O-bound operations like network requests, file operations, or database queries.

### Key Concepts

- **Concurrency**: Multiple tasks making progress without necessarily running simultaneously
- **Parallelism**: Multiple tasks running simultaneously (requires multiple cores)
- **Async/Await**: Python's syntax for writing asynchronous code
- **Coroutines**: Special functions that can pause and resume execution
- **Event Loop**: Manages and executes asynchronous tasks

---

## Synchronous vs Asynchronous

### Synchronous Code (Blocking)

```python
import time

def download_file(filename):
    print(f"Starting download: {filename}")
    time.sleep(2)  # Simulates I/O operation
    print(f"Completed download: {filename}")
    return f"Data from {filename}"

# Sequential execution - takes 6 seconds total
start = time.time()
result1 = download_file("file1.txt")
result2 = download_file("file2.txt")
result3 = download_file("file3.txt")
end = time.time()

print(f"Total time: {end - start:.2f} seconds")
# Output:
# Starting download: file1.txt
# Completed download: file1.txt
# Starting download: file2.txt
# Completed download: file2.txt
# Starting download: file3.txt
# Completed download: file3.txt
# Total time: 6.00 seconds
```

### Asynchronous Code (Non-Blocking)

```python
import asyncio
import time

async def download_file(filename):
    print(f"Starting download: {filename}")
    await asyncio.sleep(2)  # Non-blocking sleep
    print(f"Completed download: {filename}")
    return f"Data from {filename}"

async def main():
    start = time.time()
    
    # Concurrent execution - takes ~2 seconds total
    results = await asyncio.gather(
        download_file("file1.txt"),
        download_file("file2.txt"),
        download_file("file3.txt")
    )
    
    end = time.time()
    print(f"Total time: {end - start:.2f} seconds")
    return results

# Run the async code
asyncio.run(main())

# Output:
# Starting download: file1.txt
# Starting download: file2.txt
# Starting download: file3.txt
# Completed download: file1.txt
# Completed download: file2.txt
# Completed download: file3.txt
# Total time: 2.00 seconds
```

---

## Coroutines

Coroutines are functions defined with `async def` that can pause execution using `await` and resume later.

### Defining Coroutines

```python
import asyncio

# Regular function
def regular_function():
    return "Hello"

# Coroutine function
async def coroutine_function():
    return "Hello"

# Calling them
print(regular_function())  # Hello

# Calling a coroutine returns a coroutine object
coro = coroutine_function()
print(coro)  # <coroutine object coroutine_function at 0x...>

# Must use await or asyncio.run()
result = asyncio.run(coroutine_function())
print(result)  # Hello
```

### Coroutine Characteristics

```python
import asyncio

async def simple_coroutine():
    print("Coroutine started")
    await asyncio.sleep(1)
    print("Coroutine resumed")
    return "Done"

async def main():
    print("Before calling coroutine")
    result = await simple_coroutine()
    print(f"Result: {result}")
    print("After coroutine")

asyncio.run(main())

# Output:
# Before calling coroutine
# Coroutine started
# Coroutine resumed
# Result: Done
# After coroutine
```

### Awaitable Objects

Only certain objects can be awaited:
- Coroutines (functions defined with `async def`)
- Tasks (created with `asyncio.create_task()`)
- Futures (low-level awaitable objects)

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "Result"

async def main():
    # Can await coroutines
    result = await my_coroutine()
    print(result)
    
    # Can await tasks
    task = asyncio.create_task(my_coroutine())
    result = await task
    print(result)

asyncio.run(main())
```

---

## The Event Loop

The event loop is the core of async programming. It runs async tasks, executes callbacks, and manages I/O operations.

### Understanding the Event Loop

```python
import asyncio

async def task1():
    print("Task 1 started")
    await asyncio.sleep(2)
    print("Task 1 completed")
    return "Result 1"

async def task2():
    print("Task 2 started")
    await asyncio.sleep(1)
    print("Task 2 completed")
    return "Result 2"

async def main():
    print("Starting event loop")
    
    # The event loop manages both tasks concurrently
    results = await asyncio.gather(task1(), task2())
    
    print(f"Results: {results}")

asyncio.run(main())

# Output:
# Starting event loop
# Task 1 started
# Task 2 started
# Task 2 completed  (after 1 second)
# Task 1 completed  (after 2 seconds)
# Results: ['Result 1', 'Result 2']
```

### Event Loop Visualization

```python
import asyncio
import time

async def worker(name, delay):
    print(f"{time.strftime('%H:%M:%S')} - {name}: Starting")
    await asyncio.sleep(delay)
    print(f"{time.strftime('%H:%M:%S')} - {name}: Finished")
    return f"{name} result"

async def main():
    print(f"{time.strftime('%H:%M:%S')} - Main: Starting")
    
    # All tasks run concurrently on the event loop
    tasks = [
        worker("Worker-1", 3),
        worker("Worker-2", 1),
        worker("Worker-3", 2)
    ]
    
    results = await asyncio.gather(*tasks)
    print(f"{time.strftime('%H:%M:%S')} - Main: All done")
    print(f"Results: {results}")

asyncio.run(main())

# Output shows concurrent execution:
# 14:30:00 - Main: Starting
# 14:30:00 - Worker-1: Starting
# 14:30:00 - Worker-2: Starting
# 14:30:00 - Worker-3: Starting
# 14:30:01 - Worker-2: Finished
# 14:30:02 - Worker-3: Finished
# 14:30:03 - Worker-1: Finished
# 14:30:03 - Main: All done
```

---

## async and await Keywords

### The `async` Keyword

Defines a coroutine function.

```python
# Async function
async def fetch_data():
    return "data"

# Async method
class DataFetcher:
    async def fetch(self):
        return "data"

# Async lambda (not allowed)
# async_lambda = async lambda x: x  # SyntaxError
```

### The `await` Keyword

Pauses execution until the awaited coroutine completes.

```python
import asyncio

async def step1():
    print("Step 1: Starting")
    await asyncio.sleep(1)
    print("Step 1: Done")
    return "Result 1"

async def step2():
    print("Step 2: Starting")
    await asyncio.sleep(1)
    print("Step 2: Done")
    return "Result 2"

async def main():
    # Sequential execution (2 seconds total)
    result1 = await step1()
    result2 = await step2()
    print(f"Sequential: {result1}, {result2}")
    
    # Concurrent execution (1 second total)
    result1, result2 = await asyncio.gather(step1(), step2())
    print(f"Concurrent: {result1}, {result2}")

asyncio.run(main())
```

### Common Mistake: Forgetting `await`

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "data"

async def main():
    # Wrong: Returns coroutine object, doesn't execute
    result = fetch_data()
    print(result)  # <coroutine object fetch_data at 0x...>
    
    # Correct: Actually executes the coroutine
    result = await fetch_data()
    print(result)  # data

asyncio.run(main())
```

---

## Running Async Code

### Using `asyncio.run()`

The simplest way to run async code (Python 3.7+).

```python
import asyncio

async def main():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# Run the main coroutine
asyncio.run(main())
```

### Using `asyncio.create_task()`

Create tasks that run concurrently.

```python
import asyncio

async def say_after(delay, message):
    await asyncio.sleep(delay)
    print(message)

async def main():
    # Create tasks (starts running immediately)
    task1 = asyncio.create_task(say_after(1, "Hello"))
    task2 = asyncio.create_task(say_after(2, "World"))
    
    print("Tasks created")
    
    # Wait for both tasks to complete
    await task1
    await task2
    
    print("All done")

asyncio.run(main())

# Output:
# Tasks created
# Hello        (after 1 second)
# World        (after 2 seconds)
# All done
```

### Getting Task Results

```python
import asyncio

async def calculate(x, y):
    await asyncio.sleep(1)
    return x + y

async def main():
    task1 = asyncio.create_task(calculate(5, 3))
    task2 = asyncio.create_task(calculate(10, 20))
    
    # Wait and get results
    result1 = await task1
    result2 = await task2
    
    print(f"Results: {result1}, {result2}")

asyncio.run(main())
# Output: Results: 8, 30
```

---

## Concurrent Execution

### Using `asyncio.gather()`

Run multiple coroutines concurrently and collect results.

```python
import asyncio
import time

async def fetch_data(id, delay):
    print(f"Fetching data {id}")
    await asyncio.sleep(delay)
    return f"Data {id}"

async def main():
    start = time.time()
    
    # All run concurrently
    results = await asyncio.gather(
        fetch_data(1, 2),
        fetch_data(2, 1),
        fetch_data(3, 3)
    )
    
    end = time.time()
    print(f"Results: {results}")
    print(f"Time taken: {end - start:.2f}s")  # ~3s (not 6s)

asyncio.run(main())
```

### Using `asyncio.wait()`

More control over waiting for tasks.

```python
import asyncio

async def task(name, duration):
    await asyncio.sleep(duration)
    return f"{name} completed"

async def main():
    tasks = [
        asyncio.create_task(task("Task-1", 2)),
        asyncio.create_task(task("Task-2", 1)),
        asyncio.create_task(task("Task-3", 3))
    ]
    
    # Wait for all tasks to complete
    done, pending = await asyncio.wait(tasks)
    
    for task in done:
        print(task.result())

asyncio.run(main())
```

### Using `asyncio.as_completed()`

Process results as they complete.

```python
import asyncio

async def fetch_data(id, delay):
    await asyncio.sleep(delay)
    return f"Data {id} (took {delay}s)"

async def main():
    tasks = [
        fetch_data(1, 3),
        fetch_data(2, 1),
        fetch_data(3, 2)
    ]
    
    # Process results in order of completion
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"Completed: {result}")

asyncio.run(main())

# Output (order based on completion):
# Completed: Data 2 (took 1s)
# Completed: Data 3 (took 2s)
# Completed: Data 1 (took 3s)
```

---

## Async Context Managers

Context managers that support async operations.

### Defining Async Context Managers

```python
import asyncio

class AsyncResource:
    async def __aenter__(self):
        print("Acquiring resource")
        await asyncio.sleep(1)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Releasing resource")
        await asyncio.sleep(1)
        return False

async def main():
    async with AsyncResource() as resource:
        print("Using resource")
        await asyncio.sleep(1)

asyncio.run(main())

# Output:
# Acquiring resource
# Using resource
# Releasing resource
```

### Using `@asynccontextmanager`

```python
import asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_file_handler(filename):
    print(f"Opening {filename}")
    await asyncio.sleep(1)  # Simulate async operation
    
    try:
        yield f"Handle to {filename}"
    finally:
        print(f"Closing {filename}")
        await asyncio.sleep(1)

async def main():
    async with async_file_handler("data.txt") as handle:
        print(f"Working with {handle}")
        await asyncio.sleep(1)

asyncio.run(main())
```

---

## Async Iterators and Generators

### Async Iterators

```python
import asyncio

class AsyncCounter:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.current >= self.end:
            raise StopAsyncIteration
        
        await asyncio.sleep(0.5)  # Simulate async operation
        value = self.current
        self.current += 1
        return value

async def main():
    async for num in AsyncCounter(1, 5):
        print(num)

asyncio.run(main())

# Output (with 0.5s delay between each):
# 1
# 2
# 3
# 4
```

### Async Generators

```python
import asyncio

async def async_range(start, end):
    for i in range(start, end):
        await asyncio.sleep(0.5)
        yield i

async def main():
    async for num in async_range(1, 5):
        print(num)

asyncio.run(main())
```

### Async Comprehensions

```python
import asyncio

async def async_square(x):
    await asyncio.sleep(0.1)
    return x ** 2

async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def main():
    # Async list comprehension
    squares = [await async_square(x) async for x in async_range(5)]
    print(squares)  # [0, 1, 4, 9, 16]
    
    # Async generator expression
    gen = (await async_square(x) async for x in async_range(5))
    async for value in gen:
        print(value)

asyncio.run(main())
```

---

## Error Handling

### Try-Except with Async

```python
import asyncio

async def risky_operation():
    await asyncio.sleep(1)
    raise ValueError("Something went wrong!")

async def main():
    try:
        await risky_operation()
    except ValueError as e:
        print(f"Caught error: {e}")

asyncio.run(main())
# Output: Caught error: Something went wrong!
```

### Handling Errors in `gather()`

```python
import asyncio

async def task_that_fails():
    await asyncio.sleep(1)
    raise ValueError("Task failed!")

async def task_that_succeeds():
    await asyncio.sleep(1)
    return "Success"

async def main():
    # By default, gather() raises the first exception
    try:
        results = await asyncio.gather(
            task_that_succeeds(),
            task_that_fails(),
            task_that_succeeds()
        )
    except ValueError as e:
        print(f"Error: {e}")
    
    # Use return_exceptions=True to collect exceptions
    results = await asyncio.gather(
        task_that_succeeds(),
        task_that_fails(),
        task_that_succeeds(),
        return_exceptions=True
    )
    
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Task {i} failed: {result}")
        else:
            print(f"Task {i} succeeded: {result}")

asyncio.run(main())

# Output:
# Error: Task failed!
# Task 0 succeeded: Success
# Task 1 failed: Task failed!
# Task 2 succeeded: Success
```

### Timeout Handling

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(5)
    return "Done"

async def main():
    try:
        # Wait maximum 2 seconds
        result = await asyncio.wait_for(slow_operation(), timeout=2)
        print(result)
    except asyncio.TimeoutError:
        print("Operation timed out!")

asyncio.run(main())
# Output: Operation timed out!
```

---

## When to Use Async

### Good Use Cases (I/O-Bound Operations)

✅ **Network requests**
```python
import asyncio
import aiohttp

async def fetch_url(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def main():
    urls = [
        "https://example.com",
        "https://example.org",
        "https://example.net"
    ]
    
    results = await asyncio.gather(*[fetch_url(url) for url in urls])
    print(f"Fetched {len(results)} pages")

# asyncio.run(main())  # Commented to avoid actual requests
```

✅ **Database queries**
```python
# Example with async database library
async def fetch_users():
    # Using async database driver
    users = await db.execute("SELECT * FROM users")
    return users
```

✅ **File I/O operations**
```python
import aiofiles

async def read_file(filename):
    async with aiofiles.open(filename, 'r') as f:
        contents = await f.read()
        return contents
```

### Bad Use Cases (CPU-Bound Operations)

❌ **Heavy computations**
```python
# Don't use async for CPU-intensive tasks
async def calculate_primes(n):
    # This blocks the event loop!
    primes = []
    for num in range(2, n):
        is_prime = True
        for i in range(2, int(num ** 0.5) + 1):
            if num % i == 0:
                is_prime = False
                break
        if is_prime:
            primes.append(num)
    return primes

# For CPU-bound tasks, use multiprocessing or threading instead
```

### Decision Guide

Use **async/await** when:
- Making network requests (APIs, web scraping)
- Reading/writing files
- Database operations
- WebSocket connections
- Multiple I/O operations that can wait independently

Use **threading** when:
- I/O-bound operations with libraries that don't support async
- Need true parallelism with blocking I/O

Use **multiprocessing** when:
- CPU-intensive computations
- Need true parallelism for computation

---

## Practical Examples

### Web Scraping

```python
import asyncio
import time

async def fetch_page(url, session_id):
    print(f"Session {session_id}: Fetching {url}")
    await asyncio.sleep(2)  # Simulate network delay
    print(f"Session {session_id}: Completed {url}")
    return f"Content from {url}"

async def main():
    urls = [
        "https://example.com/page1",
        "https://example.com/page2",
        "https://example.com/page3",
        "https://example.com/page4",
        "https://example.com/page5"
    ]
    
    start = time.time()
    
    # Fetch all pages concurrently
    results = await asyncio.gather(
        *[fetch_page(url, i) for i, url in enumerate(urls, 1)]
    )
    
    end = time.time()
    print(f"\nFetched {len(results)} pages in {end - start:.2f} seconds")

asyncio.run(main())
```

### Rate-Limited API Calls

```python
import asyncio

class RateLimiter:
    def __init__(self, rate, per):
        self.rate = rate
        self.per = per
        self.allowance = rate
        self.last_check = asyncio.get_event_loop().time()
    
    async def acquire(self):
        current = asyncio.get_event_loop().time()
        time_passed = current - self.last_check
        self.last_check = current
        self.allowance += time_passed * (self.rate / self.per)
        
        if self.allowance > self.rate:
            self.allowance = self.rate
        
        if self.allowance < 1.0:
            sleep_time = (1.0 - self.allowance) * (self.per / self.rate)
            await asyncio.sleep(sleep_time)
            self.allowance = 0.0
        else:
            self.allowance -= 1.0

async def make_api_call(limiter, call_id):
    await limiter.acquire()
    print(f"Making API call {call_id}")
    await asyncio.sleep(0.5)  # Simulate API call
    return f"Result {call_id}"

async def main():
    # Allow 2 requests per second
    limiter = RateLimiter(rate=2, per=1.0)
    
    # Make 10 API calls
    results = await asyncio.gather(
        *[make_api_call(limiter, i) for i in range(10)]
    )
    
    print(f"Completed {len(results)} calls")

asyncio.run(main())
```

### Producer-Consumer Pattern

```python
import asyncio
import random

async def producer(queue, producer_id):
    for i in range(5):
        item = f"Item-{producer_id}-{i}"
        await queue.put(item)
        print(f"Producer {producer_id} produced: {item}")
        await asyncio.sleep(random.uniform(0.1, 0.5))
    
    await queue.put(None)  # Signal completion

async def consumer(queue, consumer_id):
    while True:
        item = await queue.get()
        
        if item is None:
            queue.task_done()
            break
        
        print(f"Consumer {consumer_id} consuming: {item}")
        await asyncio.sleep(random.uniform(0.2, 0.6))
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    
    # Create 2 producers and 3 consumers
    producers = [
        asyncio.create_task(producer(queue, i))
        for i in range(2)
    ]
    
    consumers = [
        asyncio.create_task(consumer(queue, i))
        for i in range(3)
    ]
    
    # Wait for producers to finish
    await asyncio.gather(*producers)
    
    # Wait for queue to be empty
    await queue.join()
    
    # Cancel consumers
    for c in consumers:
        c.cancel()

asyncio.run(main())
```

---

## Key Takeaways

- **Async/await** enables concurrent I/O operations without threading complexity
- **Coroutines** are special functions that can pause and resume execution
- **Event loop** manages and schedules all async operations
- Use `asyncio.run()` to run async code from synchronous code
- Use `asyncio.gather()` to run multiple coroutines concurrently
- Use `asyncio.create_task()` to schedule coroutines for execution
- **Async is best for I/O-bound operations**, not CPU-bound computations
- Always use `await` when calling coroutines (forgetting it is a common mistake)
- Handle errors with try-except and use `return_exceptions=True` in `gather()`
- Async context managers use `async with` and define `__aenter__`/`__aexit__`
- Modern Python favors async for web servers, API clients, and I/O-heavy applications

Mastering async programming enables you to write highly efficient, scalable applications that handle many concurrent operations!