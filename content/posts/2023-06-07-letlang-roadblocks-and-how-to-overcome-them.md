---
title: "Letlang — Roadblocks and how to overcome them"
date: "2023-06-07T11:05:45Z"
description: "In this article, I’ll expose some of the roadblocks I’ve met and what solution I have thought of."
tags: ["letlang", "plt", "compilers", "rust", "devlog"]
author: "David Delassus"
images: ["img/image-7cfc28cbfa16a48b.png"]
---

![](img/image-7cfc28cbfa16a48b.png)

---

## Introduction

I’ve been working on and off on my own programming language, [Letlang](https://letlang.dev), for a while now. You may have already read a bit about it in my [previous articles](/reading-lists/letlang).

I have made a lot of progress on the design of the runtime, and the features I want.

But we’re still nowhere near something “stable”. I met some roadblocks which makes me rethink the semantics and the syntax as a consequence.

In my opinion, the syntax needs to reflect clearly the semantics of the language. The syntax is “derived” from the semantics.

In this article, I’ll expose some of the roadblocks I’ve met and what solution I have thought of.

---

## Quick Recap: The type system

The type system of Letlang is dynamic and enforced at runtime. This is because of the very nature of what a “type” is in Letlang.

For a quick reminder: Values do not have a type, instead they belong to a collection, called a “class” (inspired from proper classes in set theory). The class defines the constraints a value must respect to be considered “part of the class”.

The example I like to give is with the classes `even` and `odd` :

- `even` : all integers `n` such as `n % 2 = 0`
- `odd` : all integers `n` such as `n is not even`

In the runtime, classes are modelled as functions that takes a value and returns a boolean indicating whether or not it belongs to the type.

This model prevents static type checking. When checking types during compile-time, I need to be able to verify the “compatibility” between types, consider the following classes:

- `mod2` : all integers `n` such as `n % 2 = 0`
- `mod3` : all integers `n` such as `n % 3 = 0`
- `mod4` : all integers `n` such as `n % 4 = 0`

Here, we can say that all members of `mod4` are also members of `mod2`. But we can’t say the same for `mod3`. This means:

- `mod4` is compatible with `mod2`
- `mod3` is not compatible with `mod2`
- `mod2` is not compatible with `mod4` (not all multiples of 2 are multiples of 4\)

The compiler would need to prove such statements, possibly using an SMT solver, which is slow and may lead to false positives/negatives in more complex cases.

But at runtime, we can simply run the check with the runtime value and raise a `TypeError` exception. **And I’m fine with that.**

---

## Functions, lambdas and higher-order functions

“Functions as first-class citizens” means that functions are also values that can be constructed/returned at runtime, consider the following in Python:

```python
def tag(name: str):
    def func(content: str):
        return f"<{name}>{content}</{name}>"

    return func

div = tag("div")
assert div("hello") == "<div>hello</div>"
```

I did want the same feature. But then, what is the type of a function?

At first, I thought something like: `func[(params...) -> return-type]`.

Can you see the problem?

No?

Ok, so let’s try to type-check at runtime. Consider the following functions:

```
func div4(n: mod4) -> int {
  n / 4;
}

func operation(f: func[(mod2) -> int]) -> int {
  f(2);
}
```

The `operation` function expects as argument a function which takes a `mod2`. But `div4` expects a `mod4`.

If I pass the `div4` function to the `operation` function, I would expect the type-check to fail. For this, the runtime would need to prove that not all members of `mod2` are members of `mod4`. Aaaaaaand we’re back to the beginning.

I could do that with an SMT solver, which would be a huuuuge performance hit, which would increase the size of the executable, and could still have false positives/negatives.

I decided to give up on “functions as first-class citizens”. But I still want lambdas and higher-order functions.

A lambda does 2 things:

- creates an anonymous function
- captures an “environment” (the scope it has been created in)

In an immutable language like Letlang, the “environment capture” is very easy: just copy it.

To avoid going back to what I call the “SMT problem”, the type of a lambda will be: `lambda/arity`. So for example, a lambda which accepts 2 parameters will have the type `lambda/2`.

Here is an example:

```
func operation(f: lambda/1) -> int {
  f(2);
}
```

The problem is that now, the signature of `operation` is not self-documenting anymore, what is the expected signature of `f` ?

To be honest, **I haven’t found a satisfying solution yet.**

It is important to note that type-checking happens at the function’s boundaries:

- we type-check the arguments before executing the function
- we type-check the return value after executing the function

But **lambdas are not functions** and have no type-hints. Therefore, there is no type-checking at the lambda’s boundaries.

---

## Function dispatch

In Erlang, Elixir, Haskell, and many ML languages, we can use pattern-matching in the function definition:

```
def fibonacci(0), do: 0
def fibonacci(1), do: 1
def fibonacci(n), do: fibonacci(n - 2) + fibonacci(n - 1)
```

In Letlang, I delayed implementing that feature because of the Rust-like `match` expression:

```
func fibonacci(n: int) -> int {
  match n {
    0 => 0,
    1 => 1,
    n => fibonacci(n - 2) + fibonacci(n - 1),
  };
}
```

But I like that feature too much to delay it any further, which prompted me to redesign the syntax of the language accordingly.

First, let’s put back the `let` in Letlang. A module will contains declarations of the form:

```
let symbol: definition;
```

Which gives us:

```
# no predicate
let type_name: class[base-type];

# with predicate
let type_name: class[base-type] {
  pattern (when guard-expression)? -> { ... },
  ...
};

let function_name: func[signature] {
  pattern (when guard-expression)? -> { ... },
  ...
};

let tailrecursive_function_name: tailrec[signature] {
  pattern (when guard-expression)? -> { ... },
  ...
};

let effect_name: effect[signature];
```

As an example:

```
let even: class[int] {
  n -> { n % 2 = 0; },
};

let odd: class[int] {
  n when n is not even -> { true; },
  _ -> { false; },
};

let collatz: func[(int) -> int] {
  n when n is even -> { n / 2; },
  n -> { 3 * n + 1; },
};

let log: effect[(string) -> @ok];
```

The syntax is uniform, concise, and easy to read. I like it 🙂

When no clauses matches, a `MatchFailed` exception is raised. And if that exception is raised in a guard expression ( `when ...` ), the guard will return `false`.

Similarly, for lambda expressions:

```
let main: func[() -> @ok] {
  () -> {
    collatz := lambda/1 {
      n when n is even -> { n / 2 },
      n when n is odd -> { 3 * n + 1; },
    };

    4 := collatz(1);
    collatz("hello");  # MatchFailed exception is raised

    @ok;
  },
};
```

---

## The module system

Up until now, a Letlang file had to specify its module name, just like we do in Erlang, or Go:

```
module example::main;

import std::io;

# ...
```

This has 2 problems:

- **name conflicts** if one of your dependency declares the same module name
- **dependency management:** it makes it hard to know what module come from which package

In Erlang, this makes sense, since modules are first-class citizens. In Elixir you can even define multiple modules per files, and nest modules. **They are more than just namespaces**.

Instead, I decided to go with the most simple module/import system there is: **the file system**.

```
from std::io import { println };
```

This import expects to find a `std/io.let` file in one of the search directories (similar to include paths in C and C\+\+). And that file (or module) needs to define a `println` symbol.

**NB:** Ah yes, no more `public` / `private`, everything is public, like in Python. I might reintroduce this in the future, but I’m not sure.

How does it solve name conflict?

It doesn’t really. The first file found will be the one used as module. If they don’t export the expected symbols, compilation will fail way before code-generation.

How does it solve dependency management?

The compiler now only needs a list of search paths, which makes the implementation of a package manager in the future easier: “just put the files in that folder”.

We have now many options:

**Option 1:**

There is a folder `/usr/lib/letlang/` configured as search directory.

The package manager pulls the Letlang files in a `package_name` folder in that search directory.

Now we can import `package_name::module`.

**Option 2:**

There is a folder `/usr/lib/letlang/packages/` and a file `/usr/lib/letlang/search.list`.

The package managers extracts the package’s archive (containing the `letproject.toml` and `src/` directory, etc…) in the `packages/` folder, then add the `packages/package_name/src/` folder to the `search.list` file.

**Option N:** … whatever we can imagine …

Letlang files do not need to declare their module name anymore, it’s their filename.

---

## What’s next?

I’d like to do a big refactor of the source code, splitting the code into 3 crates:

- `frontend`: containing the AST, the lexer and the grammar
- `backend`: containing the semantic validation, the code-generation
- `runtime`: containing the core dependency of every Letlang project

I’d like to use my **astmaker** crate as well:

{{< cta-link
  url="/posts/2023-05-17-astmaker-a-dsl-in-rust-for-programming-language-designers"
  title="astmaker — A DSL in Rust for programming language designers"
  description="Build Abstract Syntax Trees and tree-walking models quickly in Rust."
>}}

{{< cta-link
  url="https://github.com/linkdd/astmaker"
  title="GitHub - linkdd/astmaker: Build Abstract Syntax Trees and tree-walking models quickly in Rust."
  description="Build Abstract Syntax Trees and tree-walking models quickly in Rust. This example creates an AST for simple math…"
>}}

And for code-generation, I’d like to switch from [askama](https://docs.rs/askama/0.12.0/askama/) templates to the [quote](https://docs.rs/quote/latest/quote/) crate and return a`TokenStream` instead of a `String`.

As of now, everything I discussed in that article is not yet implemented. I don’t have a lot of time recently, but I’ll try to get to it as soon as I can 🙂

---

## Conclusion

Creating my own programming language is a fun experience, even though I can meet a lot of roadblocks (that would be obvious to more experienced language designers).

If you’re curious about the internals of Letlang, feel free to take a look at the repository:

{{< cta-link
  url="https://github.com/linkdd/letlang"
  title="GitHub - linkdd/letlang: Functional language with a powerful type system."
  description="Functional language with a powerful type system. Contribute to linkdd/letlang development by creating an account on…"
>}}

---

Feel free to clap for this article to give me more visibility 🙂

If you want to read my other articles about Letlang, it’s here → [Letlang Reading List](/reading-lists/letlang)
