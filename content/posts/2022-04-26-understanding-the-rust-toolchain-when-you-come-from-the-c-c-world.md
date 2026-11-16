---
title: "Understanding the Rust toolchain when you come from the C/C++ world"
date: "2022-04-26T21:23:29Z"
description: "While Cargo or Fleet are amazing tools to manage your project’s dependencies, it’s always interesting to see how it’s done under the hood."
tags: ["rust", "c", "compilers", "toolchain"]
author: "David Delassus"
images: ["img/image-ce2eb3d03ab3a304.jpg"]
---

![](img/image-ce2eb3d03ab3a304.jpg)

**DISCLAIMER:** This article is merely a bookmark for myself.

---

## In the C/C\+\+ world, the toolchain is pretty straightforward

It looks like this:

- `cc`/`cxx` : the compiler, which translates your C/C\+\+ code into Assembly
- `as` : the assembler, which translate the Assembly code into Machine Code
- `ld` : the linker, which create the final executable from your Machine Code

You feed your `.c`/`.cxx` files to the compiler, which will spit `.as` files (or `.s`, or `.asm`).

You feed those files to the assembler, which will spit `.o`/`.obj` files.

You can then archive those “object” files into archives:

- `.a`: usually an AR archive, for static libraries
- `.so`/`.dll`: similar to the `.a` but the executable will tell the Operating System to load this library at runtime

Finally, the linker will take your `.o`, `.a` and `.so` to make an executable.

The “object” files are also called **compilation units**. The libraries (either static or dynamic) are either provided by the Operating System (`*-dev` packages on Linux distributions) or by your package (by distributing the `.so` or `.dll` along your executable).

---

## But what does it look like in the Rust world?

The **compilation unit** in Rust is the **crate**. There is 2 kinds of crate:

- **a library crate:** this would be the equivalent to the `.a` archives in the C/C\+\+ world
- **a binary crate:** this is the final executable

A crate is, before all else, a module tree:

```
// foobar.rs
mod submodule {
  // ...
}
```

```
mod another_submodule; // exists in a separate file
```

If you compile this crate as a library, it will produce a `.rlib` file:

```
$ rustc \
    --crate-name foobar \
    --crate-type lib \
    ./foobar.rs
$ ls .
libfoobar.rlib
...
```

The `.rlib` file is an AR archive containing the “object” files. Those “object” files are split according to arbitrary rules that may change from version to version. This is our `.a` from the C/C\+\+ world!

This implies that `rustc --crate-type lib` is the equivalent of `cc`\+`as`.

The final executable is also built from a crate:

```
// myexe.rs
extern crate foobar;
```

```
fn main() {
  println!("hello world");
}
```

Using the following command:

```
$ rustc \
    --crate-name myexe \
    --crate-type bin \
    --extern "foobar=/path/to/libfoobar.rlib" \
    ./myexe.rs
$ ls .
myexe.exe
myexe.pdb
...
```

The `.pdb` file contains debugging symbols to use with your debugger.

The `--extern` argument works with the `extern crate` statement to locate your dependencies and link them together in a single static executable.

This implies that `rustc --crate-type bin` is the equivalent of `cc`\+`as`\+`ld`.

On a final note, the combination of `mod` and `use` statements make a better `#include`:

- the module is included **only once** (with `mod`) in the source tree
- the symbols provided by the module are referenced **as many time as you want** (with `use`)

---

## Conclusion

While [Cargo](https://doc.rust-lang.org/cargo/) or [Fleet](https://fleet.rs) are amazing tools to manage your project’s dependencies, it’s always interesting to see how it’s done under the hood.

Especially when you’re working on a compiled [programming language](https://letlang.dev) that targets Rust 😅
