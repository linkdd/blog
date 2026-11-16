---
title: "That’s it! I’m making my own C++ package manager…"
date: "2024-01-25T21:13:27Z"
description: "A deadly simple package manager for your C/C++ projects, written in Rust."
tags: ["programming", "rust", "build-system", "cpp", "open-source"]
author: "David Delassus"
images: ["img/image-f18735b9ed5f7a4e.png"]
---

![](img/image-f18735b9ed5f7a4e.png)

<https://xkcd.com/927/>

---

{{< cta-link
  url="https://github.com/linkdd/shipp"
  title="GitHub - linkdd/shipp: Deadly simple package manager"
  description="Deadly simple package manager. Contribute to linkdd/shipp development by creating an account on GitHub."
>}}

Managing dependencies in C/C\+\+ projects is a pain. Each library seem to have its own build system, nothing is standardized even if most of them use CMake.

I usually solve this by vendoring the dependencies, using `git submodule` for example, and compile them statically.

But when you start to have transitive dependencies, and nested git submodules, I start looking like this:

![](img/image-77d8b1aec0cff96a.png)

If my project has a direct dependency to a library, and a transitive dependency to the same library, simply using `add_subdirectory` with CMake and nested submodules will pull in the dependency twice, and build it twice. What a waste of resource…

If a transitive dependency is updated, I need to update the whole tree of nested git submodules. What a pain…

On top of that, I need to wrap the dependencies’s build system in my own, to make sure everything goes smoothly. What a chore…

That’s it! I’m making my own package manager!

---

## The inspiration

I want it to be easy to use, like `cargo`.  
I want it to vendor in my direct and transitive dependencies, like `npm`.  
I want it to fetch and build a dependency only once.  
I want it to generate a distributable `tar.gz` archive, like `mix`.  
I want it to be portable.  
I don’t want to make a build system.

The last one is really important. There are already great build systems, make, CMake, Ninja, Meson, … They handle complex workflows, like conditional compilation, platform detection, etc… I’m not gonna reinvent that wheel.

Instead, this package manager will use the existing library’s build system and simply solve the dependency fetching/building problem.

---

## The design

Let’s call this package manager **Shipp**, because all I want is to ship my software 😉

To make your project a Shipp package, you will need to add a `shipp.json` file at the root of your Git repository:

```json
{
  "name": "mypackage",
  "version": "0.1.0",
  "scripts": {
    "build": "command to build",
    "install": "command to install"
  },
  "dependencies": [
    {
      "name": "libfoo",
      "url": "https://github.com/example/libfoo.git",
      "version": "v0.1.0"
    }
  ]
}
```

For example, here is Shipp’s manifest:

```bash
{
  "name": "shipp",
  "version": "0.1.0",
  "scripts": {
    "build": "cargo build --release",
    "install": "cargo install --path . --root $SHIPP_DIST_DIR"
  }
}
```

And that’s the first nice consequence of not being a build system: it is not limited to C or C\+\+, it can build Rust projects, or Zig, or Nim, or whatever you want.

Shipp provides 4 subcommands:

`shipp deps.get` **:**  
It will perform a `git clone` or `git pull` for each direct dependencies, and recursively for each transitive dependencies. They will all be checked out in the `.shipp/deps/<dep name>`folder.

`shipp deps.build` **:**  
Will build each dependencies (in the correct order) by calling the `scripts.build` command from the manifest, then it will install the dependency in the `.shipp/dist` folder.  
That folder will contain the following folders:

- `bin` : for executables
- `lib` : for static and/or shared libraries
- `include` : for C/C\+\+ headers
- `share` : for other resources

The path to the `.shipp/dist` folder will be given via the `$SHIPP_DIST_DIR` environment variable, allowing your build system to use it to fill the compiler flags `-Iincludedir` or `-Llibdir`.

`shipp build` **:**  
This will build your project and install it in the `.shipp/dist` folder, just like your dependencies.

`shipp dist` **:**  
This will create a `tar.gz` archive of the `.shipp/dist` folder that you can distribute.

---

## The roadmap

I have no specific plan, it could be great to support “features” for conditional compilation, or allow package sources other than Git. So far this just scratches an itch I had, but if you feel like this could become something great, I’d love any kind of contribution 🙂

{{< cta-link
  url="https://github.com/linkdd/shipp"
  title="GitHub - linkdd/shipp: Deadly simple package manager"
  description="Deadly simple package manager. Contribute to linkdd/shipp development by creating an account on GitHub."
>}}
