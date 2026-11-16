---
title: "Do not wait for Rust generators"
date: "2022-05-11T09:37:59Z"
description: "What is generator?"
tags: ["rust", "programming"]
author: "David Delassus"
images: ["img/image-b35ea681816b8383.png"]
---

![](img/image-b35ea681816b8383.png)

---

## What is generator?

Generators are a pattern widely used in the Python ecosystem, like iterators, they are very useful to produce multiple values. But unlike iterators, they work by **interrupting** the execution of the function, and allows the caller to **resume** its execution later:

[Python Generators](https://gist.github.com/linkdd/439e23a6b6d90e3b305c20f1516b55a7)

```python
def my_generator():
    yield 1
    yield 2
    yield 3
    return 4

gen = my_generator()
next(gen)  # 1
next(gen)  # 2
next(gen)  # 3
next(gen)  # raise StopIteration(4)
```

The `yield` keyword interrupts the function and produce the supplied value. Now, instead of using the `next()` function to resume the generator, you can use:

- `gen.send(value)`: the `yield` keyword will return `value` when resuming
- `gen.throw(exc)`: the `yield` keyword will throw `exc` when resuming
- `gen.close()`: the generator won’t resume (and raise a `StopIteration`)

> **NB:** This is also called a **coroutine**.

They are a powerful tool. In fact, they were used for concurrency before the rise of `async`/`await`!

Unfortunately, at the time of writing, this is still an [unstable feature](https://github.com/rust-lang/rust/issues/43122) in Rust.

## Introducing… genawaiter!

{{< cta-link
  url="https://github.com/whatisaphone/genawaiter"
  title="GitHub - whatisaphone/genawaiter: Stackless generators on stable Rust."
  description="This crate implements stackless generators (aka coroutines) in stable Rust. Instead of using yield, which won't be…"
>}}

This crate gives you, thanks to some useful macros, the **unstable** feature in **stable** Rust:

[Rust generators (genawaiter)](https://gist.github.com/linkdd/1c72baaa8f0c01c0a8b11c1982a60c2f)

```rust
use genawaiter::{sync::gen, yield_, GeneratorState};

let gen = gen!({
  yield_!(1);
  yield_!(2);
  yield_!(3);
  4
});

assert_eq!(gen.resume(), GeneratorState::Yielded(1));
assert_eq!(gen.resume(), GeneratorState::Yielded(2));
assert_eq!(gen.resume(), GeneratorState::Yielded(3));
assert_eq!(gen.resume(), GeneratorState::Complete(4));
```

Just like Python, the `yield_!` macro will interrupt the execution of the code block, the `gen.resume()` function will return a `GeneratorState` enum with the yielded value or the final return value.

If called with `gen.resume_with(arg)`, the `yield_!` macro will return `arg`.

## One last trick…

Every generators implement the trait `Generator<Yield = Y, Return = R>`. If, like me, you have some Python background, you’d like to return such generators, possibly make a trait with a function to return such generators, the problem here is:

- you can’t use `impl Trait` in type-aliases or in trait function signatures (unstable feature)
- you can’t return a local variable as a `&dyn Trait` (the variable goes out of scope, this will never be a feature thankfully)

At the moment, one solution is to move the generator into a `Box<dyn Trait>`:

[Return Rust generator from function](https://gist.github.com/linkdd/48c3be67a1e283d1a7909e110e0a3771)

```rust
use genawaiter::{sync::gen, yield_, Generator};

type MyGen = Box<dyn Generator<Yield = i32, Return = i32>>;

fn make_generator() -> MyGen {
  let gen = gen!({
    yield_!(1);
    yield_!(2);
    yield_!(3);
    4
  });
  Box::new(gen)
}
```

## Conclusion

You don’t need to wait to use this feature, **genawaiter** aims to be a “drop-in” replacement of the currently unstable feature. Once it’s generally available, it should be a matter of renaming imports.
