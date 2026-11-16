---
title: "Writing Python like it’s Elixir (or Erlang)"
date: "2024-02-11T18:37:57Z"
description: "The OTP framework, the most wonderful toolbox of Erlang/Elixir, in Python, is it possible?"
tags: ["programming", "python", "erlang", "elixir", "design-patterns"]
author: "David Delassus"
images: ["img/image-7b20b3508ee34ba3.png"]
---

![](img/image-7b20b3508ee34ba3.png)

---

Elixir must be one of my favorite language, both in terms of feature-set and philosophy. I consider the BEAM (the Erlang VM) to be one of the best piece of technology we have, that is my entirely subjective opinion and nothing you can say will ever change it 🙂

I have this opinion because of 3 things mainly:

- the concurrency model, often wrongly called Actor model¹
- the ability to interoperate with every language compiling to the BEAM bytecode²
- the OTP framework, the set of tools that allow you to build your distributed system³

Notes:

1. For more informations, see: <https://softwareengineering.stackexchange.com/a/277469>
2. Erlang, Elixir, Gleam, etc… You can call code written in each of those languages, from each of those languages.
3. It’s important to note that OTP doesn’t give you a distributed system, but only the tools to build one (those tools are great though).

Designing the architecture of a system by splitting it into multiple, simple components that can talk to each other via message-passing makes each of those components easily testable, and generally easier to reason about.

Learning Erlang and Elixir made me rethink how I would design software even in other languages. This is even a source of inspiration for my own programming language [Letlang](https://letlang.dev).

This is what I will talk to you about in this article, how can you take inspiration from Erlang / Elixir, and apply their philosophy to another programming language. Let’s chose another one I’m fond of: **Python**.

---

## Why Python?

Python evolved a lot in the last 10 years, it now has:

- concurrency (with `async` and `await` keywords)
- pattern matching (with the `match` statement)
- annotations, used for dependency injection, or type hinting
- a great ecosystem

While not on par with their Erlang/Elixir equivalent, the concurrency and pattern matching of Python are tools that I require for this endeavor. In fact, those are tools I now expect to find in every language I want to use.

As for type annotations, I mainly use them for documentation purpose, or dependency injection in frameworks like FastAPI. I don’t care much about MyPy and static type checking here, this is Python, not Rust. It’s a **hint** not a guarantee 😉

Python’s default async loop implementation is provided by the module `asyncio`, but we won’t be using this one today. Instead, we will use `trio`:

{{< cta-link
  url="https://github.com/python-trio/trio"
  title="GitHub - python-trio/trio: Trio - a friendly Python library for async concurrency and I/O"
  description="Trio - a friendly Python library for async concurrency and I/O - GitHub - python-trio/trio: Trio - a friendly Python…"
>}}

## A short introduction to “trio”

As said just above, `trio` is an alternative implementation to Python’s asynchronous loop (which backs the use of `async` / `await` ). without going too much into details, every async function returns a `Coroutine` object, and the async loop is in charge of scheduling their execution until the next `await` (which acts as a breakpoint in the async function). This is how we can use a single thread to run multiple tasks concurrently.

**NB:** It should be possible to run async tasks in different threads, but thread safety features are required for that. To my knowledge, Python do not have them. If that is something you want, I might recommend you to go for Rust with Tokio.

So, let’s start with a simple “Hello world”:

```python
import trio


async def greet(name: str):
    await trio.sleep(1)
    print("Hello", name)


trio.run(greet, "world")
```

Pretty easy so far. We use `trio.run` to start an async loop, and we can call async functions with `await`.

**NB:** You can’t call `asyncio` functions within a `trio` context. If you need to use a library made for `asyncio`, you may want to use `trio-asyncio`, or `anyio`, or equivalent.

Now, the best feature of `trio` is the **nursery**. The nursery allows you to run multiple child async tasks, and group them in a **cancellation scope**. If one task is cancelled (or crashes), the nursery takes care of terminating the other tasks it manages.

```python
import random
import trio


async def may_crash(luck: float):
    while True:
        if random.random() < luck:
            raise Exception("i don't work with unlucky functions")

        await trio.sleep(1)


async def main():
    try:
        async with trio.open_nursery() as nursery:
            nursery.start_soon(may_crash, 0.0)
            nursery.start_soon(may_crash, 0.0)
            nursery.start_soon(may_crash, 0.1)

    except Exception as err:
        print(err)


trio.run(main)
```

The async context manager that is the nursery will only terminate when all children tasks have terminated (successfully). If one task raises an exception, other tasks in the nursery are cancelled.

You can even cancel the nursery yourself:

```python
nursery.cancel_scope.cancel()
```

For the cherry on top of the cake, you can control when tasks are started.  
The `nursery.start_soon()` method will add the task to the nursery and move on without blocking. But there is also the `nursery.start()` async method:

```python
import trio


async def foo_service(task_status=trio.TASK_STATUS_IGNORED):
    await trio.sleep(1)      # some slow initialization stuff
    task_status.started(42)  # notify the nursery that we are started

    while True:
        # do work
        await trio.sleep(1)


async def main():
    async with trio.open_nursery() as nursery:
        # will block until task_status.started() is called
        val = await nursery.start(foo_service)
        # val = 42 here
```

Useful if you want to start an HTTP server only after your connection to the database is up and running (to avoid failing requests for example).

Nurseries are the first step to split a complex system into multiple small components. Each async task can be viewed as an “actor”, and the nurseries help manage their lifecycle.

---

## Message passing with “trio”

A great pattern to decouple the components of your system. To implement this, we need “channels”, which are basically a queue, with a receiver to consume it, and a producer (or multiple producers) to send messages to it.

`trio` provides this with the `trio.open_memory_channel()` function:

```python
import trio


async def procuder(tx: trio.MemorySendChannel[str]):
    await tx.send("hello")  # blocks until the message is received
    tx.send_nowait("world") # non-blocking
    await tx.aclose()


async def consumer(rx: trio.MemoryReceiveChannel[str]):
    # wait for messages until all producers are closed
    async for message in rx:
        print(message)


async def main():
    tx, rx = trio.open_memory_channel[str](0)

    async with trio.open_nursery() as nursery:
        nursery.start_soon(consumer, rx)
        nursery.start_soon(producer, tx.clone())
        nursery.start_soon(producer, tx.clone())
        await tx.aclose()


trio.run(main)
```

**NB:** The `[str]` here is entirely optional, this is for type hinting only.

With this pattern, we can implement a client/server:

```python
import trio


# Python 3.12 type statements
type EchoResponse = str
type EchoRequest = tuple[trio.MemorySendChannel[EchoResponse], str]


async def echo_client(chan: trio.MemorySendChannel[EchoRequest]):
    resp_tx, resp_rx = trio.open_memory_channel[EchoResponse](0)

    req = (resp_tx.clone(), "hello")
    await chan.send(req)
    resp = await resp_rx.receive()
    await resp_tx.aclose()

    print("Client:", resp)

    await chan.aclose()


async def echo_server(chan: trio.MemoryReceiveChannel[EchoRequest]):
    async for (respond_to, payload) in chan:
        print("Server:", payload)
        await respond_to.send(payload)
        await respond_to.aclose()


async def main():
    chan_tx, chan_rx = trio.open_memory_channel[EchoRequest](0)

    async with trio.open_nursery() as nursery:
        nursery.start_soon(echo_server, chan_rx)
        nursery.start_soon(echo_client, chan_tx)


trio.run(main)
```

**NB:** Rust’s `tokio` library provide `mpsc::channel()` and `oneshot::channel()` for the same purpose.

We can make use of the `task_started` argument to create the channel within the server task:

```csharp
async def echo_server(task_status=trio.TASK_STATUS_IGNORED):
    chan_tx, chan_rx = trio.open_memory_channel[EchoRequest](0)
    task_status.started(chan_tx)

    async for (respond_to, payload) in chan_rx:
        await respond_to.send(payload)
        await respond_to.aclose()


async def main():
    async with trio.open_nursery() as nursery:
        chan_tx = await nursery.start(echo_server)
        nursery.start_soon(echo_client, chan_tx)
```

But, creating the server and sending a request to it is still a bit verbose. Let’s make it a bit more generic and reusable:

```python
from functools import wraps
import trio


class GenChannel:
    def __init__(self, tx):
        self.tx = tx

    def clone(self):
        return GenChannel(self.tx.clone())

    async def close(self):
        await self.tx.aclose()

    async def call(self, payload):
        resp_tx, resp_rx = trio.open_memory_channel(0)

        req = (resp_tx.clone(), payload)
        await self.tx.send(req)

        resp = await resp_rx.receive()
        await resp_tx.aclose()

        if isinstance(resp, Exception):
            raise resp

        return resp


def gen_server(task):
    @wraps(task)
    async def server(task_status=trio.TASK_STATUS_IGNORED, *args, **kwargs):
        chan_tx, chan_rx = trio.open_memory_channel(0)
        task_status.started(GenChannel(chan_tx))
        await task(chan_rx, *args, **kwargs)

    return server
```

Which can then be used to refactor, once again, our echo client/server:

```python
async def echo_client(chan: GenChannel):
    resp = await chan.call("hello")
    print("Client:", resp)
    await chan.close()


@gen_server
async def echo_server(chan_rx):
    async for (respond_to, message) in chan_rx:
        print("Server:", message)
        await respond_to.send(message)
        await respond_to.aclose()


async def main():
    async with trio.open_nursery() as nursery:
        chan = await nursery.start(echo_server)
        nursery.start_soon(echo_client, chan)
```

You’ll notice how the `GenChannel.call()` method raises the response if it is an exception, this is so that the server can return an error to the caller instead of crashing:

```python
@gen_server
async def kvstore(chan_rx):
    async for (respond_to, message) in chan_rx:
        try:
            # request to a remote database
            resp = "some result"

        except Exception as err:
            await respond_to.send(err)

        else:
            await respond_to.send(resp)

        await respond_to.aclose()
```

This is a common pattern in Erlang/Elixir, but instead of exceptions, they return a tuple of the form `(:ok, val)` or `(:error, reason)`. Not having the server crash allows it to still serve requests to other clients. Of course, if the error is internal to the server, it is ok to let it crash. In HTTP, this would be the difference between a `4XX` status code, and a `5XX` status code.

---

## Retrying with tenacity

In Erlang/Elixir, there is the philosophy known as “let it crash”. While it has a quite scary name, the idea behind it is quite simple. When a system grows more and more complex, it becomes harder to have a complete understanding of the possible states the system as a whole can be in. When errors happen, it can enter unexpected (or worse, invalid) states. In that case, it is better to restart, in order to come back in a known, valid state.

Erlang/Elixir, as usual, offer you tools to deal with this, **the supervision tree**. This allow you to restart only parts of your system, giving you fault tolerance at every abstraction level.

{{< cta-link
  url="https://yiming.dev/blog/2022/07/10/how-let-it-fail-leads-to-simpler-code/"
  title="How \"let it fail\" leads to simpler code - Yiming Chen"
  description="The biggest lesson I learned from the \"Let it fail\" philosophy, is to distinguish expected errors and unexpected…"
>}}

I highly recommend reading the book “[Learn you some Erlang for greater good](https://learnyousomeerlang.com/)” even if you don’t intend to develop in Erlang or Elixir (and I highly recommend reading it even if you plan to learn Elixir and not Erlang). Especially the chapter “Who supervises the supervisors?”

{{< cta-link
  url="https://learnyousomeerlang.com/supervisors"
  title="Who Supervises The Supervisors? | Learn You Some Erlang for Great Good!"
  description="A tour of OTP supervisors. We see how to supervise an OTP process with all the different restart strategies. Practical…"
>}}

To implement something similar in Python, we will use the great `tenacity` library, which is used to implement retry strategies for your functions.

{{< cta-link
  url="https://github.com/jd/tenacity"
  title="GitHub - jd/tenacity: Retrying library for Python"
  description="Retrying library for Python. Contribute to jd/tenacity development by creating an account on GitHub."
>}}

Let’s dive in some examples:

```python
from tenacity import retry


@retry
def retry_forever():
    raise RuntimeError("oops")
```

This simple decorator will catch the exception and retry executing the function. Since we passed no specific configuration, this will retry forever.

But retrying forever is not what we usually want. There are multiple common strategies:

**Retry a specific number of times:**

```python
@retry(stop=stop_after_attempt(5))
def foo():
    raise RuntimeError("oops")
```

**Retry for a specific delay (in seconds):**

```python
@retry(stop=stop_after_delay(10))
def foo():
    raise RuntimeError("oops")
```

**Combination of the previous 2 examples:**

```python
@retry(stop=stop_after_attempt(5) | stop_after_delay(10))
def foo():
    raise RuntimeError("oops")
```

When dealing with network problems, we usually want to wait a certain amount of time between each retry, with some exponantial backoff and some jitter in between retries, `tenacity` allows you to do just that:

```python
@retry(wait=wait_random_exponential(multiplier=1, max=60))
def foo():
    raise RuntimeError("oops")
```

For more details about retry strategies, I highly recommend reading this interactive article:

{{< cta-link
  url="https://encore.dev/blog/retries"
  title="Retries - An interactive study of common retry methods - Encore Blog"
  description="Visually explore different methods of retrying requests to learn why some methods are dangerous and what the best…"
>}}

In Erlang/Elixir, a supervisor will stop trying to restart a child task if it has crashed more than `x` times in the last `y` seconds. Each child can specify a restart strategy in case of failure:

- temporary: don’t restart me
- transient: only restart me on unexpected failures
- restart: always restart me, even if I quit successfully

We want to implement this retry strategy to mimic the OTP framework. Thankfully, `tenacity` provides the `retry` argument to the decorator, which accepts a callable that returns `True` if the function should be retried.

Let’s implement this:

```python
from enum import Enum, auto
from collections import deque
import tenacity
import trio


class restart_strategy(Enum):
    PERMANENT = auto()
    TRANSIENT = auto()
    TEMPORARY = auto()


class retry_strategy:
    def __init__(
        self,
        restart: restart_strategy
        max_restarts: int,
        max_seconds: float,
    ):
        self.restart = restart
        self.max_restarts = max_restarts
        self.max_seconds = max_seconds

        self.failure_times = deque()

    def __call__(self, retry_state: tenacity.RetryCallState):
        match self.restart:
            case restart_strategy.PERMANENT:
                pass

            case restart_strategy.TRANSIENT:
                if not retry_state.outcome.failed:
                    return False  # don't restart on success

            case restart_strategy.TEMPORARY:
                return False  # don't restart temporary tasks

        now = trio.current_time()
        self.failure_times.append(now)

        if len(self.failure_times) <= self.max_restarts:
            return True  # we didn't hit the max_restart threshold, restart

        oldest_failure = self.failure_times.popleft()
        # restart only if the amount of failures fit in this timespan
        return now - oldest_failure >= self.max_seconds
```

We can now decorate our tasks to restart them as needed:

```python
@retry(
    reraise=True,
    # restart the task if it fails more than 3 times in 5 seconds
    retry=retry_strategy(restart_strategy.TRANSIENT, 3, 5),
    sleep=trio.sleep,
)
async def task():
    # work
```

From this, we can implement Erlang/Elixir’s supervisor pattern:

```python
# supervisor.py
from typing import Any, Callable, Awaitable

from dataclasses import dataclass
from collections import deque
from enum import Enum, auto

import tenacity
import trio


# restart_strategy enum...
# retry_strategy class...


@dataclass
class child_spec:
    task: Callable[..., Awaitable[None]]
    args: list[Any]
    restart: restart_strategy = restart_strategy.PERMANENT


@dataclass
class options:
    max_restarts: int = 3
    max_seconds: float = 5


async def start(
    child_specs: list[child_spec],
    opts: options,
    task_status=trio.TASK_STATUS_IGNORED,
):
    async with trio.open_nursery() as nursery:
        for spec in child_specs:
            nursery.start_soon(_child_monitor, spec, opts)

        task_status.started()


async def _child_monitor(child_specs: list[child_spec], opts: options):
    @tenacity.retry(
        reraise=True,
        retry=retry_strategy(spec.restart, opts.max_restarts, opts.max_seconds),
        sleep=trio.sleep,
    )
    async def _child_runner():
        await spec.task(*spec.args)

    await _child_runner()
```

We use a nursery so that when a task finally crashes after all retry attempts, it will cancel the other children of the supervisor, and crash the supervisor as well. Then, it’s up to you to build your supervision tree accordingly, just like you would do in Erlang/Elixir:

```python
from . import supervisor


async def foo(arg1, arg2):
    do_some_work(arg1, arg2)


async def example():
    children = [
        supervisor.child_spec(
            task=foo,
            args=["hello", "bob"],
            restart=supervisor.restart_strategy.PERMANENT,
        ),
        supervisor.child_spec(
            task=foo,
            args=["hello", "alice"],
            restart=supervisor.restart_strategy.PERMANENT,
        ),
    ]
    opts = supervisor.options(
        max_restarts=3,
        max_seconds=5,
    )

    await supervisor.start(children, opts)
```

Now, spot the differences with this Elixir code:

```
defmodule MyProject.Application do
  use Application

  def start(_type, _args) do
    children = [
      %{
        id: :foo_a,
        start: {MyProject.Foo, :start_link, [[:hello, :bob]]},
        restart: :permanent,
      },
      %{
        id: :foo_b,
        start: {MyProject.Foo, :start_link, [[:hello, :alice]]},
        restart: :permanent,
      },
    ]

    opts = [
      name: MyProject.Supervisor,
      strategy: :one_for_one,
      max_restarts: 3,
      max_seconds: 5,
    ]

    Supervisor.start_link(children, opts)
```

Pretty neat, right?

---

## Mimic OTP even more with TriOTP

A few years ago, I decided to implement a few patterns from OTP in an extension of `trio` :

{{< cta-link
  url="https://github.com/linkdd/triotp"
  title="GitHub - linkdd/triotp: TriOTP, the OTP framework for Python Trio"
  description="TriOTP, the OTP framework for Python Trio. Contribute to linkdd/triotp development by creating an account on GitHub."
>}}

It provides a process registry, which is useful for message-passing, since you can refer to it to find other processes. The `mailbox` offers an abstraction that lets you send and receive messages from it. The “mailbox” has a unique ID and can be associated to one (or more) name, just like Erlang/Elixir.

It also provides an `application` abstraction to mimic the OTP application behavior, a `gen_server` implementation, a `supervisor` as seen above and a `dynamic_supervisor`. In short: the basic building blocks for you to design a complex system with decoupled components and fault tolerance at every level.

**NB:** I must mention, I did not dare try this project in production code, only for experiment, and you probably shouldn’t as well.

The main reason is that my process registry implementation is very weak, compared to what Erlang/Elixir provides with OTP. My implementation relies on Python’s `dict`. And you have to explicitly create the “mailbox” of your “processes” (though, the `gen_server` module does it for you, for this specific pattern).

---

## Conclusion

Erlang and Elixir truly changed the way I design softwares, for the better I hope. OTP is truly a wonderful toolbox to have, and every language with concurrency and channels might allow you to reuse the same patterns. This includes:

- Python with `trio` (and `triotp` 😉)
- Rust with `tokio`
- Go
- Letlang (once I finish the compiler 😉)
- and probably many more…

I hope you enjoyed this article, feel free to clap 👏 for it to give me more visibility.
