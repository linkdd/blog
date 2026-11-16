---
title: "A new milestone for Letlang"
date: "2023-03-13T06:30:50Z"
description: "Effect Handlers in Letlang are now a thing"
tags: ["letlang", "rust", "plt", "compilers", "programming"]
author: "David Delassus"
images: ["img/image-dcdcebcd80ca0263.png"]
---

![](img/image-dcdcebcd80ca0263.png)

---

For the past few months, I have been working on my own programming language, [Letlang](https://letlang.dev). You may have already read a bit about it in my [previous articles](/reading-lists/letlang).

I’m happy to announce that one of the biggest feature of the language is now specified, implemented, and working!

But what is that feature?

---

## Effect handlers

In Letlang, you can delegate computations to the caller of the function. Those are called **effects**.

In order to do this, you first need to declare the effect signature:

```
class loglevel(lvl: @debug | @info | @warn | @error);
effect log(lvl: loglevel, message: string) -> @ok;
```

This defines a symbol in the current scope, which can be used with the `perform` keyword. That keyword indicates that you want to delegate the handling of that effect to the caller:

```
result := perform log(@debug, "hello");
```

If the caller does not handle the effect, it is propagated to the caller’s caller, and so on until it has propagated to the Letlang runtime. If the Letlang runtime have a builtin handler for that effect, it will be called, otherwise the process will exit with an error.

Once the handler have been called, its return value is passed to the code which performed the effect, and execution resumes normally.

This part was already implemented, and the `std::io::println` and `std::io::readline` functions were actually performing effects already built-in the runtime.

The news here is about custom handlers. The developer can now intercept effects (with pattern matching ❤️):

```
module example::main;

import std::io;

class loglevel(lvl: @debug | @info | @warn | @error);
effect log(lvl: loglevel, message: string) -> @ok;

pub fn main() -> @ok {
  @ok := do {
    @ok := perform log(@debug, "a debug message");
    @ok := perform log(@info, "hello world");
    @ok;
  }
  intercept log(@debug, _message) {
    # silence debug loglevel
    @ok;
  }
  intercept log(_level, message) {
    std::io::println(message);
    @ok;
  };

  @ok;
}
```

The `do{}` block defines a new scope, and `intercept` clauses define effect handlers. When an effect is performed within the `do{}` block, it will use the first matching `intercept` clause as its handler. If no `intercept` clause matched, then the effect is propagated to the outer scope.

This is a huge step for Letlang, as it is one of the first feature I wanted in my language 🙂

---

## A note about exceptions

Exceptions are effects that do not resume execution.

Any value can be thrown as an exception, and just like effects, they will be propagated until they are caught, or until they reach the Letlang runtime, which will then terminate the process with an error.

Here is an example:

```
@error_caught := do {
  throw @error_a;
  throw @error_b;
}
catch @error_a {
  @error_caught;
}
catch @error_b {
  # this will never be executed
  @oops;
};
```

The `do{}` block can have `catch` clauses. When an exception is thrown, the first `catch` clause that matches will be executed, and the `do{}` block will return the value of the clause. If no clause matched, then the exception is propagated.

---

## Implementation

Declaring an effect will produce the following Rust code:

```rust
// effect log(message: string) -> @ok;

pub(crate) mod symbol_log {
  use super::*;

  #[allow(non_camel_case_types)]
  #[derive(Debug, Clone, PartialEq)]
  pub struct effect_log;

  #[async_trait]
  impl Effect for effect_log {
    async fn call(&self, context: Arc<Mutex<TaskContext>>, co: &FunctionCoroutine, args: Vec<Value>) -> Value {
      helpers::assert_param_count(&co, context.clone(), 1, args.len()).await;

      let paramtype_0 = StringType {};
      let paramval_0 = &args[0];
      helpers::assert_type(&co, context.clone(), &paramtype_0, &paramval_0).await;

      let result = co.yield_(FunctionInterruption::Effect { name: format!("{LL_MOD_PATH}::log"), args }).await;

      let return_type = ValueType { llval: Value::Atom(Atom(0)) };
      helpers::assert_type(&co, context.clone(), &return_type, &result).await;

      result
    }
  }
}
```

Let’s dissect this:

- every module-level declaration in Letlang will produce a module `symbol_<name>`
- the module will declare a type `<symbol_kind>_<symbol_name>`  
  for classes and functions, those types might have members of type `Box<dyn Type>` for each generic type parameter
- for effects, we implement the `Effect` trait, which has a `call` method taking as parameter the current coroutine and the effect arguments
- the implementation of that method is in charge of type-checking and yielding the coroutine with a `FunctionInterruption`

The `FunctionInterruption` type is defined as:

```rust
#[derive(Clone)]
pub enum FunctionInterruption {
  Effect { name: String, args: Vec<Value> },
  Exception(Value),
}
```

For the coroutine, we use the `genawaiter` crate, I’ve written about it in this article: [Do not wait for Rust generators](/posts/2022-05-11-do-not-wait-for-rust-generators)

{{< cta-link
  url="https://github.com/whatisaphone/genawaiter"
  title="GitHub - whatisaphone/genawaiter: Stackless generators on stable Rust."
  description="This crate implements stackless generators (aka coroutines) in stable Rust. Instead of using yield, which won't be…"
>}}

We then define the `FunctionCoroutine` type as:

```rust
use genawaiter::sync::Co;

pub type FunctionCoroutine = Co<FunctionInterruption, Value>;
```

Letlang functions will produce a `func_<name>` type which implements the `Function` trait:

```rust
pub mod symbol_main {
  use super::*;

  #[allow(non_camel_case_types)]
  #[derive(Debug, Clone, PartialEq)]
  pub struct func_main {}

  impl func_main {
    pub fn new() -> Self {
      Self {}
    }
  }

  impl Function for func_main {
    fn call(
      &self,
      context: Arc<Mutex<TaskContext>>,
      args: Vec<Value>,
    ) -> FunctionContinuation {
      async fn code_block(
        co: FunctionCoroutine,
        context: Arc<Mutex<TaskContext>>,
        args: Vec<Value>,
      ) -> Value {
        // type-check arguments
        // function body
        // type-check function return value
      }

      FunctionContinuation::new_boxed(|co| code_block(co, context, args))
    }
  }
}
```

The `FunctionContinuation` type is defined as:

```rust
use genawaiter::sync::Gen;
use std::pin::Pin;
use std::future::Future;

pub type FunctionContinuation = Gen<
  FunctionInterruption,
  Value,
  Pin<Box<(dyn Future<Output = Value> + Send + 'static)>>,
>;
```

Finally, calling a function will execute the following Rust code:

```rust
pub async fn call_function(
  co: &FunctionCoroutine,
  context: Arc<Mutex<TaskContext>>,
  func: &dyn Function,
  args: Vec<Value>,
) -> Value {
  let mut block = func.call(context.clone(), args);

  let ignored = Value::Boolean(bool::default());
  let mut state = block.resume_with(ignored);

  let func_result = loop {
    match state {
      GeneratorState::Yielded(interrupt) => {
        let res = co.yield_(interrupt.clone()).await;
        state = block.resume_with(res);
      },
      GeneratorState::Complete(res) => {
        break res;
      }
    }
  };

  func_result
}
```

Let’s dissect this:

- first, we get the generator (our `FunctionContinuation` type) which is returned by the `call` method
- then we start the generator (it needs an initial value that is ignored)
- then, we loop until the generator is complete
- every time the generator yields, we propagate the interruption to the parent coroutine

The `main` function will automatically start a new process, and each Letlang process will loop over its “root” function’s generator where it will intercept built-in handlers:

```rust
pub async fn run_process(
  context: TaskContext,
  func: Box<dyn Function>,
) -> Result<()> {
  let context_cell = Arc::new(Mutex::new(context));

  let task_args: Vec<Value> = vec![];
  let mut block = func.call(context_cell.clone(), task_args);

  let ignored = Value::Boolean(bool::default());
  let mut state = block.resume_with(ignored);

  loop {
    match &state {
      GeneratorState::Yielded(FunctionInterruption::Effect { name, args }) => {
        let val = builtins::effects::dispatch(
          context_cell.clone(),
          name,
          args,
        ).await?;
        state = block.resume_with(val);
      },
      GeneratorState::Yielded(FunctionInterruption::Exception(exc)) => {
        eprintln!(
          "[FATAL] Uncaught exception in task {:?}: {}",
          async {
            let context = context_cell.lock().await;
            context.pid.clone()
          }.await,
          exc.to_string(context_cell.clone()).await,
        );
        return Err(RuntimeError::UncaughtException);
      },
      GeneratorState::Complete(val) => {
        eprintln!(
          "Task {:?} terminated with: {}",
          async {
            let context = context_cell.lock().await;
            context.pid.clone()
          }.await,
          val.to_string(context_cell.clone()).await,
        );
        break;
      }
    }
  }

  Ok(())
}
```

Back to effects, when calling the `perform` keyword, the following Rust code is executed:

```rust
async {
  let effect = symbol_log::effect_log{};
  let args: Vec<Value> = vec![/* arguments */];

  effect.call(context.clone(), &co, args).await
}.await
```

The `context` and `co` arguments are from the `async fn code_block(...)` part of the `Function` trait implementation. The coroutine will be yielded within the `call` method of the effect, as shown earlier.

Similarly, a `throw` will simply produce the following Rust code:

```css
// let exc: Value
co.yield_(FunctionInterruption::Exception(exc.clone())).await;
unreachable!();
```

The final part, is the implementation of the `do{}` block. The body will be a generator as well, on which we will loop over and try to call the correct effect and/or exception handler:

```rust
async {
  let mut block_locals = Locals::new(Some(&locals));
  let block_context = context.clone();

  let mut block = Gen::new(|co| async move {
    #[allow(unused_variables)]
    #[allow(unused_mut)]
    let mut locals = block_locals;

    #[allow(unused_variables)]
    let context = block_context;

    // block body
  });


  let ignored = Value::Boolean(bool::default());
  let mut state = block.resume_with(ignored);

  let block_result = loop {
    let (finish, next_val) = 'continuation: {
      match state {
        GeneratorState::Yielded(FunctionInterruption::Effect { ref name, ref args }) => {
          // try to match an intercept clause
          // execute the clause's body if it did
          // and then: break 'continuation (false, res)

          // if no intercept clause matched, propagate:
          let res = co.yield_(FunctionInterruption::Effect { name: name.clone(), args: args.clone() }).await;
          break 'continuation (false, res);
        },
        GeneratorState::Yielded(FunctionInterruption::Exception(exc)) => {
          // try to match a catch clause
          // execute the clause's body if it did
          // and then: break 'continuation (true, res)

          // if no catch clause matched, propagate:
          co.yield_(FunctionInterruption::Exception(exc)).await;
          unreachable!();
        },
        GeneratorState::Complete(res) => {
          break 'continuation (true, res);
        }
      }
    };

    if finish {
      break next_val;
    }
    else {
      state = block.resume_with(next_val);
    }
  };

  block_result
}.await
```

Here, we use a labelled block to determine what to do with the `do{}` block body (do we need to resume it? was it completed? or do we need to exit it?).

But the gist is that, in Letlang, every block of code is a generator which can yield effects/exceptions and be resumed (or not).

---

## Conclusion

Using generators truly made the implementation simpler, it’s a shame that Rust async was stabilized while generators were not. Thankfully, the `genawaiter` crate bring that feature into the Rust stable branch (I don’t want to rely on Rust nightly).

I’m really happy I got this done, as it is IMHO one of the biggest Letlang selling points.

If you’re curious about the internals of Letlang, feel free to take a look at the repository:

{{< cta-link
  url="https://github.com/linkdd/letlang"
  title="GitHub - linkdd/letlang: Functional language with a powerful type system."
  description="Functional language with a powerful type system. Contribute to linkdd/letlang development by creating an account on…"
>}}

---

Feel free to clap for this article to give me more visibility 🙂

If you want to read my other articles about Letlang, it’s here → [Letlang Reading List](/reading-lists/letlang)
