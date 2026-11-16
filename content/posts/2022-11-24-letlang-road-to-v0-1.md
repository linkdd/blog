---
title: "Letlang — Road to v0.1"
date: "2022-11-24T06:40:35Z"
description: "A lot of work has been done on Letlang in the past few months, and many things changed. Let’s discover what!"
tags: ["letlang", "plt", "rust", "compilers", "programming"]
author: "David Delassus"
images: ["img/image-79f4bd1a9992aaee.png"]
---

![](img/image-79f4bd1a9992aaee.png)

---

Recently, I introduced you to [**Letlang**, my programming language targetting Rust](/posts/2022-05-15-hello-letlang-my-programming-language-targeting-rust). I was happy to present a proof of concept which was able to compile a few programs and its website: <https://letlang.dev>

Since then, a lot happened…

{{< cta-link
  url="https://github.com/linkdd/letlang/pull/1"
  title="💥♻️ Only Rust shall remain, all Snakes shall fade by linkdd · Pull Request #1 ·…"
  description="🔥 Scope reduction Many features proposed were quite complicated (if not impossible) to get right in a coherent way…"
>}}

---

## 💥Reducing the scope

Letlang was an ambitious project, too ambitious maybe. Many planned features were either too complex or downright impossible. This is why I decided to get rid of them. A less ambitious project is surely more likely to be released.

Among the deleted features:

**🔥 No more sets, no more infinite sets**

The usefulness of infinite sets is debatable. Regardless of that aspect, the design of the type system and the presence of sets were introducing the Russel paradox:

> Let S be the set of all sets not containing itself. Does S contains S?

> If it does not, it should. If it does, it should not.

**🔥 No more** `solvable{}` **block**

The `solvable{}` block was introducing a proof assistant right in the language, it was to be used for class definitions.

How big the runtime would be? What about the performance? Is it really useful when other really good proof assistants like [coq](https://coq.inria.fr/) exists?

Those questions ended up convincing me that, no I should not include this in the language.

**🔥 No more** `==>` **and** `<==>` **operators**

This one is a direct consequence of the previous point. Without a proof assistant, those operators do not make sense anymore (`<==>` becomes just an equality operator, duplicating `=`).

Without quantified variables (`thereis x` or `forall y`), those logical connectors proves nothing.

**♻️ Coroutines are replaced by Erlang-like processes**

Initially, Letlang’s coroutines were inspired by Go’s **goroutines**. The problem was the coherence of this feature with the side-effects/exceptions feature.

When a non-intercepted side-effect, or an uncaught exception happens in a coroutine, should it bubbles up to the caller or the runtime? What if the caller is doing something else, or has already returned?

Should the exception be raised when joining the coroutine? What about the side-effects?

The semantics around this feature were unclear. Instead, designing an actor model like [Erlang](https://erlang.org/)/[Elixir](https://elixir-lang.org/) made things way simpler and more predictable.

In a “process”, non-intercepted side effects and uncaught exceptions bubbles up to the runtime.

You don’t “join” a process, instead you **link** the target process to the current process to receive a signal when the target process terminates.

This way, the developer can control precisely when and how it receives notifications from other processes.

**♻️ Streams are replaced by Erlang-like message passing**

As a consequence of getting rid of coroutines in favor of Erlang-like processes, streams (which were inspired by Go’s **channels**) became useless.

In an Actor model, the actors (here, the “processes”) exchange messages.

This feature is what made Erlang and Elixir (and any BEAM language) great.

**🔥 No more static type checking**

This one was big. The type system of Letlang is inherently dynamic. Trying to validate it at compile time was heading me straight to the halting problem.

I admit it, I am not smart enough yet to implement a static type checker. Maybe in the future, smarter and more experienced people will be happy to bring this feature back.

In the meantime, Letlang will perform type-checking at runtime. You can see Letlang as some kind of Python where type annotations are mandatory and checked all the time.

One consequence of this has been the redesign of `let` expressions, which are now used to add constraints to (un)bound variables:

```
let a: number;
let b: number { b > a };
```

```
a := 1;
b := 1;  # ERROR: 1 > 1 is false
b := 2;
```

```
let b: number { b > 4 };  # ERROR: 2 > 4 is false
```

**🔥 No more type constraints in generics**

Generic type constraints are usually used to limit what kind of type one can give a generic class or function.

Once again, the dynamic nature of the type system makes this feature very hard to implement.

On the other hand, such constraints can be enforced with type composition using the `&`, `|` and `!` type operators on the desired class/function. Taking advantage of the dynamic type system seems like a good idea 🙂

---

## ♻️Moving from a LR(1) parser to a PEG parser

This one took me quite a few weeks.

LR stands for **“Left to Right, Rightmost derivation”**.

LR(k) parsers require that the grammar contains no ambiguity. This means being able to match one and only one grammar rule to a stream of tokens, only by looking at **k** tokens ahead.

**P**arsing **E**xpression **G**rammars on the other hand don’t mind if multiple rules can match. They will try every rule, matching the first one that succeeds, or falling back on the next one.

For example, let’s consider this grammar:

```
<ambiguity> := (<rule-1> | <rule-2>);
<rule-1> := "a";
<rule-2> := "aa";
```

And the following input:

```
aaa
```

It could parse the input as:

- `<rule-1> <rule-1> <rule-1>`
- `<rule-1> <rule-2>`
- `<rule-2> <rule-1>`

An LR(1) parser will fail to identify a single rule, while a PEG parser will simply return the first.

Restricting myself to an LR(1) parser forced me to introduce awkward tokens to disambiguate the rules. I’d rather keep the syntax easier to understand.

For the curious ones, here are the Rust crates I used to generate the parser:

{{< cta-link
  url="https://github.com/lalrpop/lalrpop/"
  title="GitHub - lalrpop/lalrpop: LR(1) parser generator for Rust"
  description="LALRPOP is a Rust parser generator framework with usability as its primary goal. You should be able to write compact…"
>}}

{{< cta-link
  url="https://github.com/kevinmehall/rust-peg/"
  title="GitHub - kevinmehall/rust-peg: Parsing Expression Grammar (PEG) parser generator for Rust"
  description="Documentation | Release Notes rust-peg is a simple yet flexible parser generator that makes it easy to write robust…"
>}}

---

## ♻️Rewrite the compiler in Rust

The first proof of concept I presented was written partially in Python.

It was nice to experiment a bit, but the end goal has always been to write the compiler entirely in Rust. The snake must die.

The Python code already grew to be messy (it was just an experiment after all) and was hard to reason about.

I find Rust code to be cleaner, and the type system is just superior in all regards to Python’s type system (even with mypy enabled).

The Rust ecosystem also provides very useful tools when writing compilers:

{{< cta-link
  url="https://github.com/maciejhirsz/logos"
  title="GitHub - maciejhirsz/logos: Create ridiculously fast Lexers"
  description="Create ridiculously fast Lexers. Logos has two goals: To make it easy to create a Lexer, so you can focus on more…"
>}}

{{< cta-link
  url="https://github.com/robbepop/string-interner"
  title="GitHub - Robbepop/string-interner: A data structure to efficiently intern, cache and restore…"
  description="Continuous Integration Test Coverage Documentation Crates.io A data structure to cache strings efficiently, with…"
>}}

{{< cta-link
  url="https://github.com/djc/askama"
  title="GitHub - djc/askama: Type-safe, compiled Jinja-like templates for Rust"
  description="Askama implements a template rendering engine based on Jinja. It generates Rust code from your templates at compile…"
>}}

---

## 📝 Updating the documentation

The website has not been updated in a long time. Also since the scope reduction, many things said on the homepage were obsolete.

The handbook presenting the features of the language needed some love as it was, up until now, empty.

It is not a tutorial, more like an introduction to the language’s features. But it is not a formal specification either, it will be the purpose of **LEP** (**L**etlang **E**nhancement **P**roposal) documents, which are still a work in progress as well.

---

## ⚙️Implementing an Actor Model in the runtime

The generated Rust code makes heavy use of **async**. It is implemented using [Tokio](https://tokio.rs) crate.

The first component is the `Node`, started at the beginning of the program. It provides a messaging queue (using `tokio::sync::mpsc`) processes can use to send requests. The node will immediately start a consumer in charge of handling the following requests:

- spawning a new process
- linking two processes
- unlinking two processes
- sending a message signal from one process to another
- sending an exit signal from one process to every linked processes

Those requests will be exposed to the developer via side effects.

The second component is the process. Each process have:

- a unique identifier (the PID)
- a mailbox (implemented with `tokio::sync::mpsc`)

Under the hood, it is implemented as:

[Basic structure of a Letlang process implementation](https://gist.github.com/linkdd/79adde47c40d7dbdb0c212b412389d91)

```rust
tokio::spawn(async move {
  let process_handle = tokio::spawn(async move {
    // run process's function
  });

  let res = process_handle.await;
  // notify the node of the process termination
});
```

Using two nested Tokio tasks allows catching panics and notifying other processes of the crash. This happens when an unknown side effects, or an uncaught exception bubbles up.

The third main aspect of the runtime is that each block of code is a “generator” (I have written about it in a [previous article](/posts/2022-05-11-do-not-wait-for-rust-generators)), whenever a side effect or an exception is triggered, this yields the generator’s coroutine:

[Letlang basic structure of a block of code](https://gist.github.com/linkdd/7233c319fe3f2048b3b6b472459fcf0d)

```rust
let task_args: Vec<Value> = vec![];
let mut block = func.call(&mut context, task_args);

let ignored = Value::Boolean(bool::default());
let mut state = block.resume_with(ignored);

loop {
  match &state {
    GeneratorState::Yielded(FunctionInterruption::Effect { name, args }) => {
      // try to handle side effect

      if unknown_side_effect {
        // print debug information
        return Err(RuntimeError::EffectNotImplemented);
      }
      else {
        let effect_return_value = { /* ... */ };
        state = block.resume_with(effect_return_value);
      }
    },
    GeneratorState::Yielded(FunctionInterruption::Exception(exc)) => {
      // print exception
      return Err(RuntimeError::UncaughtException);
    },
    GeneratorState::Complete(val) => {
      // type check function's return value
      break;
    }
  }
}
```

All of this is still a work in progress, and a redesign of the implementation might happen in the future.

---

## ➿Dealing with Tail Recursion

Up until now, there were no tail recursion optimization. Which is quite problematic in a functional language, because that’s how you implement loops!

For this feature, I introduced a new **explicit** syntax do differentiate normal functions from tail-recursive functions:

```
func notail_len<T>(items: list<T>, acc: int) -> int {
  match items {
    [] => acc,
    [_ | tail] => notail_len(tail, acc + 1),
  };
}

tailrec tail_len<T>(items: list<T>, acc: int) -> int {
  match items {
    [] => final[acc],
    [_ | tail] => recurse[tail, acc + 1],
  };
}
```

We can see 2 differences with the `notail_len` function and the `tail_len` function.

First, the keyword used to define a tail-recursive function is not `func` but `tailrec`. Then the return value of the tail-recursive function is one of:

- `final[ret_type]`
- `recurse[args_types...]`

In the runtime, a tail-recursive function will be implemented as a Rust `loop{}` , whenever the function returns `final[ret_type]` we break out of the loop, or we mutate the function parameters:

```rust
let result = loop {
  let return_val = {
    // function's body
  };

  match return_val {
    Value::TailRecFinal(value) => {
      break value;
    },
    Value::TailRecRecurse(args) => {
      // mutate function parameters
    },
    val => {
      // throw letlang exception
      unreachable()!;
    }
  }
}  
```

---

## 🪢Constraints and pattern matching

In the previous Python version, pattern matching (similar to Erlang/Elixir) was not present. This is now done!

The `:=` operator is a pattern matching operator. It tries to match the value on the right to a pattern on the left, and if the pattern does not match, an exception is thrown:

```
42 := 42;
(@ok, 42) := (@ok, 42);
(@ok, val) := (@ok, 42);
42 := val;
43 := val; ## error!
```

When a variable name appears in a pattern, the runtime will bind the matched value to it. While you can rebind a variable to a new value, the previous value is not mutated.

A variable can be bound to any value, this is where the constraints system comes into the picture. Using `let` expressions, you can constrain a variable to a type and one or more predicates:

```
let a: number;
let b: number { b > a };

a := 0;   ## OK
b := 1;   ## OK
b := -1;  ## ERROR
```

`let` expressions can also be used as assertions:

```
x := -1;
let x: number { x > 0 };  ## ERROR
```

This new syntax clears any confusion you could have between `x := y` and `let x = y` in the previous version.

---

## 🛣️Roadmap, what’s next?

While the pull request has been merged, a lot of work remains before a v0.1.0 release:

- implement `do{}` blocks with exception catching, and side effect interception
- implement process spawning and linking (runtime done, code generation to do)
- implement message passing (runtime done, code generation to do)
- implement a first draft of the standard library
- write more documentation!!

In the long term, I’m planning to implement a very simple HTTP server with that language, to benchmark it, test its limits and see how is the developer experience 🙂

Thank you for reading through all of this, here is a picture of my favorite cake:

![](img/image-4597c822f6a6cc4f.jpg)
