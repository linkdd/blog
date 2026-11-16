---
title: "astmaker — A DSL in Rust for programming language designers"
date: "2023-05-17T17:28:55Z"
description: "Build Abstract Syntax Trees and tree-walking models quickly in Rust."
tags: ["rust", "plt", "compilers", "ast"]
author: "David Delassus"
---

{{< cta-link
  url="https://github.com/linkdd/astmaker"
  title="GitHub - linkdd/astmaker: Build Abstract Syntax Trees and tree-walking models quickly in Rust."
  description="Build Abstract Syntax Trees and tree-walking models quickly in Rust. This example creates an AST for simple math…"
>}}

While working on [Letlang](https://letlang.dev), I’ve had to design its AST with the Rust type system, which is a damn good fit for that use case!

Then, I started writing tree-walking algorithms to walk through that AST during the different phases of compilation (scope analysis, code generation, …).

But it has a lot of boilerplate code which I would love to remove.

This is why I made the `astmaker` Rust crate. It provides a simple DSL to define an AST and a tree-walking algorithm for it. Let’s dive in!

---

## Installation

Simply add to your `Cargo.toml`:

```ini
[dependencies]
astmaker = "0.1"
```

---

## Define your AST

In this example, we will create an AST for simple math expressions. It will support binary operations (addition, subtraction, multiplication, division) and unary operations (plus sign, minus sign).

To do this, we use the `ast!` macro:

```rust
use astmaker::ast;

#[derive(Debug, Clone, PartialEq)]
pub enum BinOp {
  Add,
  Sub,
  Mul,
  Div,
}

#[derive(Debug, Clone, PartialEq)]
pub enum UnOp {
  Add,
  Sub,
}

ast!{
  location = ();

  pub node Expression =
    | BinOp -> Node<BinaryOperation>
    | UnOp -> Node<UnaryOperation>
    | Num -> Node<Number>
    ;

  pub node BinaryOperation = {
    lhs: Node<Expression>,
    op: BinOp,
    rhs: Node<Expression>,
  }

  pub node UnaryOperation = {
    op: UnOp,
    expr: Node<Expression>,
  }

  pub node Number = {
    value: f64,
  }
}
```

The first statement of the DSL is to specify what type to use to store the location information of each node. This is useful if you want to map a node to the source code that generated it.

```rust
ast!{
  location = ();

  // ...
}
```

Here, we don’t need any so we set it to the unit type. But the following could have been equally valid:

```rust
ast!{
  location = (usize, usize);

  // ...
}
```

```rust
struct LocationInfo {
  filename: String,
  line: usize,
  col: usize,
}

ast!{
  location = LocationInfo;

  // ...
}
```

Then, we define the `Expression` node:

```rust
ast!{
  // ...

  pub node Expression =
    | BinOp -> Node<BinaryOperation>
    | UnOp -> Node<UnaryOperation>
    | Num -> Node<Number>
    ;

  // ...
}
```

This statement will generate a Rust **enum**. As opposed to the following statement:

```rust
ast!{
  // ...

  pub node Number = {
    value: f64,
  }

  // ...
}
```

which will generate a Rust **struct**.

The `ast!` macros will generate the following types:

```rust
pub trait NodeAttributes {
  type Attributes;
}

#[derive(Debug, Clone, PartialEq)]
pub struct Node<T: NodeAttributes> {
  location: (), // or any other location type defined
  attrs: Option<T::Attributes>,
  data: Box<T>,
}
```

For each node defined, it will automatically implement the `NodeAttributes` trait:

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Number {
  value: f64,
}

impl NodeAttributes for Number {
  type Attributes = ();
}
```

If we want to override the attributes type, we can write:

```rust
struct MyAttributes;

ast!{
  // ...

  pub node Number where attrs: MyAttributes = {
    value: f64,
  }

  // ...
}
```

Then, the implementation of the `NodeAttributes` trait will be:

```rust
impl NodeAttributes for Number {
  type Attributes = MyAttributes;
}
```

---

## Write a simple interpreter

Here, we use the `model!` macro which provides a simple way to implement the tree-walking algorithm:

```rust
use astmaker::model;

pub struct Interpreter;

model!{
  for Interpreter -> f64 {
    where Expression => {
      match node.data.as_mut() {
        Expression::BinOp(child_node) => context.visit(child_node),
        Expression::UnOp(child_node) => context.visit(child_node),
        Expression::Num(child_node) => context.visit(child_node),
      }
    },
    where BinaryOperation => {
      let lhs = context.visit(&mut node.data.lhs);
      let rhs = context.visit(&mut node.data.rhs);

      match node.data.op {
        BinOp::Add => lhs + rhs,
        BinOp::Sub => lhs - rhs,
        BinOp::Mul => lhs * rhs,
        BinOp::Div => lhs / rhs,
      }
    },
    where UnaryOperation => {
      let val = context.visit(&mut node.data.expr);

      match node.data.op {
        UnOp::Add => val,
        UnOp::Sub => -val,
      }
    },
    where Number => node.data.value,
  }
}
```

The syntax is as follow:

```rust
model!{
  for $YOUR_TYPE -> $RETURN_TYPE {
    where $NODE_TYPE => $CLAUSE_BODY,
  }
}
```

The macro will define the following trait:

```rust
pub trait Visitable<T: NodeAttributes> {
  fn visit(
    context: &mut Interpreter, // or $YOUR_TYPE
    node: &mut Node<T>,
  ) -> f64; // or $RETURN_TYPE
}
```

For each `where` clause, it will implement the `Vistiable` trait for the node data variant:

```rust
impl Visitable<Number> for Number {
  fn visit(context: &mut Interpreter, node: &mut Node<Number>) {
    node.data.value // or $CLAUSE_BODY
  }
}
```

Finally, the macro will add a public function to the `Interpreter` struct:

```rust
impl Interpreter /* or $YOUR_TYPE */ {
  pub fn visit<T: NodeAttributes + Visitable<T>>(
    &mut self,
    node: &mut Node<T>,
  ) -> f64 /* or $RETURN_TYPE */ {
    T::visit(self, node)
  }
}
```

Now, we have static dispatch with `Interpreter::visit()` without the boilerplate!

---

## Usage

Let’s test this interpreter:

```rust
#[test]
fn eval() {
  let mut tree = Node::new((), Expression::BinOp(
    Node::new((), BinaryOperation {
      lhs: Node::new((), Expression::Num(
        Node::new((), Number { value: 1.0 })
      )),
      op: BinOp::Add,
      rhs: Node::new((), Expression::Num(
        Node::new((), Number { value: 2.0 })
      ))
    })
  ));

  let mut interpreter = Interpreter;
  let res = interpreter.visit(&mut tree);
  assert_eq!(res, 3.0);
}
```

Running this test should succeed. A few notes though:

- the `Node::new()` method does the `Box::new()` allocation and initializes the node attributes field to `None`
- building the AST by hand can be quite verbose (we need to pass a location to every node instance), but usually you will do this in your grammar (using `rust-peg`, `lalrpop` or another parser framework)

---

## Conclusion

This simple crate will allow me to remove a lot of boilerplate code and simplify the architecture of my language.

Though, it is far from being complete, for example I don’t support generics and lifetimes, which might be useful in Rust, don’t you think? 😁

Feel free to star the repository on Github if it sparks your interest or curiosity:

{{< cta-link
  url="https://github.com/linkdd/astmaker"
  title="GitHub - linkdd/astmaker: Build Abstract Syntax Trees and tree-walking models quickly in Rust."
  description="Build Abstract Syntax Trees and tree-walking models quickly in Rust. This example creates an AST for simple math…"
>}}

And don’t hesitate to clap 👏 for this article to give it more visibility.

---

{{< cta-link
  url="/reading-lists/letlang"
  title="Letlang"
  description="Blog posts about my programming language Letlang"
>}}
