---
title: "A very simple, yet versatile, game loop in C++ with EnTT"
date: "2023-02-01T14:15:38Z"
description: "Warmonger Dynasty Devlog #5: The game loop and scene system"
tags: ["gamedev", "programming", "cpp", "devlog"]
author: "David Delassus"
images: ["img/image-1150019ca9e56d84.png"]
---

![](img/image-1150019ca9e56d84.png)

---

This is the 5th devlog of Warmonger Dynasty, a 4X turn-based strategy game made in C\+\+/SDL. If you want to read the previous entries, take a look at my [reading list](/reading-lists/warmonger-dynasty).

When writing a game from scratch, the first step is to write the game loop. This is the part where you define what code should be run every frame, what should be run before and after the loop, and how/when you exit the loop (and therefore, the game).

---

## Game Loop 101: Starting with the basics

A typical SDL application has usually 3 parts:

- **initializion:** setting up the SDL, loading assets, etc…
- **event loop:** process SDL events, update your game state, rendering
- **teardown:** free assets, teardown SDL, etc…

It may look like this (error handling is omitted for clarity):

```cpp
int main(void) {
  // initialization
  SDL_Init(SDL_INIT_EVERYTHING);

  SDL_Window *win = SDL_CreateWindow(
    "my game",
    SDL_WINDOWPOS_UNDEFINED, SDL_WINDOW_POS_UNDEFINED,
    800, 600,
    SDL_WINDOW_SHOWN
  );

  SDL_Renderer *renderer = SDL_CreateRenderer(
    win,
    -1,
    SDL_RENDERER_PRESENTVSYNC | SDL_RENDERER_ACCELERATED | SDL_RENDERER_TARGETTEXTURE,
  );
  
  // event loop
  bool running = true;

  while (running) {
    SDL_Event evt;

    while (SDL_PollEvent(&evt) != 0) {
      switch (evt.type) {
        case SDL_QUIT:
          running = false;
          break;

        // handle other events
      }
    }

    // update game state

    SDL_RenderClear(renderer);
    // render game graphics
    SDL_RenderPresent(renderer);
  }

  // teardown
  SDL_DestroyRenderer(renderer);
  SDL_DestroyWindow(win);
  SDL_Quit();

  return 0;
}
```

Let’s create a simple abstraction so that we can easily hook things up later on.

As you may have read in the previous devlogs, I’m using the ECS pattern with the EnTT framework.

{{< cta-link
  url="https://github.com/skypjack/entt"
  title="GitHub - skypjack/entt: Gaming meets modern C++ - a fast and reliable entity component system (ECS)…"
  description="EnTT has been a dream so far, we haven't found a single bug to date and it's super easy to work with -- Every EnTT User…"
>}}

Therefore, the abstraction needs an EnTT registry, a list of systems to execute sequentially, and a list of hook callbacks to call at the right time.

But what is a system? From EnTT’s point of view, it’s any function that calls `registry.view<T...>()` and iterate over the result. From our point of view, it will be any function called once per frame.

For this purpose, we will define a `system` interface to facilitate things for later:

```cpp
class system {
  public:
    virtual void run(entt::registry &registry) = 0;
};
```

Now, what is a hook? In our game loop, we have 4 of them:

- `setup`: called before the event loop
- `frame_begin`: called at the beginning of every frame
- `frame_end`: called at the end of every frame
- `teardown`: called after the event loop

Each hook will have a list of `std::function` to call when triggered, so let’s define a class that will do just that:

```cpp
template <typename Type>
class hook;

template <typename Ret, typename ...Args>
class hook<Ret(Args...)> {
  private:
    using callback_type = std::function<Ret(Args...)>;

    std::vector<callback_type> m_callbacks;

  public:
    void connect(callback_type callback, bool front = false) {
      if (front) {
        m_callbacks.insert(m_callbacks.begin(), callback);
      }
      else {
        m_callbacks.push_back(callback);
      }

    void publish(Args... args) {
      for (auto callback : m_callbacks) {
        callback(args...);
      }
    }
};
```

Which can be used like this:

```cpp
hook<void(entt::registry &)> hook_setup;

hook_setup.connect([](entt::registry &registry) {
  // do stuff
});
hook_setup.connect([&](entt::registry &registry) {
  // do stuff
});

entt::registry registry;
hook_setup.publish(registry);
```

**NB:** While EnTT provides events and signals, it does not support all lambdas to avoid the allocation needed by `std::function`, but in our case it is simpler to rely on `std::function` because we want capturing lambdas later on.

One last detail, instead of using a boolean to control when the event loop should stop, I prefer to use an enumeration:

```cpp
enum class controlflow {
  exit,
  loop
};
```

With all of those building blocks, we can create our game loop abstraction:

```cpp
class game_loop {
  private:
    entt::registry m_registry;
    controlflow m_controlflow;
    std::vector<std::shared_ptr<system>> m_systems;
    std::vector<std::shared_ptr<system>> m_systems_last; // very dumb way of specifying when a system should execute

    hook<void(entt::registry &> m_hook_setup;
    hook<void(entt::registry &> m_hook_frame_begin;
    hook<void(entt::registry &> m_hook_frame_end;
    hook<void(entt::registry &> m_hook_teardown;

  public:
    game_loop &add_system(std::shared_ptr<system> system) {
      m_systems.push_back(system);
      return *this;
    }

    game_loop &add_system_last(std::shared_ptr<system> system) {
      m_systems_last.push_back(system);
      return *this;
    }

    game_loop &add_setup_callback(std::function<void(entt::registry &> callback) {
      m_hook_setup.connect(callback);
      return *this;
    }

    game_loop &add_frame_begin_callback(std::function<void(entt::registry &> callback) {
      m_hook_frame_begin.connect(callback);
      return *this;
    }

    game_loop &add_frame_end_callback(std::function<void(entt::registry &> callback) {
      m_hook_frame_end.connect(callback);
      return *this;
    }

    game_loop &add_teardown_callback(std::function<void(entt::registry &> callback) {
      m_hook_teardown.connect(callback, /* front = */ true);
      return *this;
    }

    void run() {
      m_controlflow = controlflow::loop;
      m_registry.ctx().emplace<controlflow &>(m_controlflow);

      m_hook_steup.publish(m_registry);

      while (m_controlflow == controlflow::loop) {
        m_hook_frame_begin.publish(m_registry);

        for (auto &sys : m_systems) {
          sys->run(m_registry);
        }

        // iterate in reverse
        for (
          auto it = m_systems_last.rbegin();
          it != m_systems_last.rend();
          ++it
        ) {
          auto &sys = *it;
          sys->run(m_registry);
        }
      }

      m_hook_teardown.publish(m_registry);
    }
};
```

Let’s dig into this code.

First, the `add_*` methods return a reference to the game loop itself, this allows chaining the methods. This is inspired by the Builder pattern.

Secondly, I add to the registry’s context a reference to the `controlflow` member variable, which gives the ability to any system, or hook, to exit the loop by getting the reference and setting it to `controlflow::exit`.

Then, I have 2 lists of systems, one to be run before the other. This is a dumb way of allowing “I want to run this system last” (useful for the render system for example). The second list is iterated in reverse so the first one in the list is actually the last to run. I’m not satisfied with the naming nor the model, but it works.

Lastly, I use shared pointers for the systems because I need to avoid once again the slicing problem (to call the correct `run` method) and I may add some to the registry’s context (like the scene system, or the soundtrack system), but maybe a reference to a unique pointer could work as well?

Now, my `main` function looks like this (error handling omitted for clarity):

```cpp
class event_system final : system {
  public:
    virtual void run(entt::registry &registry) {
      auto &cf = registry.ctx().get<controlflow &>();

      SDL_Event evt;

      while (SDL_PollEvent(&evt) != 0) {
        switch (evt.type) {
          case SDL_QUIT:
            cf = controlflow::exit;
            break;

          // ...
        }
      }
    }
};

class render_system final : system {
  public:
    virtual void run(entt::registry &registry) {
      SDL_Renderer *renderer = registry.ctx().get<SDL_Renderer *>();

      SDL_RenderClear(renderer);
      // ...
      SDL_RenderPresent(renderer);
    }
};

int main(void) {
  game_loop()
    .add_setup_callback([](entt::registry &registry) {
      SDL_Init(SDL_INIT_EVERYTHING);

      SDL_Window *win = SDL_CreateWindow(/* ... */);
      SDL_Renderer *renderer = SDL_CreateRenderer(/* ... */);

      registry.ctx().emplace<SDL_Window *>(win);
      registry.ctx().emplace<SDL_Renderer *>(renderer);
    })
    .add_teardown_callback([](entt::registry &registry) {
      SDL_Renderer *renderer = registry.ctx().get<SDL_Renderer *>();
      SDL_Window *win = registry.ctx().get<SDL_Window *>();

      SDL_DestroyRenderer(renderer);
      SDL_DestroyWindow(win);
      SDL_Quit();
    })
    .add_system(std::make_shared<event_system>())
    .add_system_last(std::make_shared<render_system>())
    .run();

  return 0;
}
```

This API might remind you of [Bevy](https://bevyengine.org/), a Rust game framework. This is normal, I was inspired by it 🙂

---

## Splitting the code

As you can see, while it is easy to hook new systems, new callbacks, etc… It stills make a quite big main function.

To solve that problem, I introduce one more interface:

```cpp
/* forward */ class game_loop;

class plugin {
  public:
    virtual void mount(game_loop &game_loop) = 0;
};

class game_loop {
  // ...

  public:
    game_loop &add_plugin(plugin &plugin) {
      plugin.mount(*this);
      return *this;
    }

  // ...
}
```

With this interface, I can put all my SDL related code into a single plugin. The same applies to all the ImGui related code. My homemade engine finally provides one plugin to set it all up:

```rust
class base_plugin final : public plugin {
  public:
    struct config {
      std::string win_title;
      Uint32 cap_fps;
    };

  private:
    sdl_plugin m_sdl;
    imgui_plugin m_imgui;
    std::shared_ptr<input::manager> m_input_manager;
    std::shared_ptr<messaging::bus> m_message_bus;

    std::shared_ptr<sys::debug_system> m_debug_system;

  public:
    base_plugin(config &config)
      : m_sdl(config.win_title, config.cap_fps),
        m_input_manager(std::make_shared<input::manager>()),
        m_message_bus(std::make_shared<messaging::bus>()),
        m_debug_system(std::make_shared<debug_system>())
    {}

    virtual void mount(game_loop &game_loop) {
      game_loop
        .add_plugin(m_sdl)
        .add_plugin(m_imgui)
        .add_setup_handler([&](entt::registry &registry) {
          registry.ctx().emplace<std::shared_ptr<input::manager>>(m_input_manager);
          registry.ctx().emplace<std::shared_ptr<messaging::bus>>(m_message_bus);
          registry.ctx().emplace<std::shared_ptr<sys::debug_system>>(m_debug_system);

          m_debug_system->register_component<components::transform>("Transform");
          m_debug_system->register_component<components::camera>("Camera");
          m_debug_system->register_component<components::background>("Background");
          m_debug_system->register_component<components::sprite>("Sprite");
        })
        .add_frame_begin_handler([&](entt::registry &registry) {
          m_input_manager->frame_begin();
        })
        .add_system(m_debug_system)
        .add_system(std::make_shared<event_system>())
        .add_system(std::make_shared<input_system>())
        .add_system_last(std::make_shared<messaging_system>())
        .add_system_last(std::make_shared<render_system>());
    }
};
```

After putting all my game-specific systems and callbacks in a plugin as well, my `main` function now looks like this:

```cpp
int main(void) {
  base_plugin::config config = {
    .win_title = "Warmonger Dynasty",
    .cap_fps = 60
  };
  base_plugin base(config);
  game_plugin game;

  game_loop()
    .add_plugin(base)
    .add_plugin(game)
    .run();

  return 0;
}
```

---

## A simple scene system

This part is a little bonus. When experimenting with Unity, one feature I really enjoyed was the Scene mechanism. It allows you to isolate entire levels and easily go from one screen to another. I’d usually have a scene for the main menu, a scene for the lobby, a scene for in-game, etc…

In Godot, you also have Scenes, in Game Maker Studio 2 you have “rooms”, each general purpose engine have some concept of it.

Therefore, let’s implement this feature as well. I want to be able to run code when entering or exiting the scene (to create and destroy entities), and for each frame (to render the GUI with ImGui).

The following interface should do the trick nicely:

```cpp
class scene {
  public:
    virtual void on_enter(entt::registry &registry) = 0;
    virtual void on_update(entt::registry &registry) = 0;
    virtual void on_exit(entt::registry &registry) = 0;
};
```

Which I would implement as follows:

```cpp
class ingame final : public scene {
  private:
    std::vector<entt::entity> m_entities;
    // ...

  public:
    virtual void on_enter(entt::registry &registry) {
      auto entity = registry.create();
      // add components to entity
      m_entities.push_back(entity);
    }

    virtual void on_update(entt::registry &registry) {
      auto &add_ui_command = registry.ctx().get<imgui_plugin::ui_command_request>();
      add_ui_command([&](entt::registry &registry) {
        // render ImGui interface
      });
    }

    virtual void on_exit(entt::registry &registry) {
      registry.destroy(m_entities.begin(), m_entities.end());
      m_entities.clear();
    }
};
```

Then, we implement a new system which holds the current scene and provide a method to switch to a new scene:

```cpp
class scene_system final : public system {
  private:
    std::unique_ptr<scene> m_current_scene{nullptr};

  public:
    virtual void run(entt::registry &registry) {
      if (m_current_scene != nullptr) {
        m_current_scene->on_update(registry);
      }
    }

    void change_scene(ent::registry &registry, std::unique_ptr<scene> new_scene) {
      if (m_current_scene != nullptr) {
        m_current_scene->on_exit(registry);
      }

      m_current_scene = std::move(new_scene);

      if (m_current_scene != nullptr) {
        m_current_scene->on_enter(registry);
      }
    }
};
```

The `on_enter` / `on_exit` methods are called as soon as we request a scene change. The `on_update` of the new scene will be called on the next frame.

In our plugin, we add the following:

```cpp
class game_plugin {
  private:
    std::shared_ptr<scene_system> m_scene_manager{
      std::make_shared<scene_system>()
    };

  public:
    void mount(game_loop &game_loop) {
      game_loop
        // ...
        .add_setup_callback([&](entt::registry &registry) {
          registry.ctx().emplace<std::shared_ptr<scene_system>>(m_scene_manager);
          m_scene_manager->change_scene(registry, std::make_unique<scene_menu_main>());
        })
        .add_teardown_callback([&](entt::registry &registry) {
          m_scene_manager->change_scene(registry, nullptr);
        })
        .add_system(m_scene_manager);
    }
};
```

This will add to the registry’s context a reference to our scene manager, so that we can call the `change_scene` method. During shutdown, we set the current scene to a null pointer so that the `on_exit` method of the current scene is called. Finally, we add the system to the game loop.

---

Using this scene system, I created 4 scenes for the game:

- the main menu, from which we will create a new game, join an existing game, or edit the game settings
- a server creation scene, which displays a loading screen while the game server is starting
- a lobby scene, which all clients eventually join
- an in-game scene, which all clients enters once the game starts

Which gives us:

![](img/image-74bca6d6ac23feef.png)

Main Menu scene

![](img/image-6c237cdb68b10a63.png)

Lobby scene

![](img/image-198af2a79c4cf2c7.png)

InGame scene

Yes, it’s pretty crude for now, but it will get better I promise 🙂

---

## Conclusion

Having solid foundations is important to be able to iterate quickly. Thanks to my previous experience with other game engines, especially Unity and Bevy (Rust), and my experience in home-made game engines which will never get used or produce a single game, I managed to get to a prototype quite quickly.

The key was to understand that I did not need a general purpose game engine and that a very thin wrapper over my dependencies was good enough. For example, my current engine does not handles collisions or any physics. It also does not need to, 4X strategy games are not physical simulations. In fact, at the moment, my engine does not even support animations (I don’t have any, yet?).

I’ve got the core features down:

- an asset pipeline ([devlog #1](/posts/2022-12-25-im-embedding-my-game-assets-into-the-executable-here-s-how-i-did-it))
- networking/multiplayer ([devlog #2](/posts/2023-01-02-multiplayer-multithreading-and-an-actor-model-in-c))
- an ECS with EnTT ([devlog #3](/posts/2023-01-08-a-short-introduction-to-entity-component-system-in-c-with-entt))
- an input system ([devlog #4](/posts/2023-01-25-decoupling-input-bindings-from-game-systems-with-c-sdl))
- a GUI framework (with [dear ImGui](https://github.com/ocornut/imgui))
- a rendering system (which supports cameras and shaders)

I’m currently working on map generation, and this is quite the rabbit hole as well (hello perlin noise, wave function collapse, etc…). So there may be some time before the next devlog.

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 3 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for the game is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here → [Devlogs Reading List](/reading-lists/warmonger-dynasty)
