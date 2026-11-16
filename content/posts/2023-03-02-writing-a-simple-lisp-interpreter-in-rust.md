---
title: "Writing a simple Lisp interpreter in Rust"
date: "2023-03-02T23:00:26Z"
description: "Lisp is a family of multi-paradigm programming languages (functional, procedural, reflective, …), which can be similar to Lambda Calculus…"
tags: ["rust", "lisp", "plt", "programming"]
author: "David Delassus"
images: ["img/image-cfef9fe8d5d9cec8.png"]
---

![](img/image-cfef9fe8d5d9cec8.png)

---

Lisp is a family of multi-paradigm programming languages (functional, procedural, reflective, …), which can be similar to [Lambda Calculus](https://en.wikipedia.org/wiki/Lambda_calculus). The name stands for “list processing”.

The syntax is based on the concept of [S-Expression](https://en.wikipedia.org/wiki/S-expression) (short for: Symbolic Expression). The code is therefore organized in a tree-like data structure. In Lisp, code is data, and data is code.

In this article, we will write a very simple (non standard?) Lisp interpreter in Rust. This will be a great way to explore the Rust ecosystem, and what it can do for us, language designers.

The interpreter will be split in 3 parts:

- the frontend, to parse S-Expressions
- the backend, to evaluate the S-Expressions
- the REPL, to provide a simple CLI to execute Lisp code

**NB:** All the code present in this article is available in this repository:

{{< cta-link
  url="https://github.com/linkdd/lispers"
  title="GitHub - linkdd/lispers: Educational project: How to implement a Lisp interpreter in Rust?"
  description=""
>}}

---

## Part 1: Parsing S-Expressions

Lisp usually defines 3 types of data:

- **atoms:** symbols, numbers, booleans
- **strings:** delimited by double-quotes
- **lists:** delimited by parenthesis

Here is an example of S-Expression:

```
(a b (c d) "hello world" (+ 1 2))
```

---

### Step 1: Lexical analysis

Or in other words: transforming the stream of characters into a stream of tokens. If we take the above example, the corresponding tokens would be:

```
ParenOpen
Symbol("a")
Symbol("b")
ParenOpen
Symbol("c")
Symbol("d")
ParenClose
String("hello world")
ParenOpen
Symbol("+")
Integer(1)
Integer(2)
ParenClose
ParenClose
```

For this, we will use the Rust crate `logos` which provides a very easy way to define the lexer.

{{< cta-link
  url="https://github.com/maciejhirsz/logos"
  title="GitHub - maciejhirsz/logos: Create ridiculously fast Lexers"
  description="Create ridiculously fast Lexers. Logos has two goals: To make it easy to create a Lexer, so you can focus on more…"
>}}

We start by defining a `Token` enumeration:

```rust
use logos::Logos;

#[derive(Logos, Debug, Clone, PartialEq)]
pub enum Token {
```

We defined our tokens use macros from the `logos` crate, let’s start simple with the parenthesis and the booleans:

```rust
// ...
pub enum Token {
  #[token("(")]
  ParenOpen,

  #[token(")"]
  ParenClose,

  #[token("true")]
  True,

  #[token("false")]
  False,
```

The `token` macro defines a parser that match the string passed in argument, that parser will return the corresponding enum variant.

**NB:** `logos` disambiguates tokens with the following rule of thumb:

- Longer beats shorter
- Specific beats generic

For more information, the [README](https://github.com/maciejhirsz/logos#token-disambiguation) goes into more details.

For symbols, we want a token that will capture everything that is not parenthesis, or a string, (or a number, but we’ll see how to do that later):

```rust
// ...
pub enum Token {
  // ...

  #[regex("[^ \\t\\r\\n\\f\"\\(\\)]+", |lex| lex.slice().parse())]
  Symbol(String),
```

The `regex` macro defines a parser that match using the regex passed in argument, that parser will return the corresponding enum variant. This variant captures the slice of characters matched by the parser. We pass a custom lambda to turn the `lex.slice()` (which returns `&str`) into an owned `String`. This simplifies lifetime management.

**NB:** While this regex (generic token) will match `true` and `false`, the specific tokens defined with the macro `token` have priority and will match before. But `truefalse` or `falsetrue` is longer, and therefore the Symbol token will match instead.

Similarly, we can define a String token:

```rust
use snailquote::unescape;

// ...
pub enum Token {
  // ...
  #[regex("\"(?:[^\"]|\\\\\")*\"", |lex| {
    unescape(lex.slice())
  })]
  String(String),
```

The captured slice is `"\"hello world\""`, we don’t want those extra quotes, and we wish to unescape potential escaped-quotes in the string. To do this, we rely on the Rust crate `snailquote`.

{{< cta-link
  url="https://github.com/euank/snailquote"
  title="GitHub - euank/snailquote: A shell-inspired escape+unescape library for strings"
  description="This library provides functions to escape and unescape strings. It escapes them in a roughly 'sh' compatible way (e.g…"
>}}

Next, we want to parse decimal numbers (potentially written in scientific notation). Rust already provides a `&str::parse()` function which returns an `f64`. We only need to define a regex.

Because decimal numbers can already be matched by the `Symbol` token’s regex, we give this one a higher priority:

```rust
// ...
pub enum Token {
  // ...
  #[regex(r"[+-]?((\d+\.?\d*)|(\.\d+))(([eE][+-]?)?\d+)?", |lex| {
    lex.slice().parse()
  }, priority = 2)]
  Float(f64),
```

For integers, we want to parse using the prefixes `0b` (binary), `0o` (octal) and `0x` (hexadecimal). With no prefixes, we should parse the number as base 10 (decimal). To do this, we rely on the Rust crate `parse_int`.

{{< cta-link
  url="https://gitlab.com/dns2utf8/parse_int"
  title="dns2utf8 / parse_int · GitLab"
  description="GitLab.com"
>}}

We will generate one token per base to split the regex and make it more manageable, and since both the `Symbol` tokens and the `Float` tokens can match integers (at least base 10 for the `Float` tokens), we also give those integer tokens a higher priority:

```rust
// ...
pub enum Token {
  // ...

  #[regex(r"0b_*[01][_01]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase2(i64),

  #[regex(r"0o_*[0-7][_0-7]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase8(i64),

  #[regex(r"[+-]?(0|[1-9][_0-9]*)", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase10(i64),

  #[regex(r"0x_*[0-9a-fA-F][_0-9a-fA-F]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase16(i64),
```

Finally, `logos` requires a default token for error handling, and skipping characters (for whitespaces and comments):

```rust
// ...
pub enum Token {
  // ...

  #[error]
  #[regex(r"[ \t\r\n\f]+", logos::skip)]
  Error,
}
```

The full enum should look like this:

```rust
use logos::Logos;
use snailquote::unescape;

#[derive(Logos, Debug, Clone, PartialEq)]
pub enum Token {
  #[token("(")]
  ParenOpen,

  #[token(")")]
  ParenClose,

  #[token("true")]
  True,

  #[token("false")]
  False,

  #[regex("[^ \\t\\r\\n\\f\"\\(\\)]+", |lex| lex.slice().parse())]
  Symbol(String),

  #[regex("\"(?:[^\"]|\\\\\")*\"", |lex| {
    unescape(lex.slice())
  })]
  String(String),

  #[regex(r"[+-]?((\d+\.?\d*)|(\.\d+))(([eE][+-]?)?\d+)?", |lex| {
    lex.slice().parse()
  }, priority = 2)]
  Float(f64),

  #[regex(r"0b_*[01][_01]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase2(i64),

  #[regex(r"0o_*[0-7][_0-7]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase8(i64),

  #[regex(r"[+-]?(0|[1-9][_0-9]*)", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase10(i64),

  #[regex(r"0x_*[0-9a-fA-F][_0-9a-fA-F]*", |lex| {
    parse_int::parse::<i64>(lex.slice())
  }, priority = 3)]
  IntegerBase16(i64),

  #[error]
  #[regex(r"[ \t\r\n\f]+", logos::skip)]
  Error,
}

impl std::fmt::Display for Token {
  fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
    write!(f, "{:?}", self)
  }
}
```

Now, we can use our lexer to do the lexical analysis of our source code:

```cpp
let tokens = Token::lexer("(a b c)");
```

---

### About string interning…

We want to intern the symbols strings, in order to speed up lookups and comparisons later on. To do this, we will use the Rust crate `string-interner`.

{{< cta-link
  url="https://github.com/robbepop/string-interner"
  title="GitHub - Robbepop/string-interner: A data structure to efficiently intern, cache and restore…"
  description="Continuous Integration Test Coverage Documentation Crates.io A data structure to cache strings efficiently, with…"
>}}

This crate provides 3 kinds of symbols:

- `SymbolU16` for 2-bytes symbols
- `SymbolU32` for 4-bytes symbols
- `SymbolUsize` for platform-dependent size symbols

They all implement the `Symbol` trait, also provided by the crate.

It also provides 4 types of backends (each with different trade-offs):

- `SimpleBackend`
- `StringBackend`
- `BufferBackend`
- `BucketBackend`

They all implement the `Backend` trait, also provided by the crate. This trait has an associated type `Symbol`.

As an exercise, in the next parts, we will make our code to be generic over the backend and symbol implementation. We will also need to use the symbol as a key in a hashmap later on, so it must implement the `Hash` trait.

A limitation of the parser generator crate we will use is that it does not support (yet?) associated types to traits. The following code is a workaround:

```rust
use std::hash::Hash;

pub use string_interner::{symbol, backend};

pub trait Symbol: symbol::Symbol + Hash {}
pub trait Backend<S: Symbol>: backend::Backend<Symbol = S> {}

impl Symbol for symbol::SymbolU16 {}
impl Symbol for symbol::SymbolU32 {}
impl Symbol for symbol::SymbolUsize {}

impl<S: Symbol> Backend<S> for backend::SimpleBackend<S> {}
impl<S: Symbol> Backend<S> for backend::StringBackend<S> {}
impl<S: Symbol> Backend<S> for backend::BufferBackend<S> {}
impl<S: Symbol> Backend<S> for backend::BucketBackend<S> {}

pub use string_interner::StringInterner;
```

---

### Step 3: Grammatical analysis, the AST

During the lexical analysis, we transformed the source code into tokens that have more meaning than just characters. During grammatical analysis, we will generate a structure that represent the information expressed by the code.

This step is important in order to reject invalid sequences of tokens (syntax errors). For example, this sequence of tokens is not a valid S-Expression:

```
// ((a b)
ParenOpen ParenOpen Symbol("a") Symbol("b") ParenClose
```

Ideally, we want to parse the following S-Expression:

```
(a b (c d))
```

Into the following tree:

```
List(
  Literal(Symbol("a")),
  Literal(Symbol("b")),
  List(
    Literal(Symbol("c")),
    Literal(Symbol("d")),
  ),
)
```

Let’s define our Absract Syntax Tree:

```rust
use std::hash::Hash;
use crate::intern::Symbol;

#[derive(Debug, Clone, PartialEq)]
pub enum Literal<S: Symbol> {
  Boolean(bool),
  Integer(i64),
  Float(f64),
  String(String),
  Symbol(S),
}

#[derive(Debug, Clone, PartialEq)]
pub enum SExpression<S: Symbol> {
  Literal(Literal<S>),
  List(Vec<SExpression<S>>),
}
```

---

### Step 3: Grammatical Analysis, the grammar (choosing a library)

Now that the AST is defined, we need to specify the grammar. There are a few different ways to do this.

**Using parser combinators**

Parsers tries to consume tokens, and return a result (ie. an AST node). Parser can be combined through composition ([higher order functions](https://en.wikipedia.org/wiki/Higher-order_function) ❤️) to produce higher level parsers.

Your final grammar is only one parser that returns the whole AST.

For a parser combinator framework, I would recommend the crate `nom`.

{{< cta-link
  url="https://github.com/rust-bakery/nom"
  title="GitHub - rust-bakery/nom: Rust parser combinator framework"
  description="nom is a parser combinators library written in Rust. Its goal is to provide tools to build safe parsers without…"
>}}

**Using a LALR(1) parser generator**

LALR(k) means “Look Ahead, Left-to-Right Rightmost derivation”. The `k` indicates the number of token to look ahead.

This means the generated parser will match rules by looking ahead in the stream of tokens. It will read tokens from left to right, and disambiguates using the rightmost token.

As an example, consider a grammar which have 3rules:

- `rule1` will match the tokens `A A`
- `rule2` will match the tokens `A B`
- `rule3` will match the tokens `B A`

When the parser finds the token `A`, both `rule1` and `rule2` could match, the token ahead will disambiguates the rules.

But now, consider those rules instead:

- `rule1` will match the tokens `A A A`
- `rule2` will match the tokens `A A B`
- `rule3` will match the tokens `B A A`

When the parser find the token `A`, both `rule1` and `rule2` could match, but the token ahead won’t disambiguates the rules. Indeed, `A A` could still be matched by both rules. We’d need to look at the token ahead one more time, but this is not LALR(1) anymore, this would be LALR(2).

For a LALR(1) parser, I would recommend the crate `lalrpop`.

{{< cta-link
  url="https://github.com/lalrpop/lalrpop"
  title="GitHub - lalrpop/lalrpop: LR(1) parser generator for Rust"
  description="LALRPOP is a Rust parser generator framework with usability as its primary goal. You should be able to write compact…"
>}}

**Using Parsing Expression Grammars**

PEG parsers disambiguates rules using “ordered choice”. In other words, the first rule that match is returned, even if subsequent rules could match.

They have no limit on the number of tokens they can look ahead and are in the category of “recursive descent parsers”.

For a PEG parser, I recommend the Rust crate `peg`.

{{< cta-link
  url="https://github.com/kevinmehall/rust-peg"
  title="GitHub - kevinmehall/rust-peg: Parsing Expression Grammar (PEG) parser generator for Rust"
  description="Documentation | Release Notes rust-peg is a simple yet flexible parser generator that makes it easy to write robust…"
>}}

---

### Step 3: Grammatical Analysis, the grammar (implementation)

For the purpose of this article, we will use a PEG parser generator, with the Rust crate `peg`.

By default, `peg` will parse a stream of characters, which means we would need to care about whitespaces. This would, in my opinion, complexify the grammar needlessly. Instead, we want it to parse our stream of tokens that we got from **Step 1**, with the crate `logos`.

To do this, we need a `TokenStream` structure that will implement the following traits:

- `peg::Parse` : required for all input types
- `peg::ParseElem` : implements the `[_]` pattern operator, which we will use like this: `[Token::True]`
- `peg::ParseLiteral` : implements matching against a `"string"` literal (which we won’t use)
- `peg::ParseSlice` : implements the `$()` operator, returning a slice from a span of indeces (which we won’t use)

First, let’s define our `TokenStream` structure:

```rust
use line_col::LineColLookup;
use logos::{Logos, Span};

// our Result and SyntaxError types
use crate::prelude::*;

// Token from Step 1
mod tokenizer;
pub use self::tokenizer::Token;

pub struct TokenStream<'source> {
  filename: Option<std::path::PathBuf>,
  size: usize,
  tokens: Vec<(Token, Span)>,
  linecol_lookup: LineColLookup<'source>,
}

impl<'source> TokenStream<'source> {
  pub fn new(filename: Option<std::path::PathBuf>, input: &'source str) -> Result<Self> {
    let token_stream = Self {
      filename,
      size: input.len(),
      tokens: Token::lexer(input).spanned().collect(),
      linecol_lookup: LineColLookup::new(input),
    };

    for (token, span) in token_stream.tokens.iter() {
      if let Token::Error = token {
        let (line, col) = token_stream.linecol_lookup.get(span.start);

        return Err(SyntaxError::InvalidToken {
          token: format!("{}", token),
          filename: token_stream.filename,
          line,
          col
        })
      }
    }

    Ok(token_stream)
  }
}
```

The `Span` type is the range in the character stream our token corresponds to.

Also, I’m using the Rust crate `line_col` out of laziness, to map a `Span` to a line/column number.

{{< cta-link
  url="https://github.com/TheBerkin/line-col-rs"
  title="GitHub - TheBerkin/line-col-rs: Rust crate for calculating line/column positions within a string"
  description="A simple crate for calculating 1-based line/col numbers for a string slice. This crate offers two methods for…"
>}}

While `Token::lexer(input)` will return an iterator of `Token`,   
`Token::lexer(input).spanned()` will return an iterator of `(Token, Span)`.

We store it in a `Vec` and will use it in the `peg` traits implementations. We also make sure that there is no `Token::Error` variant, which would mean our lexer failed to parse the input string.

The `peg::Parse` trait have an associated type `PositionRepr` that we need to define, so let’s do it:

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct TokenLocation {
  pub filename: Option<std::path::PathBuf>,
  pub linecol: (usize, usize),
  pub token: Option<Token>,
}

impl std::fmt::Display for TokenLocation {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    let (line, col) = self.linecol;
    let filename = self.filename
      .as_ref()
      .and_then(|path| path.to_str())
      .unwrap_or("<>");

    write!(f, "{}[{};{}]", filename, line, col)
  }
}
```

If an error happens during parsing, `peg` will return a n instance of `peg::error::ParseError<TokenLocation>`. We implement the `From` trait for our own `SyntaxError` type:

```rust
type ParseError = peg::error::ParseError<TokenLocation>;

impl From<ParseError> for SyntaxError {
  fn from(err: ParseError) -> Self {
    let TokenLocation {
      token,
      filename,
      linecol: (line, col),
    } = err.location;

    Self::UnexpectedToken {
      token: token.map(|tok| tok.to_string()).unwrap_or("<>".to_string()),
      filename,
      line,
      col,
    }
  }
}
```

The `peg::Parse` trait needs 3 functions:

- `start` to define the starting position of our input type, this is usually `0`
- `is_eof` which determines if a position is past the end of our input
- `position_repr` which returns an instance of the associated type given a position

```rust
impl<'source> peg::Parse for TokenStream<'source> {
  type PositionRepr = TokenLocation;

  fn start<'input>(&'input self) -> usize {
    0
  }

  fn is_eof<'input>(&'input self, pos: usize) -> bool {
    pos >= self.tokens.len()
  }

  fn position_repr<'input>(&'input self, pos: usize) -> Self::PositionRepr {
    let (token, linecol) = match self.tokens.get(pos) {
      Some((token, span)) => {
        (Some(token.clone()), self.linecol_lookup.get(span.start))
      },
      None => {
        (None, self.linecol_lookup.get(self.size))
      }
    };

    Self::PositionRepr {
      filename: self.filename.clone(),
      linecol,
      token,
    }
  }
}
```

The `peg::ParseElem` trait only needs 1 function:

- `parse_elem`: will return a token given a position

```rust
impl<'source, 'input> peg::ParseElem<'input> for TokenStream<'source> {
  type Element = &'input Token;

  fn parse_elem(&'input self, pos: usize) -> peg::RuleResult<Self::Element> {
    match self.tokens.get(pos) {
      Some((token, _)) => {
        peg::RuleResult::Matched(pos + 1, token)
      },
      None => {
        peg::RuleResult::Failed
      },
    }
  }
}
```

The `peg::ParseLiteral` trait only needs 1 function:

- `parse_string_literal`: will succeed if the token at the given position matches the given string literal

We will only make it succeed if the token at the given position is a `Token::Symbol` variant. Which means that string literals in our grammar rule will match symbols.

```rust
impl<'source> peg::ParseLiteral for TokenStream<'source> {
  fn parse_string_literal(&self, pos: usize, literal: &str) -> peg::RuleResult<()> {
    match self.tokens.get(pos) {
      Some((Token::Symbol(sym), _)) if sym == literal => {
        peg::RuleResult::Matched(pos + 1, ())
      },
      _ => {
        peg::RuleResult::Failed
      }
    }
  }
}
```

The `peg::ParseSlice` trait needs only 1 function:

- `parse_slice` : returns a slice of tokens given a begin/end positions

```rust
 impl<'source, 'input> peg::ParseSlice<'input> for TokenStream<'source> {
  type Slice = Vec<&'input Token>;

  fn parse_slice(&'input self, begin_pos: usize, end_pos: usize) -> Self::Slice {
    self.tokens[begin_pos .. end_pos]
      .into_iter()
      .map(|(token, _)| token)
      .collect()
  }
}
```

Now that the glue between our lexer and `peg` is done, we can begin our grammar:

```rust
use std::hash::Hash;
use crate::intern::{StringInterner, Backend, Symbol>};
use crate::lexer::{Token, TokenStream};
use crate::ast::{SExpression, Literal};

peg::parser!{
  pub grammar module_parser<'source>() for TokenStream<'source> {
    // ...
  }
}
```

Within the `peg::parser!` macro, we define our grammar for our input type. This will define a module of the same name, containing our generated parsers.

We will need the string interner as a parameter for our rules. Each rule will need to be generic over the Symbol type as well, because the produced AST is generic.

The syntax to define a rule is as follow:

```
rule rule_name<type params>(rule params) -> ReturnType  
  = pattern
  { rust code returning ReturnType }
```

We start by defining a public rule `module` which will serve as entrypoint. A module is a sequence of S-Expressions:

```rust
pub rule mpdule<S: Symbol, B: Backend<S>>(
  interner: &mut StringInterner<S>
) -> Vec<SExpression<S>>
  = exprs:s_expression(interner)+
  { exprs }
```

Here, we call another rule, `s_expression(interner)`, the `+` indicates that we should match this rule **one or more** times. We then capture the result in `exprs`. The corresponding Rust code simply return the captured result.

Let’s define the `s_expression` rule:

```rust
rule s_expression<S: Symbol, B: Backend<S>>(
  interner: &mut StringInterner<S>
) -> SExpression<S>
  = literal(interner)
  / list(interner)
```

This rule will first try to match a literal, and if it fails, it will try to match a list. We simply return the result of any matching option.

Let’s define the `literal` rule:

```rust
rule literal<S: Symbol, B: Backend<S>>(
  interner: &mut StringInterner<S>
) -> SExpression<S>
  = literal_boolean()
  / literal_integer()
  / literal_float()
  / literal_string()
  / literal_symbol(interner)
```

A literal is either a boolean, an integer, a float, a string or a symbol (which we will intern).

The `literal_boolean` rule is quite simple to implement:

```rust
rule literal_boolean<S: Symbol>() -> SExpression<S>
  = literal_boolean_true()
  / literal_boolean_false()

rule literal_boolean_true<S: Symbol>() -> SExpression<S>
  = [Token::True]
  { SExpression::Literal(Literal::Boolean(true)) }

rule literal_boolean_false<S: Symbol>() -> SExpression<S>
  = [Token::False]
  { SExpression::Literal(Literal::Boolean(false)) }
```

A boolean is either the token `Token::True` or the token `Token::False` and the result is therefore the `SExpression::Literal` variant, containing a `Literal::Boolean` variant.

Similarly, we can define the `literal_float` rule as:

```rust
rule literal_float<S: Symbol>() -> SExpression<S>
  = [Token::Float(n)]
  { SExpression::Literal(Literal::Float(*n)) }
```

A float is the token `Token::Float` and we capture its inner value in the reference `n`, which allows use to construct the `Literal::Float` variant.

The last rule, `literal_symbol` will use the string interner:

```rust
rule literal_symbol<S: Symbol, B: Backend<S>>(
  interner: &mut StringInterner<S>
) -> SExpression<S>
  = [Token::Symbol(s)]
  {
    let sym = interner.get_or_intern(s);
    SExpression::Literal(Literal::Symbol(sym))
  }
```

The final grammar looks like this:

```rust
use crate::intern::{StringInterner, Backend, Symbol};
use crate::lexer::{Token, TokenStream};
use crate::ast::{SExpression, Literal};

peg::parser!{
  pub grammar module_parser<'source>() for TokenStream<'source> {
    pub rule module<S: Symbol, B: Backend<S>>(
      interner: &mut StringInterner<B>
    ) -> Vec<SExpression<S>>
      = exprs:s_expression(interner)+
      { exprs }

    rule s_expression<S: Symbol, B: Backend<S>>(
      interner: &mut StringInterner<B>
    ) -> SExpression<S>
      = literal(interner)
      / list(interner)

    rule list<S: Symbol, B: Backend<S>>(
      interner: &mut StringInterner<B>
    ) -> SExpression<S>
      = [Token::ParenOpen] children:s_expression(interner)* [Token::ParenClose]
      { SExpression::List(children) }

    rule literal<S: Symbol, B: Backend<S>>(
      interner: &mut StringInterner<B>
    ) -> SExpression<S>
      = literal_boolean()
      / literal_integer()
      / literal_float()
      / literal_string()
      / literal_symbol(interner)

    rule literal_boolean<S: Symbol>() -> SExpression<S>
      = literal_boolean_true()
      / literal_boolean_false()

    rule literal_boolean_true<S: Symbol>() -> SExpression<S>
      = [Token::True]
      { SExpression::Literal(Literal::Boolean(true)) }

    rule literal_boolean_false<S: Symbol>() -> SExpression<S>
      = [Token::False]
      { SExpression::Literal(Literal::Boolean(false)) }

    rule literal_integer<S: Symbol>() -> SExpression<S>
      = literal_integer_2()
      / literal_integer_8()
      / literal_integer_10()
      / literal_integer_16()

    rule literal_integer_2<S: Symbol>() -> SExpression<S>
      = [Token::IntegerBase2(n)]
      { SExpression::Literal(Literal::Integer(*n)) }

    rule literal_integer_8<S: Symbol>() -> SExpression<S>
      = [Token::IntegerBase8(n)]
      { SExpression::Literal(Literal::Integer(*n)) }

    rule literal_integer_10<S: Symbol>() -> SExpression<S>
      = [Token::IntegerBase10(n)]
      { SExpression::Literal(Literal::Integer(*n)) }

    rule literal_integer_16<S: Symbol>() -> SExpression<S>
      = [Token::IntegerBase16(n)]
      { SExpression::Literal(Literal::Integer(*n)) }

    rule literal_float<S: Symbol>() -> SExpression<S>
      = [Token::Float(n)]
      { SExpression::Literal(Literal::Float(*n)) }

    rule literal_string<S: Symbol>() -> SExpression<S>
      = [Token::String(s)]
      { SExpression::Literal(Literal::String(s.clone())) }

    rule literal_symbol<S: Symbol, B: Backend<S>>(
      interner: &mut StringInterner<B>
    ) -> SExpression<S>
      = [Token::Symbol(s)]
      {
        let sym = interner.get_or_intern(s);
        SExpression::Literal(Literal::Symbol(sym))
      }
  }
}
```

Lastly, we write a helper function that takes our source code and a string interner as input, and returns our AST:

```rust
mod prelude;
mod intern;
mod lexer;
mod ast;
mod grammar;

use self::{
  prelude::*,
  intern::{StringInterner, Backend, Symbol},
};
pub use self::{
  prelude::SyntaxError,
  ast::{SExpression, Literal},
};

pub fn parse<S: Symbol, B: Backend<S>>(
  filename: Option<std::path::PathBuf>,
  input: &str,
  interner: &mut StringInterner<B>,
) -> Result<Vec<SExpression<S>>>
{
  let stream = lexer::TokenStream::new(
    filename,
    input
  )?;
  let ast = grammar::module_parser::module(&stream, interner)?;
  Ok(ast)
}
```

And this is it for the interpreter’s frontend.

---

## Part 2: Evaluating S-Expressions

In Lisp, data (booleans, numbers, strings, the empty list) evaluates to itself.

Symbols however, evaluates to another value (it might be a function, a list, or data).

A Form is a program that evaluates to a value, it is represented as a list of at least one element, its first element must be a symbol which evaluates to a function (a primitive, or a lambda), the rest of the list is the function’s arguments.

---

### Step 1: Runtime Type Information

For simplicity, we will do type-checking at runtime. It should be possible however to walk the AST and type-check it. The process would be identical to evaluation, but instead of returning values, we would return types.

This means that we need runtime type information, this can be done with a simple enum:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Type {
  Boolean,
  Integer,
  Float,
  String,
  Symbol,
  List,
  Function,
}

impl Type {
  // small utility function to generate a type error
  pub fn error(got: Type, expected: Type) -> RuntimeError {
    RuntimeError::TypeError {
      expected: format!("{:?}", expected),
      got: format!("{:?}", got)
    }
  }
}
```

---

### Step 2: Representing values

Every S-Expression will evaluate to a value, therefore, we need to define a type to represent every type of values:

```rust
#[derive(Clone)]
pub enum Value<S: Symbol> {
  Boolean(bool),
  Integer(i64),
  Float(f64),
  String(String),
  Symbol(Sym<S>),
  List(List<S>),
  Function(Function<S>),
}

impl<S: Symbol> Default for Value<S> {
  fn default() -> Self {
    Self::List(List::NIL)
  }
}

impl<S: Symbol> Value<S> {
  pub fn as_type(&self) -> Type {
    match self {
      Value::Boolean(..) => Type::Boolean,
      Value::Integer(..) => Type::Integer,
      Value::Float(..) => Type::Float,
      Value::String(..) => Type::String,
      Value::Symbol(..) => Type::Symbol,
      Value::List(..) => Type::List,
      Value::Function(..) => Type::Function,
    }
  }
}
```

We encapsulate the generic parameter `S` into a `Sym` structure to faciliate implementations of the `TryFrom` trait later on:

```rust
#[derive(Clone)]
pub struct Sym<S: Symbol>(S);

impl<S: Symbol> Sym<S> {
  pub fn as_symbol(&self) -> S {
    self.0
  }
}

impl<S: Symbol> From<S> for Sym<S> {
  fn from(sym: S) -> Self {
    Self(sym)
  }
}

impl<S: Symbol> From<&S> for Sym<S> {
  fn from(sym: &S) -> Self {
    Self(*sym)
  }
}
```

Our type-checking will rely on implementations of the `TryFrom` trait:

```rust
impl<S: Symbol> TryFrom<Value<S>> for bool {
  type Error = RuntimeError;

  fn try_from(value: Value<S>) -> std::result::Result<Self, Self::Error> {
    match value {
      Value::Boolean(val) => Ok(val),
      _ => Err(Type::error(value.as_type(), Type::Boolean)),
    }
  }
}

impl<S: Symbol> TryFrom<&Value<S>> for bool {
  type Error = RuntimeError;

  fn try_from(value: &Value<S>) -> std::result::Result<Self, Self::Error> {
    match value {
      Value::Boolean(val) => Ok(*val),
      _ => Err(Type::error(value.as_type(), Type::Boolean)),
    }
  }
}

// boilerplate code skipped
```

This will allow us to get the encapsulated data and early-return in case of type errors:

```rust
fn example<S: Symbol>(val: Value<S>) -> Result<()> {
  let b: bool = val.try_into()?;
  Ok(())
}
```

As of now, this type is incomplete. We need to define the `List` and `Function` types.

---

### Step 3: The List type

The `List` type is a linked list, one way to implement it in Rust is to use the `Rc<RefCell<T>>` type:

- `Rc<T>` is a reference counted pointer, the memory will be freed when the reference counter drops to 0
- `RefCell<T>` provides interior mutability, it’s basically borrow checking at runtime through the `RefCell::borrow()` and `RefCell::borrow_mut()` methods

The list cell type will contain the value of the current cell, and the pointer to the next cell in the list. And the list container type will contain an optional pointer to the head of the list (the first cell).  
If that pointer is `None` then the list is empty.

```rust
#[derive(Clone)]
pub struct ConsCell<S: Symbol> {
  pub car: Value<S>,
  pub cdr: Option<Rc<RefCell<ConsCell<S>>>>,
  // that's a lot of > 😅
}

#[derive(Clone)]
pub struct List<S: Symbol> {
  head: Option<Rc<RefCell<ConsCell<S>>>>,
}

impl<S: Symbol> List<S> {
  pub const NIL: Self = Self { head: None };

  pub fn empty(&self) -> bool {
    self.head.is_none()
  }

  pub fn car(&self) -> Result<Value<S>> {
    self.head
      .as_ref()
      .map(|rc| rc.borrow().car.clone())
      .ok_or_else(|| RuntimeError::NilValue {
        detail: "car called on empty list".to_string(),
      })
  }

  pub fn cdr(&self) -> Self {
    let head = self.head
      .as_ref()
      .and_then(|rc| rc.borrow().cdr.as_ref().cloned());

    Self { head }
  }

  pub fn cons(&self, val: Value<S>) -> Self {
    Self {
      head: Some(Rc::new(RefCell::new(ConsCell {
        car: val,
        cdr: self.head.clone()
      })))
    }
  }
}
```

The methods `car`, `cdr` and `cons` will help us implement the Lisp primitives of the same name.

We will also want to iterate over our lists at some point in the interpreter:

```rust
#[derive(Clone)]
pub struct ListIterator<S: Symbol>(Option<Rc<RefCell<ConsCell<S>>>>);

impl<'a, S: Symbol> IntoIterator for &'a List<S> {
  type Item = Value<S>;
  type IntoIter = ListIterator<S>;

  fn into_iter(self) -> Self::IntoIter {
    ListIterator(self.head.clone())
  }
}

impl<S: Symbol> Iterator for ListIterator<S> {
  type Item = Value<S>;

  fn next(&mut self) -> Option<Self::Item> {
    self.0.clone().map(|cons| {
      let val = cons.borrow().car.clone();
      self.0 = cons.borrow().cdr.clone();
      val
     })
  }
}
```

---

### Step 4: The Function type

We will have 2 kinds of functions:

- `NativeFn`: Rust functions exposed to the Lisp code, we will use this to implement the primitives (`cons`, `car`, `cdr`, `lambda`, etc…)
- `Lambda` : functions defined in Lisp code

```rust
#[derive(Clone)]
pub struct Lambda<S: Symbol> {
  pub params: Vec<S>,
  pub body: Box<Value<S>>,
}

pub type NativeFn<S> = fn(
  Rc<RefCell<Env<S>>>,
  Vec<Value<S>>,
) -> Result<Value<S>>;

#[derive(Clone)]
pub enum Function<S: Symbol> {
  NativeFn(NativeFn<S>),
  Lambda(Lambda<S>),
}
```

---

### Step 5: The Environment

The Lisp environment is the data structure that holds the definitions of the current scope. It has a mapping of symbols to values and an optional pointer to the parent scope:

```rust
pub struct Env<S: Symbol> {
  parent: Option<Rc<RefCell<Env<S>>>>,
  values: HashMap<S, Value<S>>,
}

impl<S: Symbol> Env<S> {
  pub fn new() -> Self {
    Self {
      parent: None,
      values: HashMap::new(),
    }
  }
}
```

When creating lambda functions, we will need to create a new scope from this one:

```rust
impl<S: Symbol> Env<S> {
  pub fn extend(parent: Rc<RefCell<Env<S>>>) -> Self {
    Self {
      parent: Some(parent),
      values: HashMap::new(),
    }
  }
}
```

Whenever we need to resolve a symbol’s value, we can use the environment’s `get` method:

```rust
impl<S: Symbol> Env<S> {
  pub fn get(&self, symbol: S) -> Option<Value<S>> {
    if let Some(val) = self.values.get(&symbol) {
      Some(val.clone())
    }
    else if let Some(parent) = &self.parent {
      parent.borrow().get(symbol)
    }
    else {
      None
    }
  }
}
```

The `def` Lisp primitive will assign a value to a symbol in the current scope, this is done using the environment’s `define` method:

```rust
impl<S: Symbol> Env<S> {
  pub fn define(&mut self, symbol: S, value: Value<S>) {
    self.values.insert(symbol, value);
  }
}
```

The `set!` Lisp primitive will reassign a value to a symbol in the scope it was defined, this is done using the environment’s `set` method:

```rust
impl<S: Symbol> Env<S> {
  pub fn set(&mut self, symbol: S, value: Value<S>) -> Result<(), S> {
    if self.values.contains_key(&symbol) {
      self.values.insert(symbol, value);
      Ok(())
    }
    else if let Some(parent) = &self.parent {
      parent.borrow_mut().set(symbol, value)
    }
    else {
      Err(symbol)
    }
  }
}
```

Finally, the `undef` Lisp primitive will unassign the value from a symbol in the scope it was defined, this is done using the environment’s `undefine` method:

```typescript
impl<S: Symbol> Env<S> {
  pub fn undefine(&mut self, symbol: S) {
    if self.values.contains_key(&symbol) {
      self.values.remove(&symbol);
    }
    else if let Some(parent) = &self.parent {
      parent.borrow_mut().undefine(symbol);
    }
  }
}
```

Finally, we create a helper function to populate the default environment with our operators and similar functions:

```rust
pub fn default_env<S, B>(interner: &mut StringInterner<B>) -> Env<S>
  where
    S: Symbol,
    B: Backend<S>
{
  let mut env = Env::new();

  env.define(
    interner.get_or_itern("+"),
    Value::Function(Function::NativeFn(primitives::arithmetic::iadd)),
  );
  // boilerplate code...

  env
}
```

And here is an example of primitive implementation:

```rust
mod primitives {
  pub fn iadd<S: Symbol>(
    _env: Rc<RefCell<Env<S>>>,
    args: Vec<Value<S>>,
  ) -> Result<Value<S>> {
    assert_at_least_args(2, args.len())?;

    let mut result = 0;

    for arg in args {
      let val: i64 = arg.try_into()?; // type checking
      result += val;
    }

    Ok(Value::Integer(result))
  }
}
```

---

### Step 6: The interpreter

The interpreter is what connects the dots between our AST and our Lisp values/environments. It is also the interpreter which will own the string interner:

```rust
pub struct Interpreter<S: Symbol, B: Backend<S>> {
  interner: StringInterner<B>,
  marker: std::marker::PhantomData<S>,
}

impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  pub fn new() -> Self {
    Self {
      interner: StringInterner::new(),
      marker: std::marker::PhantomData{},
    }
  }
}
```

We can then add 2 helper functions, to create the environment and format a value to a string:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  pub fn default_env(&mut self) -> Rc<RefCell<Env<S>>> {
    Rc::new(RefCell::new(default_env(&mut self.interner)))
  }

  pub fn format_value(&self, value: &Value<S>) -> String {
    match value {
      Value::Boolean(val) => format!("{}", val),
      Value::Integer(val) => format!("{}", val),
      Value::Float(val) => format!("{}", val),
      Value::String(val) => format!("{}", val),
      Value::Symbol(sym) => {
        self.interner.resolve(sym.as_symbol()).unwrap_or("<>").to_string()
      },
      Value::List(list) => {
        let repr = list
          .into_iter()
          .map(|value| self.format_value(&value))
          .collect::<Vec<String>>()
          .join(" ");

        format!("({})", repr)
      },
      Value::Function(func) => match func {
        Function::NativeFn(native_func) => {
          format!("[function {:p}]", native_func)
        },
        Function::Lambda(lambda) => {
          format!("[function {:p}]", lambda)
        },
      },
    }
  }
}
```

Because we need the string interner to resolve a symbol to a string, we can’t simply implement the `Display` trait on our `Value<S>` data structure. This method is actually the easiest way to get this behavior.

We now need to transform our AST (`SExpression<S>`) into a Lisp value:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  fn parse_sexpression(&self, sexpression: &SExpression<S>) -> Result<Value<S>> {
    match sexpression {
      SExpression::Literal(val) => {
        Ok(self.parse_literal(val))
      },
      SExpression::List(elements) => {
        let mut list = List::NIL;

        for element in elements.iter().rev() {
          let val = self.parse_sexpression(element)?;
          list = list.cons(val);
        }

        Ok(Value::List(list))
      }
    }
  }

  fn parse_literal(&self, literal: &Literal<S>) -> Value<S> {
    match literal {
      Literal::Boolean(val) => Value::Boolean(*val),
      Literal::Integer(val) => Value::Integer(*val),
      Literal::Float(val) => Value::Float(*val),
      Literal::String(val) => Value::String(val.clone()),
      Literal::Symbol(sym) => Value::Symbol(sym.into()),
    }
  }
}
```

The final part is the **eval** function, its purpose is simple: evaluate a Lisp program.

In Lisp, programs are a sequence of Lisp forms, which are either:

- data (booleans, numbers, strings, empty lists), which evaluates to itself
- symbols, which evaluates to a value from the environment
- lists where the first element is a function or a symbol which evaluates to a function, and the remaining elements are the arguments of the function

Let’s start:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  pub fn eval_expression(
    &mut self,
    env: Rc<RefCell<Env<S>>>,
    expression: Value<S>,
  ) -> Result<Value<S>> {
    match expression {
      Value::Symbol(sym) => {
        todo!("eval symbol")
      },
      Value::List(list) if !list.empty() => {
        todo!("eval function")
      },
      _ => {
        todo!("eval data")
      },
    }
  }
}
```

The `eval data` part is easy, we just return the expression itself:

```rust
_ => {
  Ok(expression)
},
```

For the `eval symbol` part, we need to get the value from the environment, or return an error if the symbol does not exist:

```rust
Value::Symbol(sym) => {
  let sym = sym.as_symbol();
  env.borrow().get(sym).ok_or_else(|| RuntimeError::UndefinedSymbol {
    detail: self.interner.resolve(sym).unwrap_or("<>").to_string()
  })
},
```

The `eval function` part is trickier because we have some builtin functions that are not part of the environment:

- `quote`: which returns the arguments without evaluating them
- `def`: which assign a value to a symbol in the environment
- `set!`: which re-assign a value to an existing symbol in the environment
- `if`: which evaluates a condition, and if that condition is true, it will evaluate the second argument (the `true` branch), if not it will evaluate the third argument (the `else` branch)
- `lambda`: which creates a new function
- `let`: which defines local variables (listed in the first argument) and evaluates the second argument using those local variables
- …

If the first element of the list is one of those symbols, we need to call the builtin function. If not, we need to evaluate that element and check that its type is a function. If it is, we need to evaluate all the arguments and call the function:

```rust
Value::List(list) if !list.empty() => {
  let func = list.car()?;
  let args: Vec<Value<S>> = list.cdr().into_iter().collect();

  if let Value::Symbol(sym) = &func {
    let sym = sym.as_symbol();
    // for simplicity, we resolve the symbol to a &str type
    // for performance, we should have a map from the symbol to the function
    let func_name = self.interner.resolve(sym).unwrap_or("<>");

    match func_name {
      "println" => return self.builtin_println(env.clone(), args),
      "quote" => return self.builtin_quote(args),
      "def" => return self.builtin_define(env.clone(), args),
      "set!" => return self.builtin_set(env.clone(), args),
      "if" => return self.builtin_if(env.clone(), args),
      "lambda" => return self.builtin_lambda(args),
      "let" => return self.builtin_let(env.clone(), args),
      _ => {}
    }
  }

  let func = self.eval_expression(env.clone(), func)?;
  let func: Function<S> = func.try_into()?;

  let mut eval_args = Vec::with_capacity(args.len());

  for arg in args {
    let arg = self.eval_expression(env.clone(), arg)?;
    eval_args.push(arg);
  }

  self.eval_function(env.clone(), func, eval_args)
},
```

The final part of our **eval** function is the `eval_function()` method. For native Rust functions, we simply need to call the function. For lambdas, we need to create a new scope, assign the arguments to the new scope, and evaluate the body of the lambda:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  fn eval_function(
    &mut self,
    env: Rc<RefCell<Env<S>>>,
    func: Function<S>,
    args: Vec<Value<S>>,
  ) -> Result<Value<S>> {
    match func {
      Function::NativeFn(func) => {
        func(env.clone(), args)
      },
      Function::Lambda(lambda) => {
        assert_exactly_args(lambda.params.len(), args.len())?;

        let mut env = Env::extend(env.clone());

        for (name, arg) in std::iter::zip(lambda.params, args) {
          env.define(name, arg);
        }

        self.eval_expression(
          Rc::new(RefCell::new(env)),
          lambda.body.as_ref().clone(),
        )
      },
    }
  }
}
```

As an example, here is the `builtin_lambda()` implementation:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  pub fn builtin_lambda(&mut self, args: Vec<Value<S>>) -> Result<Value<S>> {
    assert_exactly_args(2, args.len())?;

    let params = &args[0];
    let body = &args[1];

    let params: List<S> = params.try_into()?;
    let mut param_names = Vec::new();

    for param in params.into_iter() {
      let param_name: Sym<S> = param.try_into()?;
      param_names.push(param_name.as_symbol());
    }

    let lambda = Value::Function(Function::Lambda(Lambda {
      params: param_names,
      body: Box::new(body.clone()),
    }));

    Ok(lambda)
  }
}
```

One final touch is the ability to evaluate a string or a file:

```rust
impl<S: Symbol, B: Backend<S>> Interpreter<S, B> {
  pub fn eval_file<P: AsRef<std::path::Path>>(
    &mut self,
    env: Rc<RefCell<Env<S>>>,
    input_path: P,
  ) -> Result<Value<S>> {
    let input = std::fs::read_to_string(input_path)?;
    self.eval_string(env, &input)
  }

  pub fn eval_string(
    &mut self,
    env: Rc<RefCell<Env<S>>>,
    input: &str,
  ) -> Result<Value<S>> {
    // we call our parser
    let sexpressions = crate::frontend::parse(
      None,
      &input,
      &mut self.interner,
    )?;

    let mut last_result = Value::default();

    for sexpression in sexpressions.iter() {
      let expression = self.parse_sexpression(sexpression)?;
      last_result = self.eval_expression(env.clone(), expression)?;
    }

    Ok(last_result)
  }
}
```

---

## Part 3: The REPL

REPL stands for Read-Eval-Print Loop. Given a prompt, the user enters some code, which is parsed then evaluated, and the result is printed on the screen. Rinse, Repeat.

To implement the prompt, we will use the Rust crate `rustyline`, which is an implementation of Readline. It supports an history, auto-completion (which we will not cover here), etc…

{{< cta-link
  url="https://github.com/kkawakam/rustyline"
  title="GitHub - kkawakam/rustyline: Readline Implementation in Rust"
  description="Readline implementation in Rust that is based on Antirez' Linenoise Supported Platforms Unix (tested on FreeBSD, Linux…"
>}}

The code is straightforward:

- we instantiate the interpreter
- we write a loop which reads a line from the prompt, and call the interpreter `eval_string()` method

```rust
use rustyline::error::ReadlineError;
use rustyline::DefaultEditor;

use crate::intern::{backend::DefaultBackend, symbol::SymbolUsize};
use crate::interpreter::Interpreter;

type Symbol = SymbolUsize;
type Backend = DefaultBackend<Symbol>;

fn main() {
  let mut interpreter: Interpreter<Symbol, Backend> = Interpreter::new();
  let env = interpreter.default_env();

  let mut rl = DefaultEditor::new()?;

  loop {
    match rl.readline(">>> ") {
      Ok(line) => {
        if let Err(err) = rl.add_history_entry(line.as_str()) {
          eprintln!("ReadlineError: {:?}", err);
        }

        match interpreter.eval_string(env.clone(), &line) {
          Ok(value) => {
            println!("{}", interpreter.format_value(&value));
          },
          Err(err) => {
            eprintln!("{}", err);
          },
        }
      },
      Err(ReadlineError::Interrupted) => {},
      Err(ReadlineError::Eof) => {
        break;
      },
      Err(err) => {
        eprintln!("ReadlineError: {:?}", err);
      }
    }
  }
}
```

And voilà! We got a Lisp interpreter in Rust!

---

## Conclusion

Lisp is a fun language, and quite easy to implement (I can recommend reading this article as well: [Lisp in less than 200 lines of C](https://carld.github.io/2017/06/20/lisp-in-less-than-200-lines-of-c.html)).

The Rust ecosystem is also full of amazing crates for language designers, you’re almost never gonna need to reinvent the wheel.

The traits `From<T>`, `TryFrom<T>`, etc… are great, especially for runtime type checking like in this project.

Though, I have to admit, trying to write generic code can be a pain. In the end, trying to be generic over the Symbol/Backend types of the string interner brought us nothing but boilerplate. In programming, every choice you make is a tradeoff. Using generics and traits shifts the decision to the user of your code. This might make sense for a library (especially in the standard library, where you don’t want to be opinionated), but for an application like this? I’m not so sure.

I still have troubles with designing dependency injection properly in Rust. In languages like Python, or C#, I’d use a global singleton container which would contain the dependencies. But unless I work with `lazy_static!` and `Box<dyn Trait>`, there is no such thing. I need to improve on that, because I can’t count the number of times I wanted to have the string interner as a global object. Initially, the `builtin_*` functions were supposed to be values in the environment, but some of them (hello `println`) needed the string interner to resolve the symbols back to strings.

By the way, if you’re writing Python and wish to use dependency injection, I recommend this library:

{{< cta-link
  url="https://python-dependency-injector.ets-labs.org/"
  title="Dependency Injector - Dependency injection framework for Python - Dependency Injector 4.41.0…"
  description="Dependency Injector is a dependency injection framework for Python. It helps to maintain you application structure. It…"
>}}

---

**NB:** All the code present in this article is available in this repository:

{{< cta-link
  url="https://github.com/linkdd/lispers"
  title="GitHub - linkdd/lispers: Educational project: How to implement a Lisp interpreter in Rust?"
  description=""
>}}

Feel free to clap 👏 for this article to give me more visibility 🙂

If you read until here, thank you! And have a slice of chocolate cake:

![](img/image-5f79e41bf210119b.jpg)
