---
title: "Yet another static site generator, but for language designers"
date: "2024-01-02T11:42:24Z"
description: "You are a language designer, you are working on your own programming language, maybe because you had an innovative idea, or maybe you want…"
tags: ["letlang", "static-site-generator", "programming", "plt", "web"]
author: "David Delassus"
images: ["img/image-301684663c46ceca.png"]
---

![](img/image-301684663c46ceca.png)

---

You are a language designer, you are working on your own programming language, maybe because you had an innovative idea, or maybe you want to create a monstrosity (hello [INTERCAL](https://en.wikipedia.org/wiki/INTERCAL)).

Either way, you want to show it to the world, so you start making a website. If you are like me, you’ll go for a static website generator, and [there](https://gohugo.io/) [are](https://jekyllrb.com/) a [ton](https://www.mkdocs.org/) of [them](https://github.com/rust-lang/mdBook)…

For [Letlang](https://letlang.dev), I chose Hugo.

{{< cta-link
  url="https://gohugo.io"
  title="The world's fastest framework for building websites"
  description="The world's fastest framework for building websites"
>}}

It’s nice, I can write the content using Markdown, design a nice HTML template, host it on [Netlify](https://www.netlify.com) for free, really great.

---

## “Do It Yourself” creeps in…

I have one issue with Hugo though. Its syntax highlighting engine, [Chroma](https://github.com/alecthomas/chroma), does not support my language. Why would it?

To extend it, I would need to create a [Pygments](https://pygments.org/) lexer, recompile Chroma, and build Hugo with my modified Chroma.

![](img/image-453351f2cb3fa1f5.jpg)

I already have made 3 grammars for my language:

- using [rust-peg](https://docs.rs/peg/latest/peg/) in my compiler
- using BNF in the documentation (I need to update it)
- a TexMate one for VS Code syntax highlighting

Now, I need a 4th one? Nope, nope, nope…

Then, I found Shiki:

{{< cta-link
  url="https://github.com/shikijs/shiki"
  title="GitHub - shikijs/shiki: A beautiful Syntax Highlighter."
  description="A beautiful Syntax Highlighter. Contribute to shikijs/shiki development by creating an account on GitHub."
>}}

It works in the browser, it supports custom TexMate grammars, and even multiple themes (I really like monokai).

Problem solved, right?

---

## I said, “Do It Yourself”

It’s a shame to have a static website generator, and then on the client-side having to download a syntax highlighting engine just to add colors to some code blocks.

This also mean that on slow connections, there is a time where the code blocks are not highlighted.

Here comes **Gin**:

{{< cta-link
  url="https://github.com/linkdd/gin"
  title="GitHub - linkdd/gin: Opinionated static website generator"
  description="Opinionated static website generator. Contribute to linkdd/gin development by creating an account on GitHub."
>}}

This static generator is inspired by Hugo (markdown content, HTML templates), but it uses Shiki as its syntax highlighting engine, allowing me to give it my TexMate grammar and highlight the code **at compile time**.

---

## Battling with procrastination

![](img/image-d77628ad1458efab.png)

This static generator was made 6 months ago. But I did not write the documentation. I thought I got all the time in the world, and suddenly, out of nowhere, **IT’S 2024!** (happy new year by the way 🎉).

Well, this year, even if it’s not finished, I’ll publish my work. As you can see, the documentation on the **Gin** website ( <https://linkdd.github.io/gin/> ) is very limited, but that’s okay. It’s better out there than rotting on my computer.

---

## As a bonus, a plugin!

I wanted **Gin** to be extensible, so that I can add plugins for every feature I did not imagine yet.

The plugin system allows me to create new custom Markdown blocks, and I have one plugin in store (aka: rotting on my computer for a few months) that will soon land as well: **Grammkit**.

If you are a language designer like me, you might want to use Grammkit:

{{< cta-link
  url="https://github.com/dundalek/GrammKit"
  title="GitHub - dundalek/GrammKit: Generate diagrams for parser grammars"
  description="Generate diagrams for parser grammars. Contribute to dundalek/GrammKit development by creating an account on GitHub."
>}}

**TL;DR:** It turns a BNF grammar into a railroad diagram:

```plaintext
start = left ("+" / "-") right
```

![](img/image-03d9aaebba3444ed.png)

Which is great for documenting the syntax of your language!

Now imagine having this in your Markdown document:

```markdown
# Introduction

The syntax is:

::: grammar
start = left ("+" / "-") right
:::
```

And have it automatically render the railroad diagram **at compile time**. 🙂

---

## Conclusion

Now that I published my static site generator, I am able to work on updating the Letlang website, a lot has happened since the last few months, but I have not pushed a single commit (damn you procrastination!), a lot of work could be lost if my computer burst into flames.

Stay tuned for some more news!
