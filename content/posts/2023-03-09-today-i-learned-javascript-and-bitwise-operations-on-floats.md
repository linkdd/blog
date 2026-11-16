---
title: "Today I learned: Javascript and bitwise operations on floats"
date: "2023-03-09T01:58:23Z"
description: "How did Javascript implement bitwise operations on floating point numbers?"
tags: ["letlang", "plt", "javascript", "bitwise", "floating-point"]
author: "David Delassus"
---

While working on my own programming language, [Letlang](https://letlang.dev) (read more about it [here](/reading-lists/letlang)), I was faced with an issue.

For the “Proof of Concept”, I decided to have only 64-bits floating point numbers instead of separate integer/float types, just like Javascript. Though , I intend to introduce those separate types later on.

This begs the question: **What about bitwise operations?**

On integer types, they are pretty straightforward, simply take the bits representation of the integer, and apply the bitwise operations (`^`, `|`, `&`, `~`) on those bits.

But for floats, it’s tricky. Rust (my compilation target) does not define bitwise operations on the `f64` type. Indeed, `4.2 | 2.3` does not make a lot of sense.

Applying the same rules of integers on floats would be the wrong thing to do, because in the IEE-754 specification, there are bit representations for `Infinity`, `NaN`, etc… which could be the result of such operations.

64-bits floating point numbers can represent integers up to 53 bits. Should I apply the bitwise operations on those 53 bits only? What about the signed vs unsigned? Also, when working with bitwise operations, we expect to work with 8 bits, 16 bits, 32 bits or 64 bits integers. 53 bits is so unexpected that it would be beyond wrong to use such a rule.

**NB:** To me, this is proof that using floats for all numbers is not something to wish for. But I’ll delay that change for later, there are other things I want to implement before.

Since I copied Javascript on that part, I decided to look up what rules they settled on. And what a surprise!

---

In Javascript, every bitwise operations converts the number to a 32 bits integer.

This is surprising, but I think it makes some sense.

While we can convert a 64-bits floating point number (with no fractional part) to a 64-bits integer, the opposite is not true. Indeed, 53 bits do fit in 64 bits, but 64 bits do not fit in 53 bits.

Casting the number first to a 32 bits integer ensure that the conversion back to floats will fit the result.

The equivalent Rust code would be:

```rust
let a: f64 = 4.2;
let b = a as i32;
// do bitwise operations on b
let result = b as f64;
```

---

While this behavior might be a bit surprising, I decided to reproduce it in Letlang.

It’s ok, since I plan to have separate types for integers and floats down the road, and bitwise operations will be defined only for integers.

Anyway, that’s it for this short post, if you enjoyed it, feel free to clap 👏 for it.
