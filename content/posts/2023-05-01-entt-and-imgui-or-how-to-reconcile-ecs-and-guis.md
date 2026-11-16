---
title: "EnTT and ImGui, or how to reconcile ECS and GUIs?"
date: "2023-05-01T00:06:37Z"
description: "ECS and GUIs are not trivial to mix, in this article I’ll take inspiration from React to make it nicer."
tags: ["cpp", "programming", "ecs", "gamedev", "react"]
author: "David Delassus"
images: ["img/image-2c33fa8831ec9e6c.png"]
---

![](img/image-2c33fa8831ec9e6c.png)

---

## 📝 Reminder: What is an ECS?

**E**ntity **C**omponent **S**ystems are a pattern you’ll often see in game development. They are a “data oriented” way of organizing your code, by separating the data and the behaviors (game logic). EnTT is a C\+\+ header-only library which provides one implementation of that pattern:

{{< cta-link
  url="https://github.com/skypjack/entt"
  title="GitHub - skypjack/entt: Gaming meets modern C++ - a fast and reliable entity component system (ECS)…"
  description="EnTT has been a dream so far, we haven't found a single bug to date and it's super easy to work with -- Every EnTT User…"
>}}

I also wrote an article as part of my devlog series for my game “Warmonger Dynasty”:

{{< cta-link
  url="/posts/2023-01-08-a-short-introduction-to-entity-component-system-in-c-with-entt"
  title="A short introduction to Entity-Component-System in C++ with EnTT"
  description="Warmonger Dynasty Devlog #3: An ECS with EnTT"
>}}

If you’re interested here is the link to the series:

{{< cta-link
  url="/reading-lists/warmonger-dynasty"
  title="Warmonger Dynasty Devlogs"
  description=""
>}}

---

## 🖥️ Declarative GUI vs Immediate GUI

In the GUI landscape, we have basically 2 drastically different paradigms:

- **Declarative:** the developper declares the layout of the UI, and the framework will use that description to render it. Actions are usually handled via signals and slots. This is what frameworks like **Qt** or **GTK** are.
- **Immediate:** As soon as you call the function to display a button, it is rendered “immediately”. This approach is procedural, and you handle actions right there.

For the UI of my game, I have decided to use **dear ImGui** which is a truly great library with plenty of backends.

{{< cta-link
  url="https://github.com/ocornut/imgui"
  title="GitHub - ocornut/imgui: Dear ImGui: Bloat-free Graphical User interface for C++ with minimal…"
  description="\"Give someone state and they'll have a bug one day, but teach them how to represent state in two separate locations…"
>}}

There is also [Nuklear](https://github.com/Immediate-Mode-UI/Nuklear) in C, or [egui](https://egui.rs) in Rust.

As an example, here is what a declarative UI framework usually looks like:

```python
import tkinter as tk

class App:
  def __init__(self):
    self.counter = 0

    self.root = tk.Tk()
    self.root.title("Counter")
    
    self.counter_lbl = tk.Label(self.root, text=f"Counter: {self.counter}")
    self.counter_lbl.pack(pady=10)

    self.inc_btn = tk.Button(self.root, text="Click Me", command=self.clicked)
    self.inc_btn.pack()

  def clicked(self):
    self.counter += 1
    self.counter_lbl.config(text=f"Counter: {self.counter}")

  def run(self):
    self.root.mainloop()

if __name__ == "__main__":
  App().run()
```

The equivalent with an immediate UI framework:

```cpp
int main(void) {
  // init sdl, opengl, imgui

  int counter = 0;

  while (true) {
    // sdl events
    // imgui events

    if (ImGui::Begin("Counter")) {
      ImGui::Text("Counter: %d", counter);

      if (ImGui::Button("Click Me")) {
        counter++;
      }
    }
    ImGui::End();
    
    // render
  }

  // teardown sdl, opengl, imgui
  return 0;
}
```

In the declarative version, we define the layout of the UI once, and then it cannot be changed (actually it can, but it’s not really straightforward). We then give the button widget a callback to call, which will mutate the state and the UI.

In the immediate version, we create a window with `ImGui::Begin()`, it returns `true` if the window is not collapsed (this way, we can avoid rendering if it was). And the `ImGui::Button()` function returns `true` if it was clicked. Everything is rendered as soon as the ImGui functions are called.

---

## 🧩 The awkward integration of ECS and GUIs

Maybe this is just me, but I’ve found that ECS works great to model game logic and game data, but for GUIs? I usually don’t use the ECS. At best, I have a single system which runs once every frame and renders the GUI. It might iterate over some entities, or get some known entities directly to display information (for example: the list of units/buildings you have).

But I have no way of “declaring” my UI in a component and have a system render it. Especially with immediate GUI frameworks.

*Spoiler: I’ve found a way that I think is nice, and that’s why I’m writing this article.*

---

## 💡 Some inspiration from React

In React, and most reactive UI frameworks, there is a clear separation of data and layout. It is often said that:

> The User Interface is a function of the state.

Let’s take as example, a React component:

```javascript
const MyCounter = () => {
  const [value, setValue] = useState(0);

  return (
    <div>
      <p>Counter: {value}</p>
      <button onclick={() => setValue(value + 1)}>Click me</button>
    </div>
  )
}
```

Here, the `useState` hook will return a pair:

- a value
- a function to mutate the value (will hint React that a rerender is needed)

If the value did not exist, then a default one is returned (here: `0`).

---

## 🗺️ The plan

We will re-use those concepts to build some sort of “reactive” framework over ImGui (in fact, over any immediate-mode GUI library):

- a `hooks` class, which will hold the component’s state and provide the `use_state` / `use_effect` / `use_callback` hooks
- a `component` class, which will provide the `frame_begin` / `frame_update` / `frame_end` methods, and an abstract virtual method `render`

Before jumping into the implementation, it’s important to understand how does React’s hook work under the hood.

The following article covers it pretty well for the `useState` hook:

{{< cta-link
  url="https://rajatexplains.com/usestate-hook-behind-the-hood"
  title="Behind the hood implementation of useState react hook"
  description="Hooks are special functions which let you \" hook into\" different React features. State, being one of its features, is…"
>}}

Every time we call the `useState` function in a component, it increments an index. That index is used to identify the value and its setter function. The article provides a simplified implementation:

```javascript
let componentHooks = [];
let currentHookIndex = 0;

// How useState works inside React (simplified).
function useState(initialState) {
  let pair = componentHooks[currentHookIndex];
  if (pair) {
    // This is not the first render,
    // so the state pair already exists.
    // Return it and prepare for next Hook call.
    currentHookIndex++;
    return pair;
  }

  // This is the first time we're rendering,
  // so create a state pair and store it.
  pair = [initialState, setState];

  function setState(nextState) {
    // When the user requests a state change,
    // put the new value into the pair.
    pair[0] = nextState;
    updateDOM();
  }

  // Store the pair for future renders
  // and prepare for the next Hook call.
  componentHooks[currentHookIndex] = pair;
  currentHookIndex++;
  return pair;
}
```

Similarly, the `useCallback` hook will increment the `currentHookIndex` and **memoize** the callback function only if its dependencies have changed.

And finally, the `useEffect` hook will also increment the `currentHookIndex` and **call** the callback function only if its dependencies have changed. Optionally, the callback function can return a cleanup function that will be called when the component unmounts.

Since we have no concept of “(un)mounting components”, we will call the cleanup function when the dependencies change, before executing the effect.

For more informations about those hooks, I recommend consulting the official documentation:

{{< cta-link
  url="https://react.dev/reference/react"
  title="Built-in React Hooks - React"
  description="Hooks let you use different React features from your components. You can either use the built-in Hooks or combine them…"
>}}

---

## 🏗️ Implementation

To store our state data, we will use the `entt::registry::context` type, which is a generic container that can hold any types.

We will use the `entt::id_type` as our index, it will serve as a key to identify the state data in the container.

Finally, instead of mutating the state in the setter function, we will push the mutation to a queue that will be consumed at the end of the frame.

With that in mind, we can get started with the `hooks` class:

```cpp
#pragma once

#include <type_traits>
#include <concepts>

#include <functional>
#include <optional>
#include <vector>
#include <tuple>

#include <entt/entt.hpp>

namespace gui {
  class hooks {
    private:
      using mutation_type = std::function<void()>;

      entt::id_type m_current_index{0};
      entt::registry::context m_state;
      std::vector<mutation_type> m_mutation_queue;

      void reset() {
        m_current_index = 0;
      }

      void commit() {
        for (const auto &mutation : m_mutation_queue) {
          mutation();
        }

        m_mutation_queue.clear();
      }
  };
}
```

Now, we can implement the `use_state` hook:

```cpp
template <typename T>
using setter_fn_type = std::function<void(const T&)>;

template <typename T>
using state_pair_type = std::pair<const T&, setter_fn_type<T>>;

class hooks {
  // ...

  public:
    template <typename T>
    state_pair_type<T> use_state(const T& initial_value) {
      entt::id_type key = m_current_index++;

      // if the value does not exist yet, we create it
      if (!m_state.contains<T>(key)) {
        m_state.emplace_as<T>(key, initial_value);
      }

      // we get the current value
      const T& value = m_state.get<T>(key);
      // and generate a setter function
      auto update_fn = [this, key](const T& new_value) -> void {
        // the setter function simply push the mutation to the queue
        m_mutation_queue.push_back([this, key, new_value]() -> void {
          // we replace the value in the state data
          m_state.erase<T>(key);
          m_state.emplace_as<T>(key, new_value);
        });
      };

      // we return the pair
      return {value, update_fn};
    }
};
```

The next step is the `use_effect` hook:

```cpp
using cleanup_fn_type = std::function<void()>;
using effect_fn_type  = std::function<std::optional<cleanup_fn_type>()>;

class hooks {
  // ...

  public:
    template <typename... Deps>
    void use_effect(effect_fn_type effect_fn, std::tuple<Deps...> deps) {
      entt::id_type effect_key = m_current_index++;
      entt::id_type deps_key   = m_current_index++;

      // if the effect does not exist yet
      if (!m_state.contains<std::tuple<Deps...>>(deps_key)) {
        // we insert the current dependencies values in the state data
        m_state.emplace_as<std::tuple<Deps...>>(deps_key, deps);

        // we call the effect
        auto cleanup_fn = effect_fn();
        if (cleanup_fn) {
          // if the effect returned a cleanup function, we add it to the state
          if (m_state.contains<cleanup_fn_type>(effect_key)) {
            m_state.erase<cleanup_fn_type>(effect_key);
          }

          m_state.emplace_as<cleanup_fn_type>(effect_key, *cleanup_fn);
        }
      }
      else {
        const auto& old_deps = m_state.get<std::tuple<Deps...>>(deps_key);

        // the effect exists, so we check if the dependencies changed
        if (deps != old_deps) {
          // if they did, we call the cleanup function (if it exists)
          if (m_state.contains<cleanup_fn_type>(effect_key)) {
            m_state.get<cleanup_fn_type>(effect_key)();
            m_state.erase<cleanup_fn_type>(effect_key);
          }

          // we replace the dependencies
          m_state.erase<std::tuple<Deps...>>(deps_key);
          m_state.emplace_as<std::tuple<Deps...>>(deps_key, deps);

          // we call the effect
          auto cleanup_fn = effect_fn();
          if (cleanup_fn) {
            // and set the cleanup function if needed
            if (m_state.contains<cleanup_fn_type>(effect_key)) {
              m_state.erase<cleanup_fn_type>(effect_key);
            }

            m_state.emplace_as<cleanup_fn_type>(effect_key, *cleanup_fn);
          }
        }
      }
    }
};
```

The last hook we will implement is the `use_callback` hook:

```cpp
// we want to restrain the callback type to std::function<R(Args...)>
// but since we already have a pack expansion with the dependencies,
// we need a C++ concept
namespace details {
  template <typename T>
  struct is_callback_fn : std::false_type {};

  template <typename R, typename... Args>
  struct is_callback_fn<std::function<R(Args...)>> : std::true_type {};
};

template <typename T>
concept is_callback_fn = details::is_callback_fn<T>::value;

class hooks {
  // ...

  public:
    // here, the type parameter F must satisfy the concept we created earlier
    // this avoids the erroneous:
    // template <typename R, typename... Args, typename... Deps>
    template <is_callback_fn F, typename... Deps>
    F use_callback(F&& callback_fn, std::tuple<Deps...> deps) {
      entt::id_type cb_key   = m_current_index++;
      entt::id_type deps_key = m_current_index++;

      // if the callback does not exists yet
      if (!m_state.contains<std::tuple<Deps...>>(deps_key)) {
        // we insert the current dependencies into the state data
        m_state.emplace_as<std::tuple<Deps...>>(deps_key, deps);

        // and memoize the callback function
        if (m_state.contains<F>(cb_key)) {
          m_state.erase<F>(cb_key);
        }

        m_state.emplace_as<F>(cb_key, callback_fn);
      }
      else {
        const auto& old_deps = m_state.get<std::tuple<Deps...>>(deps_key);

        // if the callback exists and the dependencies changed
        if (deps != old_deps) {
          // we replace the dependencies into the state data
          m_state.erase<std::tuple<Deps...>>(deps_key);
          m_state.emplace_as<std::tuple<Deps...>>(deps_key, deps);

          // we memoize the new callback function
          if (m_state.contains<F>(cb_key)) {
            m_state.erase<F>(cb_key);
          }

          m_state.emplace_as<F>(cb_key, callback_fn);
        }
      }

      // we return the memoized callback
      return m_state.get<F>(cb_key);
    }
};
```

React provides other hooks, like `useRef`, `useContext`, etc… For now, I haven’t felt the need to use them, but those could be added later on.

Now, all we need, is the `component` class:

```cpp
class component {
  private:
    hooks m_hooks;

  public:
    virtual void render(entt::registry &registry, hooks &h) = 0;

    void frame_begin() {
      m_hooks.reset();
    }

    void frame_update(entt::registry &registry) {
      render(registry, m_hooks);
    }

    void frame_end() {
      m_hooks.commit();
    }
};
```

All we need to do, is call the `frame_begin` / `frame_update` / `frame_end` methods at the right time in our main loop, and voilà!

---

## ✨ How to use it?

Let’s make a simple counter widget, it will display the number of times we clicked on a button.

To do this, we need to inherit from the `component` class, and implement the `render` method:

```cpp
#include "gui-framework.hpp"
#include <imgui.h>
#include <iostream>

class counter final : public gui::component {
  public:
    virtual void render(entt::registry &registry, gui::hooks &h) override {
      // example of state management
      auto [count, set_count] = h.use_state<int>(0);

      // example of callback
      auto click = h.use_callback<std::function<void()>, int>(
        [&]() {
          set_count(count + 1);
        },
        { count }
      );

      // this will be called whenever the "count" changed
      h.use_effect<int>(
        [&]() {
          std::cout << "clicked: " << count << std::endl;
          return std::nullopt;
        },
        { count }
      );

      // we do our rendering with ImGui (or anything else)
      ImGui::Text("Count: %d", count);

      if (ImGui::Button("Click")) {
        click();
      }
    }
};
```

From here, I can add `std::unique_ptr<gui::component>` instances as a component to any entities, then have a single system which will iterate over those entities to render the GUI.

---

## Conclusion

Up until now, my UI code was a bit of a mess. The “immediate mode GUI” pattern is very much “imperative”, and I find it easier to reason about in a “functional” way (this might just be me).

Taking inspiration from the architecture of React was a huge step towards clean/readable UI code for me. I can split my UI into functions, put the different parts of my UI into components for my ECS, allowing me to add/remove parts of the UI dynamically quite easily:

- I clicked on a unit, let’s create an entity with the “unit display” component
- I deselected the unit, let’s remove that entity

I love it.

**NB:** This might very well be a case of “over-engineering”, but it was fun!

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 6 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for my game **Warmonger Dynasty** is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here →

{{< cta-link
  url="/reading-lists/warmonger-dynasty"
  title="Warmonger Dynasty Devlogs"
  description=""
>}}
