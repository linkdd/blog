---
title: "Letlang specification finalized!"
date: "2024-02-04T13:12:50Z"
description: "After a lot of work, a specification describing Letlang’s intended behavior and semantics has finally been completed!"
tags: ["plt", "programming", "rust", "compilers", "letlang"]
author: "David Delassus"
images: ["img/image-dcdcebcd80ca0263.png"]
---

![](img/image-dcdcebcd80ca0263.png)

---

{{< cta-link
  url="https://letlang.dev"
  title="Letlang"
  description="General purpose programming language"
>}}

A long time has passed since the last update on Letlang… This is mainly because I hadn’t had the free time in the past months. But the last few weeks, I took the time to finalize the specification for the v0.1.0 🙂

For this specification, the syntax of the language has changed to better reflect its semantics. The architecture of the compiler has also been redesigned quite a bit.

All those changes were for the sake of simplicity and clarity. With this specification, the language is now more coherent and less like an aggregate of random features.

---

## A new command line interface

One thing I like about C and C\+\+ (and Rust without Cargo) is the simplicity of the CLI, you take one source file and make a build artifact:

```
$ gcc -c foo.c -o foo.o
```

Up until now, Letlang needed a `letproject.toml` file and expected a specific folder tree. It was inspired by the design of Cargo. But Cargo is not a compiler, it is a package manager/build system. It relies on `rustc`, the actual Rust compiler.

One thing that remains the same is that every Letlang module is translated to a Rust crate. There are 2 kinds of crates that Letlang will produce: library crates, and executable crates. The way to build them has changed though:

```
$ letlangc --type=lib --rpath /path/to/runtime-crate foo.let
$ letlangc --type=exe --rpath /path/to/runtime-crate bar.let
```

Library artifacts will produce a `libfoo.lla` archive containing the following:

```
|-- libfoo.lli
|-+ lldep_foo/
  |-- Cargo.toml
  |-+ src/
    |-- lib.rs
```

The `libfoo.lli` file contains what is called the “Binary Module Interface”. It is a description of the exported Letlang symbols that will be used during the name resolution phase of compilation. This allows the developer to not need the actual Letlang code for a module in order to depend on it.

To add a dependency to another module, you will need a combination of the `-L` and `-l` flags. Just like with `gcc`, the `-L` flag adds a directory to the search paths used by Letlang to locate the `.lla` files, and the `-l` flag specify the `.lla` file to add as dependency:

```
$ letlangc --type=lib --rpath /path/to/runtime-crate foo.let
$ letlangc --type=exe --rpath /path/to/runtime-crate bar.let -L. -lfoo
$ ./bar
```

From this simple CLI, a package manager/build system will be easier to implement later on, and it will remain decoupled from the compiler.

---

## About symbol declarations

In Letlang, there has always been 3 kinds of symbols: functions, classes and effect signatures. This fact still is true, but the way to declare them has been redesigned.

**Before:**

```
class even(n: int) {
  n % 2 = 0;
}

effect log(string) -> @ok;

func main() -> @ok {
  @ok;
}
```

**After:**

```
let even: class[int] {
  (n) -> n % 2 = 0,
};

let log: effect[(string) -> @ok];

# the `pub` keyword adds the symbol to the Binary Module Interface
let pub main: func[() -> @ok] {
  () -> @ok,
};
```

The main difference here is where I put the emphasis. I use `let` to declare a **symbol**. To that symbol, I associate on of Letlang’s primitive, a class, an effect or a function.

A symbol can have type parameters, those will be put on the name:

```
let swap<T, U>: func[(T, U) -> (U, T)] {
  (a, b) -> (b, a),
};
```

This means every symbol have a “type arity” in the Binary Module Interface.

---

## Pattern-based dispatch

With the new syntax, pattern matching now takes a more central place in the language. Consider this example:

```
# the `tailrec` keyword is used for tail-recursive functions
let factorial_impl: tailrec[(int, int) -> int] {
  (0, acc) -> final[acc],
  (n, acc) -> recurse[n - 1, n * acc],
};

let pub factorial: func[(int) -> int] {
  (n) -> factorial_impl(n, 1),
};

let pub collatz: func[(int) -> int] {
  (n) when n % 2 = 0 -> n / 2,
  (n) -> 3 * n + 1,
};
```

The body of a function is composed of multiple “clauses” which associate an expression (or a code-block) to a pattern (and an optional guard expression).

This concept is also used for class constraints:

```
let io_error: class[term] {
  ( (@error, (@io, _)) ) -> true,
  ( (@io_error, _)     ) -> true,
};
```

In fact, every branching based on a pattern use exactly the same syntax:

`match` **expressions:**

```
match foo() {
  ( (@ok, val)       ) -> val,
  ( (@error, reason) ) -> throw reason,
};
```

`receive` **expressions:**

```
receive {
  (@message, pid, msg) -> {
    # ...
  },
  (@exited, pid, reason) -> {
    # ...
  },
};
```

`do` **expressions:**

```
do {
  # ...
}
intercept log {
  (msg) -> {
    # ...
  },
}
catch {
  ( (@error, (@io, reason)) ) -> {
    # ...
  },
  ( (@error, (@os, reason)) ) -> {
    # ...
  },
};
```

This actually makes the language more coherent, and predictable. Thus, easier to read.

---

## Add variable constraints with lazy assertions

Lazy assertions were already part of the language. It gave the ability to the developer to add type and expression constraints to a variable that is yet to be defined.

It used to look like this:

```
let x: int;
let y: int with { x > y };

x := 1;
y := 2;  # oops!!
```

But the semantics were complex, and unclear, consider the following:

```
let x: int;

x := 0;

let x: string;
# x is now `int & string` which contains no value
# but x already is bound to the value `0`, so error here!
```

Now you have not-so-lazy assertions, which is kind of weird and unintuitive.

Letlang is an immutable language, this means that you can’t modify data, only bind variables to new values (leaving the previous value untouched). But somehow, the lazy assertion expression is mutating the current scope.

The key to simplify this expression is “mutating the current scope”. What if we don’t do that? Instead, we create a whole new scope, and suddenly the expression becomes:

```
@ok := let (x: int, y: int) with x > y {
  x := 1;
  y := 2;  # oops!
  @ok;
};
```

---

## This is just a draft

This specification will follow [semantic versioning](https://semver.org), and the current version is `0.0.0`. This means it is still experimental, and still subject to changes.

Nothing is set in stone until the `1.0.0` is released.

But now that I have a specification, I will track updates to it through what I call “Letlang Enhancement Proposals”, inspired by Python’s PEPs.

Some that will come in the near future are:

- Rust Foreign Functions: to be able to call Rust code from Letlang
- Lambdas: to enable Higher Order Functions in a satisfying way
- …

---

## The site has finally been updated

All those changes have been finally published, and I am happy to say that the website ( <https://letlang.dev> ) is **up-to-date**.

It relies on [**Gin**](https://linkdd.github.io/gin/), a static website generator I wrote inspired by Hugo, which supports features nice for language designers, like Railroad diagrams, or a syntax highlighting engine that supports custom grammars. I talked about it in a previous article:

{{< cta-link
  url="/posts/2024-01-02-yet-another-static-site-generator-but-for-language-designers"
  title="Yet another static site generator, but for language designers"
  description="You are a language designer, you are working on your own programming language, maybe because you had an innovative…"
>}}

---

## So, what now?

After all those steps forward, I had to take a step back. Writing this specification made me aware of oddities in the language, that were solved by clarifying the syntax and semantics. Thanks to this specification, I even found bugs in my initial (unfinished) implementation.

This is why I’m starting from scratch again for a specification-compliant implementation. Well, not entirely from scratch, a lot of the work that has been done on the runtime will be re-usable as-is, and a lot of the experience/skill gained from the first proof of concept will be immensely valuable.

Anyway, stay tuned for more! 😉

---

If you’re curious about the internals of Letlang, feel free to take a look at the repository:

{{< cta-link
  url="https://github.com/linkdd/letlang"
  title="GitHub - linkdd/letlang: Functional language with a powerful type system."
  description="Functional language with a powerful type system. Contribute to linkdd/letlang development by creating an account on…"
>}}

Feel free to clap for this article to give me more visibility 🙂
