---
layout: post
title: "Request Hedging: Fighting Tail Latencies"
date: 2024-06-10
categories: [distributed-systems, system-design]
tags: [system-design, distributed-systems, latency, performance, python, asyncio]
---

# Request Hedging: Fighting Tail Latencies

In systems with low latency SLA requirements, it becomes very important to address the tail latencies as it can have a significant impact on users.

Imagine you have multiple services in a request path. Each request adds up its own latency to the overall response.

Request hedging offers a simple solution to this problem by sending multiple duplicate requests and using the first response that comes back.

## Understanding Request Hedging

Request hedging is conceptually simple: instead of waiting for a single request to complete, you send multiple identical requests (potentially to different backend servers) and use whichever response arrives first.

## When to Use Request Hedging

Hedging is particularly valuable when:

- **Tail latency matters**: In user-facing applications where consistent response times are crucial
- **Resources aren't constrained**: You have spare capacity to handle duplicate requests
- **Operations are idempotent**: The same request can be processed multiple times without side effects
- **Backend variability exists**: Some servers may be slower than others due to resource contention, garbage collection, etc.

## How Hedging Differs from Retries

While both strategies involve sending multiple requests, there's a key difference:

- **Retries**: Send a second request only after the first one fails or times out
- **Hedging**: Send additional requests proactively, without waiting for the first to fail

This distinction makes hedging much more effective at reducing tail latency.

## Hedging Strategies

Several variations of request hedging exist:

- **Naive hedging**: Send multiple requests simultaneously
- **Delayed hedging**: Send the first request, then send a second after a delay if the first hasn't completed
- **Adaptive hedging**: Adjust hedging behavior based on recent performance patterns

## Implementing Request Hedging with Python Asyncio

Let's implement a delayed hedging strategy using Python's asyncio:

```python
import asyncio
import time
import aiohttp
import logging
from typing import List, TypeVar, Callable, Awaitable, Optional, Any

T = TypeVar('T')

async def hedged_request(
    request_fn: Callable[[], Awaitable[T]],
    hedge_delay: float = 0.1,
    max_hedges: int = 3,
    timeout: Optional[float] = None
) -> T:
    """
    Execute a request with hedging.

    Args:
        request_fn: Async function that makes the actual request
        hedge_delay: Seconds to wait before sending each additional request
        max_hedges: Maximum number of concurrent requests (including the original)
        timeout: Total timeout for all hedged requests

    Returns:
        Result from the first successful request

    Raises:
        asyncio.TimeoutError: If all requests time out
        Exception: Any exception raised by all request attempts
    """

    # Keep track of sent requests and their tasks
    hedge_tasks = []
    request_count = 0
    exceptions = []

    # Set up a flag to track if we've already returned a result
    got_result = False

    async def execute_request(request_id: int) -> T:
        nonlocal got_result

        try:
            start_time = time.time()
            logging.info(f"Starting request {request_id}")

            result = await request_fn()

            # If we haven't already returned a result, log the success
            if not got_result:
                got_result = True
                duration = time.time() - start_time
                logging.info(f"Request {request_id} succeeded after {duration:.3f}s")

            return result

        except Exception as e:
            logging.warning(f"Request {request_id} failed with: {e}")
            exceptions.append(e)
            raise

    # Create the initial request
    request_count += 1
    main_task = asyncio.create_task(execute_request(request_count))
    hedge_tasks.append(main_task)

    # Set up the overall timeout if specified
    if timeout:
        deadline = asyncio.create_task(asyncio.sleep(timeout))
        hedge_tasks.append(deadline)

    # Main hedging loop
    while request_count < max_hedges:
        # Wait for either a result or the hedge delay
        delay_task = asyncio.create_task(asyncio.sleep(hedge_delay))

        # Create a list of pending tasks to await
        pending = [t for t in hedge_tasks if not t.done()] + [delay_task]

        if not pending:
            # All tasks have completed or failed
            break

        # Wait for the first task to complete
        done, pending = await asyncio.wait(
            pending,
            return_when=asyncio.FIRST_COMPLETED
        )

        # Cancel the delay task if it's still pending
        if not delay_task.done():
            delay_task.cancel()

        # Check if we have a deadline and it's done
        if timeout and deadline in done:
            # Cancel all other tasks
            for task in hedge_tasks:
                if not task.done() and task != deadline:
                    task.cancel()
            raise asyncio.TimeoutError("All hedged requests timed out")

        # Check if we got a result from an existing task
        for task in done:
            if task in hedge_tasks and task != deadline:
                try:
                    result = task.result()

                    # Cancel all other pending tasks
                    for t in hedge_tasks:
                        if not t.done() and t != deadline and t != task:
                            t.cancel()

                    return result
                except Exception:
                    # This task failed, continue with hedging
                    pass

        # If we've reached the hedge delay and haven't returned yet, send another request
        if delay_task in done:
            request_count += 1
            hedge_task = asyncio.create_task(execute_request(request_count))
            hedge_tasks.append(hedge_task)

    # Wait for any remaining tasks
    remaining = [t for t in hedge_tasks if t != deadline and not t.done()]
    if remaining:
        done, pending = await asyncio.wait(remaining, return_when=asyncio.FIRST_COMPLETED)

        # Check if any succeeded
        for task in done:
            try:
                return task.result()
            except Exception:
                # Task failed, continue
                pass

    # If we get here, all requests failed
    if exceptions:
        raise exceptions[0]  # Re-raise the first exception
    else:
        raise RuntimeError("All hedged requests failed with unknown errors")
```

Now let's create a practical example using this implementation to make HTTP requests:

```python
async def fetch_with_hedging(url: str) -> str:
    """Fetch a URL with request hedging."""

    # Define our actual request function
    async def make_request():
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                response.raise_for_status()
                return await response.text()

    # Use our hedging function
    return await hedged_request(
        request_fn=make_request,
        hedge_delay=0.1,  # Send a new request after 100ms
        max_hedges=3,     # Send at most 3 requests
        timeout=2.0       # Total timeout of 2 seconds
    )
```

## Real-World Considerations

While request hedging can dramatically improve tail latency, it comes with important trade-offs:

### 1. Increased Load

By definition, hedging generates additional load on your backends. This amplification factor needs careful consideration:

- Start with minimal hedging (2 total requests)
- Monitor backend load closely
- Consider only hedging a percentage of requests
- Implement circuit breakers to stop hedging during high load

### 2. Idempotent Operation

Hedging is only safe when the operation itself is idempotent.

### 3. Adaptive Hedging

In production systems, adding adaptive hedging helps scenarios to handle bursts:

- Only hedges when response times exceed certain percentiles
- Use hedging selectively for faster instances

## Conclusion

Request hedging is a simple but powerful approach for improving tail latency in systems. By sending multiple requests and using the first response, you can significantly reduce the impact of occasional slowdowns and provide a more consistent user experience.
