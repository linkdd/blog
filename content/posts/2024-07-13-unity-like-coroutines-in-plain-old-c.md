---
title: "Unity-like coroutines in plain old C"
date: "2024-07-13T17:08:23Z"
description: "Coroutines are a great tool in game development, be it for polishing, or implementing complex behavior over time. Learn how to use them in…"
tags: ["c", "gamedev", "programming", "design-patterns"]
author: "David Delassus"
images: ["img/image-06ca3d21ea97845d.png"]
---

![](img/image-06ca3d21ea97845d.png)

---

Let’s just begin with this: Coroutines are great!

For those who don’t know what coroutines are, let me explain:

Coroutines allow a function to be paused and resumed. This is often used to allow the concurrent execution of multiple functions (and is often what is used to implement `async` / `await` in most programming languages), but in Game Development, they are also used to spread a long running function over multiple frames.

![](img/image-19d6a703cf394fe5.gif)

In the above screencast, clicking on the button “DAMAGE” sets the red bar (representing health) to a specific value, and triggers a coroutine that changes the value of the orange bar progressively over time.

In this article, we’ll see how to implement this in C.

---

## In-depth explanation of coroutines

{{< cta-link
  url="https://en.wikipedia.org/wiki/Coroutine"
  title="Coroutine - Wikipedia"
  description="Coroutines are computer program components that allow execution to be suspended and resumed, generalizing subroutines…"
>}}

At its core, a coroutine can be modeled as a state machine:

```c
typedef enum {
    CORO_SUSPENDED,
    CORO_COMPLETED
} coro_state;

coro_state my_coroutine(void) {
    static int state = 0;

    switch (state) {
        case 0:
            printf("Step 1\n");
            state = 1;
            return CORO_SUSPENDED;

        case 1:
            printf("Step 2\n");
            state = 2;
            return CORO_SUSPENDED;

        case 2:
        default:
            return CORO_COMPLETED;
    }
}

int main(void) {
    while (my_coroutine() != CORO_COMPLETED) {}

    return 0;
}
```

There are 2 kinds of coroutines:

- **stackful:** each coroutine has its own call stack independent from the main thread
- **stackless:** each coroutine shares its call stack with the main thread

**NB:** The above example is a **stackless** coroutine.

The main advantage of stackful coroutines is that they can easily handle more complex control flows like recursion. However, they do use more memory and can have an impact on performance. They can be implemented using `setjmp` / `longjmp`. Therefore, “yielding”/pausing a coroutine can be done wherever in the call stack:

```c
void f(void) {
    coroutine_yield();
    printf("Step 3\n"); // will resume here
}

void my_coroutine() {
    printf("Step 1\n");
    coroutine_yield();

    printf("Step 2\n");
    f(); // will yield
}
```

A stackless coroutine however, is more lightweight and a great option when performance is required. They are well fit for simple tasks, but cannot handle the complex control flows that stackful coroutines allow. If the above example were to be recursive, the explicit handling of the coroutine’s state can become hard to follow quite quickly.

---

### Coroutines in other languages

C\+\+20 introduced stackless coroutines:

{{< cta-link
  url="https://en.cppreference.com/w/cpp/language/coroutines"
  title="Coroutines (C++20)"
  description="A coroutine is a function that can suspend execution to be resumed later. Coroutines are stackless: they suspend…"
>}}

In Unity, coroutines are also stackless, and even transformed into state machines. They are resumed each frame:

{{< cta-link
  url="https://docs.unity3d.com/Manual/Coroutines.html"
  title="Coroutines"
  description="A coroutine allows you to spread tasks across several frames. In Unity, a coroutine is a method that can pause…"
>}}

However, Lua coroutines are stackful:

{{< cta-link
  url="https://www.lua.org/pil/9.1.html"
  title="Programming in Lua : 9.1"
  description="This first edition was written for Lua 5.0. While still largely relevant for later versions, there are some…"
>}}

---

### Asynchronous tasks, generators, and… Unity-like?

You may have seen:

```csharp
async Task DoSomething() {
    await DoSomethingElse();
    await DoSomethingElse();
}

void Example() {
    var task = DoSomething(); // the task in run in a separate thread
    while (!task.IsCompleted) {
        Sleep(1);
    }
}
```

This is a C# example of an asynchronous task. A special kind of coroutine which is to be executed in a separate thread (in parallel). The `await` keyword is responsible for pausing the coroutine.

We also have generators, which can be used to create iterators:

```csharp
IEnumerable<int> Example() {
    for (int i = 1; i <= 10; i++) {
        yield return i;
    }
}
```

The `yield return` statement is responsible for pausing the coroutine, it is also used to send back data to the caller. This is especially useful to implement infinite lists for example:

```csharp
IEnumerable<int> Example() {
    while (true) {
        yield return RandomInteger();
    }
}
```

Now, what do I mean by **Unity-like**? What is so special about Unity coroutines? Aren’t they just C# generators/iterators?

Well, yes they are. But the exposed API allow to yield another coroutine, which will pause the current coroutine until the yielded one completed!

```csharp
IEnumerable<int> Do123() {
    yield return 1;
    yield return 2;
    yield return 3;
}

IEnumerable<int> Do456() {
    yield return 4;
    yield return 5;
    yield return 6;
}

IEnumerable<int> Do123456() {
    yield return Do123();
    yield return Do456();
}
```

This API is incredibly useful in Game Development, this allows you to reuse simple coroutines to build more complex behaviors:

```csharp
IEnumerable Wait(float seconds) {
    while (seconds > 0f) {
        seconds -= Time.deltaTime;
        yield return null;
    }
}

IEnumerable MoveTowards(Vector3 target) {
    while(DistanceBetween(transform.position, target) > someThreshold) {
        yield return null;
    }
}

IEnumerable WaitAndMove() {
    yield return Wait(5);
    yield return MoveTowards(someTarget);
}
```

---

### For the C\+\+ developers

I already implemented such API in my C\+\+ game framework using C\+\+20 coroutines:

{{< cta-link
  url="https://github.com/trollworks/sdk-core#coroutines"
  title="GitHub - trollworks/sdk-core: Core SDK of Trollworks engine"
  description="Core SDK of Trollworks engine. Contribute to trollworks/sdk-core development by creating an account on GitHub."
>}}

So if you are not interested by the C implementation, you can skip the rest of the article.

---

## Coroutines in C, where to start?

{{< cta-link
  url="https://github.com/edubart/minicoro"
  title="GitHub - edubart/minicoro: Single header stackful cross-platform coroutine library in pure C."
  description="Single header stackful cross-platform coroutine library in pure C. - edubart/minicoro"
>}}

**minicoro** is a single-file library inspired by Lua coroutines, it is mainly used as a backend for the [Nelua](https://github.com/edubart/nelua-lang) programming language. As such, they are **stackful**.

Not only it is easy to integrate in your own project, it also supports custom allocators, which is great!

This is what we will use to build a Unity-like API on top.

---

### A small detour: a custom allocator

One thing I consider best practice when writing a C library, is to define an allocator interface, so that the user of your library can plug in his own allocator.

I take a lot of inspiration from this article, which I highly recommend you read before continuing:

{{< cta-link
  url="https://nullprogram.com/blog/2023/12/17/"
  title="So you want custom allocator support in your C library"
  description="nullprogram.com/blog/2023/12/17/ This article was discussed on Hacker News and on reddit. Users of mature C libraries…"
>}}

As such, here is the interface that I often use:

```c
#include <assert.h>
#include <stdint.h>
#include <string.h>

typedef struct allocator allocator;
struct allocator {
    void *(*alloc)(size_t sz, void *ctx);
    void *(*realloc)(void *ptr, size_t oldsz, size_t newsz, void *ctx);
    void (*dealloc)(void *ptr, size_t sz, void *ctx);
    void *ctx;
};

void *allocator_alloc(allocator *self, size_t sz) {
    assert(self != NULL);
    assert(self->alloc != NULL);

    return self->alloc(sz, self->ctx);
}

void *allocator_calloc(allocator *self, size_t nmemb, size_t membsz) {
    if (nmemb > 0 && membsz > SIZE_MAX / nmemb) {
        // integer overflow, abort
        return NULL;
    }

    void *ptr = allocator_alloc(self, nmemb * membsz);
    if (ptr != NULL) {
        memset(ptr, 0, nmemb * membsz);
    }
    return ptr;
}

void *allocator_realloc(allocator *self, void *ptr, size_t oldsz, size_t newsz) {
    assert(self != NULL);
    assert(self->realloc != NULL);

    return self->realloc(ptr, oldsz, newsz, self->ctx);
}


void allocator_dealloc(allocator *self, void *ptr, size_t sz) {
    assert(self != NULL);
    assert(self->dealloc != NULL);

    return self->dealloc(ptr, sz, self->ctx);
}
```

If you wish to use the `stdlib` allocation functions, this can then easily be done with the following code:

```c
#include <stdlib.h>

static void *_malloc(size_t sz, void *ctx) {
    (void)ctx;

    return malloc(sz);
}

static void *_realloc(void *ptr, size_t oldsz, size_t newsz, void *ctx) {
    (void)ctx;
    (void)oldsz;

    return realloc(ptr, newsz);
}

static void _free(void *ptr, size_t sz, void *ctx) {
    (void)ctx;
    (void)sz;

    free(ptr);
}

void allocator_init_stdlib(allocator *self) {
    assert(self != NULL);

    self->alloc   = _malloc;
    self->realloc = _realloc;
    self->dealloc = _free;
    self->ctx     = NULL;
}
```

Which can then be used:

```c
allocator a = {0};
allocator_init_stdlib(&a);
```

Parenthesis closed, let’s go back to our coroutines.

---

## A coroutine manager

What we actually want is the ability to create coroutines and schedule them for execution each frame of our game loop (until they are done).

To do this, we need a data structure to keep track of the scheduled coroutines. Because our coroutines can be nested (by yielding a new coroutine), this data structure is a tree.

Let’s define a few types:

```c
typedef struct coroutine_manager coroutine_manager;
typedef struct coroutine coroutine;
typedef struct coroutine_udata coroutine_udata;
typedef void (*coroutine_fn)(coroutine *coro);
```

The first one represent our manager that will keep track of our scheduled coroutines.  
The second one represent a wrapper to the **minicoro** coroutine structure, it will contain:

- the underlying minicoro coroutine
- the user data passed to it
- a reference to the coroutine manager, and the allocator
- links to the siblings and parent coroutines in the tree

Here is the definition:

```c
/// Structure passed as userdata to the underlying coroutine
struct coroutine_udata {
    coroutine *self;
    coroutine_fn fn;
};

struct coroutine {
    /// copy of the initial data passed as argument to the coroutine
    void *arg;
    size_t argsz;

    allocator *allocator;
    coroutine_manager *mgr;

    coroutine_udata udata;
    mco_desc desc; /// The underlying coroutine descriptor
    mco_coro *co;  /// The actual underlying coroutine

    // tree structure
    coroutine *parent;
    struct {
        coroutine *prev;
        coroutine *next;
    } siblings;
};

struct coroutine_manager {
    allocator *allocator;

    struct {
        coroutine *head;
        coroutine *tail;
    } coroutines;
};
```

The API of a `coroutine` structure is quite simple: a constructor, a destructor, and a wrapper to the **minicoro** `yield` function.

The constructor takes as input our coroutine implementation, and the address of initial data to pass as argument to the coroutine function. It must allocate the `coroutine` structure, and the memory to store the initial data. Then it must initialize the underlying coroutine with our wrapper implementation:

```c
// wrapper coroutine implementation
static void _coro_task(mco_coro *co) {
    coroutine_udata *udata = mco_get_user_data(co);
    udata->fn(udata->self); // calls the coroutine_fn function
}

coroutine *coroutine_new(
    coroutine_manager *mgr,
    coroutine_fn fn,
    void *arg,
    size_t argsz
) {
    assert(self != NULL);
    assert(fn != NULL);

    coroutine *coro = allocator_alloc(mgr->allocator, sizeof(coroutine));
    if (coro == NULL) {
        // allocation failed
        return NULL;
    }

    if (arg != NULL) {
        assert(argsz > 0);

        coro->arg = allocator_alloc(mgr->allocator, argsz);
        if (coro->arg == NULL) {
            // allocation failed
            allocator_dealloc(mgr->allocator, coro, sizeof(coroutine));
            return NULL;
        }

        memcpy(coro->arg, arg, argsz);
        coro->argsz = argsz;
    }
    else {
        coro->arg   = NULL;
        coro->argsz = 0;
    }

    coro->allocator = mgr->allocator;
    coro->mgr       = mgr;

    coro->udata.self = coro;
    coro->udata.fn   = fn;

    coro->desc           = mco_desc_init(_coro_task, 0);
    coro->desc.user_data = &coro->udata;

    coro->parent        = NULL;
    coro->siblings.next = NULL;
    coro->siblings.prev = NULL;

    mco_result res = mco_create(&coro->co, &coro->desc);
    if (res != MCO_SUCCESS) {
        // coroutine creation failed

        if (coro->arg != NULL) {
            allocator_dealloc(mgr->allocator, coro->arg, coro->argsz);
        }

        allocator_dealloc(mgr->allocator, coro, sizeof(coroutine));
        return NULL;
    }

    return coro;
}
```

**NB:** It’s worth mentioning that minicoro also provides an interface to override the allocation functions. We can plug in our allocator:

```c
static void *_mco_alloc(size_t size, void *udata) {
    assert(udata != NULL);
    return allocator_alloc((allocator *)udata, size);
}

static void *_mco_dealloc(void *ptr, size_t size, void *udata) {
    assert(udata != NULL);
    allocator_dealloc((allocator *)udata, ptr, size);
}

coroutine *coroutine_new(coroutine_manager *mgr,
    coroutine_fn fn,
    void *arg,
    size_t argsz
) {
    // ...

    coro->desc                = mco_desc_init(_coro_task, 0);
    coro->desc.alloc_cb       = _mco_alloc;
    coro->desc.dealloc_cb     = _mco_dealloc;
    coro->desc.allocator_data = mgr->allocator;
    coro->desc.user_data      = &coro->udata;

    // ...
}
```

The desctructor is quite straightforward, simply free the memory:

```c
void coroutine_free(coroutine *self) {
    assert(self != NULL);

    mco_destroy(self->co);

    if (self->arg != NULL) {
        allocator_dealloc(self->allocator, self->arg, self->argsz);
    }

    allocator_dealloc(self->allocator, self, sizeof(coroutine));
}
```

And finally, the wrapper to `yield`. The minicoro library provides a LIFO data structure to exchange data between the coroutine and the manager. We can push data on this structure right before we yield, so that the manager can pop it. This is where we will push the address of the nested coroutine to run:

```c
void coroutine_yield(coroutine *self, coroutine *nested) {
    assert(self != NULL);

    mco_push(self->co, &nested, sizeof(coroutine*));
    mco_yield(self->co);
}
```

Which will be used like this:

```c
void mycoro(coroutine *coro) {
    // C#: yield return null
    coroutine_yield(coro, NULL);

    // C#: yield return OtherCoroutine()
    coroutine *other = coroutine_new(coro->mgr, other_coroutine_fn, NULL, 0);
    // ...error handling...
    coroutine_yield(coro, other);
}
```

The coroutine API being done, we are now missing the coroutine manager scheduling API. Which consist of 2 functions:

- `schedule` : to add a coroutine to the list of scheduled coroutines
- `unschedule` : to remove a coroutine from that list

It’s a classic doubly-linked list implementation, so let’s get to it without commentary:

```rust
void coroutine_manager_schedule(coroutine_manager *self, coroutine *coro) {
    assert(self != NULL);
    assert(coro != NULL);

    if (self->coroutines.head == NULL) {
        self->coroutines.head = coro;
        self->coroutines.tail = coro;
    }
    else {
        self->croutines.tail->siblings.next = coro;
        coro->siblings.prev                 = self->coroutines.tail;
        self->coroutines.tail               = coro;
    }
}

void coroutine_manager_unschedule(coroutine_manager *self, coroutine *coro) {
    assert(self != NULL);
    assert(coro != NULL);

    if (coro->siblings.prev != NULL) {
        coro->siblings.prev->siblings.next = coro->siblings.next;
    }
    else {
        self->coroutines.head = coro->siblings.next;
    }

    if (coro->siblings.next != NULL) {
        coro->siblings.next->siblings.prev = coro->siblings.prev;
    }
    else {
        self->coroutines.tail = coro->siblings.prev;
    }
}
```

The final part is the API to run all the scheduled coroutines until they are done. We do this by iterating through the list in the manager, resuming the coroutine. If the coroutine is still suspended, we pop from its LIFO data structure. If we find a `NULL` pointer, we do nothing, otherwise, it’s a pointer to a new coroutine that must run until completion before resuming the old one. So we will unschedule the old one and schedule the new one. However, if the coroutine is completed (or “dead”), we simply unschedule it and free it. If it had a parent, we reschedule the parent:

```c
void coroutine_manager_run(coroutine_manager *self) {
    assert(self != NULL);

    coroutine *coro = self->coroutines.head;
    while (coro != NULL) {
        // we are going to modify the iterator or even free it
        // so, let's save the reference to the next item right away
        coroutine *next = coro->siblings.next;

        // resume the underlying coroutine and fetch its status
        mco_resume(coro->co);
        mco_state state = mco_status(coro->co);

        if (state == MCO_SUSPENDED) {
            // the coroutine yielded, fetch the value pushed on the LIFO
            coroutine *nested = NULL;
            mco_pop(coro->co, &nested, sizeof(coroutine*));

            // if there is a new coroutine to run, schedule it
            if (nested != NULL) {
                nested->parent = coro; // so that we can reschedule it once completed
                coroutine_manager_unschedule(self, coro);
                coroutine_manager_schedule(self, nested);
            }
        }
        else if (state == MCO_DEAD) {
            // the coroutine completed, let's reschedule its parent
            if (coro->parent != NULL) {
                coroutine_manager_schedule(self, coro->parent);
            }

            // and unschedule this one, and destroy it
            coroutine_manager_unschedule(self, coro);
            coroutine_free(coro);
        }

        coro = next;
    }
}
```

Our coroutine manager is now fully implemented 🙂 Thanks **minicoro** for the heavy lifting.

---

## A simple example

Let’s write a coroutine that yields until some time has passed (using SDL):

```c
#include <SDL.h>

static void _wait_for_fn(coroutine *coro) {
    float seconds = *(float *)coro->arg;

    Uint32 start = SDL_GetTicks();

    while (((float)(SDL_GetTicks() - start) / 1000.0f) < seconds) {
        coroutine_yield(coro, NULL);
    }
}

coroutine *wait_for(coroutine_manager *mgr, float seconds) {
    return coroutine_new(mgr, _wait_for_fn, &seconds, sizeof(float));
}    
```

Then, a coroutine to decrease the orange bar (represented by the `health_dirty` value) until it matches the red bar (represented by the `health_real` value):

```c
static void _update_health_dirty_fn(coroutine *coro) {
    player *p = *((player **)coro->arg);

    while (true) {
        p->health_dirty = lerp(p->health_dirty, p->health, 0.1f);
        if (fabsf(p->health - p->health_dirty) < 0.0001f) {
            p->health_dirty = p->health;
        }

        if (p->health != p->health_dirty) {
            coroutine_yield(coro, NULL);
        }
        else {
            break;
        }
    }
}

coroutine *update_health_dirty(coroutine_manager *mgr, player *p) {
    return coroutine_new(mgr, _update_health_dirty_fn, &p, sizeof(player*));
}
```

And finally, the coroutine that waits a bit before updating the health bar:

```c
static void _update_health_bar_fn(coroutine *coro) {
    player *p = *((player **)coro->arg);

    coroutine_yield(coro, wait_for(coro->mgr, 0.5f /* seconds */));
    coroutine_yield(coro, update_health_dirty(coro->mgr, p));
}

coroutine *update_health_bar(coroutine_manager *mgr, player *p) {
    return coroutine_new(mgr, _update_health_bar_fn, &p, sizeof(player*));
}
```

Now, in our game loop:

```c
while (true) {
    // ...input...

    // ...game logic...
    if (damage_taken) {
        coroutine_manager_schedule(
            &coro_mgr,
            update_health_bar(&coro_mgr, &player_data)
        );
    }

    coroutine_manager_run(&coro_mgr);

    // ...rendering...
}
```

And the final result:

![](img/image-19d6a703cf394fe5.gif)

---

## Conclusion

Coroutines are a great tool in game development, be it for polishing (like in our example), or implementing complex behavior over time.

And since **minicoro** is inspired by Lua coroutines, that means the code above can be ported to Lua quite easily, and therefore you can benefit from it with game frameworks like [LÖVE](https://love2d.org/).

Consider the code above free of use (CC0, public domain, WTFPL, whichever you like), and have fun in your game development journey 🙂

---

Feel free to clap 👏 for this article to give it more visibility.
