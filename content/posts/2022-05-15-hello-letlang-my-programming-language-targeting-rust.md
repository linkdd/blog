---
title: "Hello Letlang! My programming language targeting Rust"
date: "2022-05-15T00:14:36Z"
description: "A new programming language"
tags: ["letlang", "plt", "rust", "python", "open-source"]
author: "David Delassus"
images: ["img/image-9a33674f00fc6404.png"]
---

![](img/image-9a33674f00fc6404.png)

---

## A new programming language

It started with a question:

> What my ideal programming language would look like?

It continued with months of sketching up ideas on paper to answer that question. Then, another few months of designing the language’s syntax and features. Then, another few months of researching how to implement it.

Now, we’re almost 2 years later, and the project is still in its very early stages, **but the “hello world” program compiles! 🎉**

Let’s dive in a bit…

## The inspiration 💭

**Letlang** is inspired by mathematics and the following languages:

- **Elixir**, for its pattern matching and immutability
- **Python**, for its simplicity
- **Rust**, for its type system
- **Go**, for its goroutines and channels
- **TypeScript**, for its type system

## Some key features ✨

It’s key feature is the type system. Objects do not have a **single** type, instead we have **collections** of objects (called **classes**) that determines which objects are included in said collection:

```
class even(n: int) {
  n % 2 = 0;
}
```

```
class odd(n: int & !even);
```

Which lets us check that:

```
42 is int = true;
43 is int = true;
42 is even = true;
42 is odd = false;
43 is even = false;
43 is odd = true;
```

It is also possible to add arbitrary checks using `let` expressions:

```
let n: number;
let n < 0;
n := 42;  # fails because 42 < 0 is false
```

I won’t give too much in details here, as it is already explained on the language’s homepage: <https://letlang.dev>.

Another key feature is that [functions do not have colors](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). Just like Go, any function can be executed in parallel with **coroutines**:

```
func expensive() -> @ok {
  # ...
  @ok;
}
```

```
func main() -> @ok {
  c := coro expensive();
  # ...
  let @ok = join c;
}
```

And just like Go, we can communicate with coroutines, **streams**:

```
let s: stream<int>;
let v: int;
```

```
# write a value to the stream
s |<< 42;
```

```
# block until a value is read from the stream, to a variable:
s |>> v;
```

```
let v = 42;
```

Yet another key features are side-effects and exceptions. The Letlang runtime, which is implemented in Rust, will provide some builtin side effects to interact with the operating system in a safe manner. The developer can then trigger such side effects:

```
func main() -> @ok {
  let @ok = perform debug("hello world");
  @ok;
}
```

Or they can choose to intercept them:

```
func main() -> @ok {
  let @intercepted = do {
    perform debug("hello world");
  }
  intercept debug(_message) {
    @intercepted;
  };
  @ok;
}
```

Or they can define their own side-effects:

```
class log_level(l: "debug" | "info" | "warn" | "error");
effect log(level: log_level, message: string) -> @ok;
```

```
func main() -> @ok {
  let @ok = do {
    let @ok = perform log("debug", "hello");
    let @ok = perform log("info", "world");
  }
  intercept log("debug", message) {
    perform debug(message);
  }
  intercept log(_level, _message) {
    @ok;
  };
  @ok;
}
```

Oh yes, there is **pattern matching** just like Elixir 🙂

Exceptions are just like side-effects, except that they don’t resume the execution:

```
let @caught = do {
  throw (@error, @caught);
  @unreachable;
}
catch (@error, reason) {
  reason;
};
```

Under the hood, the runtime uses the Rust crate [genawaiter](https://docs.rs/genawaiter/latest/genawaiter/) to make every code block a coroutine that can be interrupted by yielding the side effect, or the exception. See my [previous article](/posts/2022-05-11-do-not-wait-for-rust-generators) on this subject.

## The technical stack 🏗️

The parser is written in Rust with the following crates:

- [Logos](https://docs.rs/logos/latest/logos/): to transform the source code (character stream) into a token stream
- [LALRPOP](http://lalrpop.github.io/lalrpop/): to transform the token stream into an Abstract Syntax Tree

Those crate are just amazing, I highly recommend you to use them even for small purposes.

Finally, the parser is given Python bindings thanks to PyO3:

{{< cta-link
  url="https://github.com/PyO3/pyo3"
  title="GitHub - PyO3/pyo3: Rust bindings for the Python interpreter"
  description="Rust bindings for Python, including tools for creating native Python extension modules. Running and interacting with…"
>}}

The compiler is written in Python and will translate the AST into Rust source code. Each Letlang module will be its own Rust crate. The whole source tree will generate a Cargo workspace, including all generated crates.

Each generated crate have a dependency to the Letlang Runtime, which is written in Rust.

Finally, all of it is brought together in a single executable thanks to [PyInstaller](https://pyinstaller.org/en/stable/).

## What can you do with it?

Right now, you can compile the following program:

```
module "hello.main";
```

```
func main() -> @ok {
  perform debug("hello world");
  @ok;
}
```

Which will produce the following output:

```
('hello world')
```

As I said, the project is still in its very early stage. I’m still working on it to implement all the other features, but it is a side-project, so I do not have much time to allocate.

You can find the source code on Github, released under the terms of the **MIT** license:

{{< cta-link
  url="https://github.com/linkdd/letlang"
  title="GitHub - linkdd/letlang: Functional language with a powerful type system."
  description="If you want to contribute, please make a pull request."
>}}

I hope this picked your interest enough to look forward to its first stable release 🙂
