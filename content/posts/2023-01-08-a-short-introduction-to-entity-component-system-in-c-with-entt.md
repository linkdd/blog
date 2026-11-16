---
title: "A short introduction to Entity-Component-System in C++ with EnTT"
date: "2023-01-08T18:09:46Z"
description: "Warmonger Dynasty Devlog #3: An ECS with EnTT"
tags: ["gamedev", "programming", "ecs", "cpp"]
author: "David Delassus"
images: ["img/image-bacf751586708ac4.png"]
---

![](img/image-bacf751586708ac4.png)

This is the third devlog in my series about my game **Warmonger Dynasty** (a 4X turn-based strategy game). If you want to read the previous entries, take a look at my [reading list](/reading-lists/warmonger-dynasty).

Today, we are gonna talk about Entity-Component-Systems in C\+\+ (ECS for friends), and especially about one great **header-only** library simply called **EnTT**.

{{< cta-link
  url="https://github.com/skypjack/entt"
  title="GitHub - skypjack/entt: Gaming meets modern C++ - a fast and reliable entity component system (ECS)…"
  description="EnTT has been a dream so far, we haven't found a single bug to date and it's super easy to work with -- Every EnTT User…"
>}}

---

## What is an ECS?

The concept was born in a Computer Science field called “Data Oriented Design”. To understand it better, it’s best to look at alternatives first.

In every game, we can clearly identify different parts which will have their own logic, for example:

- the character
- the enemies
- items and/or bonuses
- …

In Unity, we call those “Game Objects”. In Unreal Engine, we call those “Actors”. In Godot, we call those “Scenes”.

They usually gather data like health, strength, position, velocity, … and logic like “move”, “attack”, …

In an Object-Oriented fashion, we could design those like this:

```cpp
class character : public game_object {
  private:
    int health;
    int strength;
    vec2 position;
    vec2 velocity;

  public:
    virtual void on_update() {
      // check input
      // move character
      // attack
      // ...
    }

    virtual void on_draw() {
      // draw sprite, or animation, or ...
    }
};
```

Then, in our game loop, we would iterate throuh all our game objects every frame:

```cpp
// gather events and inputs

// update game state:
for (auto &game_object : game_objects) {
  game_object.on_update();
}

// clear screen, prepare rendering, ...

// draw game objects:
for (auto &game_object : game_objects) {
  game_object.on_draw();
}
```

This can work well for small code-bases, but soon enough (earlier than you think), copy/pasting code becomes a problem. So we might split game objects into components (Unity calls them “MonoBehaviours”):

```cpp
class transform_component : public component {
  private:
    vec2 position;

  public:
    // ...
};

class movement_component : public component {
  private:
    vec2 velocity;

  public:
    virtual void on_update() {
      auto &transform = this->get_component<transform_component>();

      // check input
      // update transform's position
    }
};

class character_component : public component {
  private:
    int health;
    int strength;

  public:
    virtual void on_update() {
      // check input
      // attack and/or take damage
    }

    virtual void on_draw() {
      // render sprite or animation or ...
    }
};

std::shared_ptr<game_object> make_character() {
  auto game_object = std::make_shared<game_object>();
  game_object->add_component<transform_component>(/* ... */);
  game_object->add_component<movement_component>(/* ... */);
  game_object->add_component<character_component>(/* ... */);
  return game_object;
}
```

Then, our game loop would look like this:

```cpp
// gather events and inputs

// update game state:
for (auto &game_object : game_objects) {
  for (auto &component : game_object.iter_components()) {
    component.on_update();
  }
}

// clear screen, prepare rendering, ...

// draw game objects:
for (auto &game_object : game_objects) {
  for (auto &component : game_object.iter_components()) {
    component.on_draw();
  }
}
```

This also works well, and it scales better with the codebase because you can reuse components on multiple different game objects.

One problem that can arise is performance, and that’s the problem “Data Oriented Design” tackles. Your game logic manipulates data, and that data has to be stored somewhere. The closest the data is to the CPU, the faster its manipulation will be (overly simplified). The CPU has access to different kinds of storage, here is a non-exhaustive list, from fastest to slowest:

- CPU registers
- CPU caches
- RAM

Again, I will not stress this enough, this is an **overly simplified** take on how things really works at the hardware level.

When you have a sequence (array) of data in memory you want to iterate over, it will be loaded in a CPU cache to speed things up.

The problem with our current “Object Oriented” design is that we have a nested loop!

```cpp
// game_objects array is loaded into a CPU cache
for (auto &game_object : game_objects) {
  // the components array is loaded into a CPU cache
  for (auto &component : game_object.iter_components()) {
    component.on_update();
  }

  // oops, the game_objects array is not in the cache anymore
  // so we have to go fetch it from RAM to continue the loop
}
```

This might not be a bottleneck for small games, but once you start to have tens of thousands, or millions, of game objects, this can start to have a non-negligeable effect.

The idea behind ECS is quite simple: What if all the data was contiguous so that we can take better advantage of CPU caches?

Imagine you have a big array of your data. Your “game objects” are now just an index in that array (we call them **entities**), and your logic simply iterate over that array:

```cpp
struct transform_component {
  vec2 position;
};

struct velocity_component {
  vec2 velocity;
};

void movement_system() {
  // components_array is loaded into a CPU cache
  for (auto &component_set : components_array) {
    auto &transform = component_set.get<transform_component>();
    auto &velocity  = component_set.get<velocity_component>();
    // modify transform using velocity data
  }
}

entity_type make_entity() {
  auto entity = components_array.add_entity();
  components_array.emplace<transform_component>(entity, /* ... */);
  components_array.emplace<velocity_component>(entity, /* ... */);
  return entity;
}
```

Regardless of performance, “Data Oriented Design” and the ECS pattern are about a much more important idea: **separating data from logic**.

In the earlier examples, we had both the data and logic in a single class. When some logic needs the data of multiple components, it creates a strong coupling as we saw the `movement_component` which needs the `transform_component`.

With an ECS, instead you have multiple “data-only” components which know nothing about other components, and systems which acts on entities which have a precise set of components.

It is the system that is coupled to the components it needs. Any entity that does not satisfy the requirements of the system will simply be ignored by it.

This architecture also makes it easier to parallelize computations because most of the time we’ll be dealing with embarrassingly parallel loops:

```cpp
// sequential
for (auto &component_set : components_array) {
  // ...
}

// parallel
std::for_each(
  std::execution::par,
  components_array.begin(),
  components_array.end(),
  [&](auto &component_set) {
    // ...
  }
);
```

**NB:** It should only be done if one system proves to be a bottleneck.

Ok, enough pseudo-code, let’s talk about actual implementation…

…

…

… nope, writing an efficient ECS implementation is **hard** and I’m lazy. Let’s use one that already exists.

---

## An ECS library: EnTT

![](img/image-bacf751586708ac4.png)

{{< cta-link
  url="https://github.com/skypjack/entt"
  title="GitHub - skypjack/entt: Gaming meets modern C++ - a fast and reliable entity component system (ECS)…"
  description="EnTT has been a dream so far, we haven't found a single bug to date and it's super easy to work with -- Every EnTT User…"
>}}

> `EnTT` is a header-only, tiny and easy to use library for game programming and much more written in modern C\+\+.  
> [Among others](https://github.com/skypjack/entt/wiki/EnTT-in-Action), it's used in [Minecraft](https://minecraft.net/en-us/attribution/) by Mojang, the [ArcGIS Runtime SDKs](https://developers.arcgis.com/arcgis-runtime/) by Esri and the amazing [Ragdoll](https://ragdolldynamics.com/).

The fact that it’s a header-only library means you can simply copy/paste it in your source-tree, somewhere the compiler can find the headers (usually by configuring the `-I` flag).

EnTT is not only an ECS, it also provides:

- generic containers (dense map, dense set) which are used to implement the “components array” we talked about earlier
- a cooperative scheduler, which you can use to implement your game loop (to chain your systems)
- events, signals and slots
- resource management, which is what I talked about in [my first devlog](/posts/2022-12-25-im-embedding-my-game-assets-into-the-executable-here-s-how-i-did-it) when implementing my asset manager
- runtime type information and runtime reflection
- service locator, which can be used as a form of dependency injection

This makes it a great framework to start building your engine and I cannot recommend it enough. For now, we’ll only talk about the ECS part.

The central element of the ECS is the **registry** (`entt::registry`). It provides the following API (list non-exhaustive):

- `entt::entity entt::registry::create()`   
  Creates a new entity and returns its ID.
- `T& entt::registry::emplace<T>(entt::entity eid)`   
  Assigns a component `T` to the entity `eid`, and return a reference to it.
- `void entt::registry::erase<T>(entt::entity eid)`   
  Removes the component `T` from the entity `eid`.
- `T& entt::registry::get<T>(entt::entity eid)`   
  Gets a reference to the component `T` on the entity `eid`.
- `bool entt::registry::valid(entt::entity eid)`   
  Checks if the entity `eid` is valid.
- `entt::basic_view entt::registry::view<T...>()`   
  Gets an iterable of all entities with the components `T...`.
- `T& entt::basic_view::get<T>(entt::entity eid)`   
  Gets a reference from the view to the component `T` on the entity `eid`.
- `bool entt::basic_view::contains(entt::entity eid)`   
  Checks if an entity exists in the view (if it doesn’t, it means that one or more components required by the view are missing on the entity)

We won’t need many more for the purpose of this article, feel free to consult the [documentation](https://skypjack.github.io/entt/) for more information.

It’s worth noting though, that the registry have a “dependency container” called a “context”. It allows us to emplace data to be used by anyone who have a need for it. It’s a form of dependency injection:

```cpp
entt::registry registry;

registry.ctx().emplace<int>(42);
auto &answer = registry.ctx().get<int>();

// or with a key
using namespace entt::literals;

registry.ctx().emplace<int>("foo"_hs, 1);
registry.ctx().emplace<int>("bar"_hs, 2);

auto &foo = registry.ctx().get<int>("foo"_hs);
auto &bar = registry.ctx().get<int>("bar"_hs);
```

In my case, I use the registry’s context to store a reference to the `asset_manager` (discussed in [the first devlog](/posts/2022-12-25-im-embedding-my-game-assets-into-the-executable-here-s-how-i-did-it)), the `input_manager` (will be the subject of a future devlog), and a few other utilities.

The engine I made to develop my game provides a few components (list non-exhaustive):

```cpp
struct transform {
  math::vec2 position;
  math::vec2 scale;
  // float rotation; -- not (yet?) used in my game
};

struct camera {
  int depth;
  math::rect view;      // portion of the world to render
  math::rect viewport;  // portion of the screen to render to

  // a few utility functions:
  math::vec2 center() const; // viewport's center
  math::vec2 scale() const;  // view/viewport size ratio
  math::vec2 offset() const; // view's position from the world origin

  math::vec2 world_to_screen(math::vec2 world_pos) const;
};

struct sprite {
  SDL_Texture *texture;
  SDL_Rect source;       // portion of the texture to render
  int layer;
  int order_in_layer;
};

struct shader {
  GLuint program;  // the asset manager is responsible for compiling the vertex/fragment shader
};
```

All of those components are used by the `render_system` provided by the engine.

I can create multiple cameras (to implement a split screen, or a minimap) by assigning a `camera` component to multiple entities. Then, every entities with both the `transform` and `sprite` components will be rendered in each camera. If the entity have a `shader` component, it will be used to render the sprite.

I intend to introduce a few other components, like `shape`, `circle`, or `square` for some HUD elements (like health bars).

Like every good 4X game, I need to control the camera with either the keyboard or the mouse, this will be done in a dedicated system. That system will iterate over all camera entities which have the following component as well:

```cpp
struct strategic_camera {
  math::vec2 target_position;
  math::vec2 original_size;

  struct {
    math::vec2 topleft;
    math::vec2 bottomright;
  } clamp;

  std::optional<math::vec2> drag_position;

  // decrease zoom_factor = zoom in
  // increase zoom_factor = zoom out
  float zoom_factor;
};
```

Now that we have the required components, let’s implement that camera controller system. First we need an interface, my engine provides this one:

```cpp
class system {
  public:
    virtual void run(entt::registry &registry) = 0;
};
```

The game loop contains a list of systems, each frame, they will be run sequentially (I chosed to not rely on EnTT’s cooperative scheduler to make things simpler in the beginning). When run, the system will receive the current registry from which we will iterate over the entities.

By being a class, a system can have configuration data associated to it, this is the case for the camera controller:

```cpp
class camera_controller_system final : public system {
  private:
    float m_pan_speed{16.0};
    float m_min_zoom_factor{0.1};
    float m_max_zoom_factor{1.0};
    float m_zoom_factor_step{0.05};

  public:
    virtual void run(entt::registry &registry);
};
```

Those member variables are used to control the behavior of the controller:

- `m_pan_speed` to control how fast the camera should move when using the keyboard
- `m_min_zoom_factor` / `m_max_zoom_factor` to clamp the camera’s `zoom_factor`
- `m_zoom_factor_step` to increase/decrease the camera’s `zoom_factor` when using the mouse wheel

The implementation looks like this (I omitted the input handling details, I’m going to write another devlog about it):

```cpp
void camera_controller_system::run(entt::registry &registry) {
  // get inputs from the input manager

  auto view = registry.view<camera, strategic_camera>();

  for (auto entity : view) {
    auto &camera  = view.get<camera>(entity);
    auto &scamera = view.get<strategic_camera>(entity);

    if (scamera.drag_position.has_value()) {
      // we are dragging the camera

      if (/* left click released */) {
        // we are no longer dragging the camera

        scamera.drag_position = std::nullopt;
      }
      else {
        // move the camera target
        auto mouse_pos = /* get mouse position from inputs */;
        auto delta_pos = scamera.drag_position.value() - mouse_pos;
        auto motion    = delta_pos * (1.0 / scamera.zoom_factor);

        scamera.target_position += motion;
        scamera.drag_position = mouse_pos;
      }
    }
    else if (/* left click pressed */) {
      // we start dragging the camera

      m_drag_pos = /* get mouse position from inputs */;
    }
    else {
      // we are not dragging the camera

      if (/* zooming in/out */) {
        auto zoom_sign = /* -1 for zoom in, 1 for zoom out */;
        scamera.zoom = math::clamp(
          scamera.zoom_factor + zoom_sign * m_zoom_step,
          m_min_zoom_factor,
          m_max_zoom_factor
        );
      }

      // move the camera target
      auto pan_direction = /* get vector from keyboard's arrow keys */;
      auto scaled_speed  = m_pan_speed * (1.0 / scamera.zoom_factor);
      auto motion        = pan_direction * scaled_speed;
      scamera.target_position += motion;

      // calculate the new camera size based on the zoom factor
      auto current_size = math::vec2(camera.view.w, camera.view.h);
      auto new_size     = math::lerp(
        current_size,
        scamera.original_size * scamera.zoom_factor,
        0.2
      );

      camera.view.w = new_size.x;
      camera.view.h = new_size.y;
    }

    // make sure we don't move the camera too far
    scamera.target_position.x = math::clamp(
      scamera.target_position.x,
      scamera.clamp.topleft.x,
      scamera.clamp.bottomright.x
    );
    scamera.target_position.y = math::clamp(
      scamera.target_position.y,
      scamera.clamp.topleft.y,
      scamera.clamp.bottomright.y
    );

    // move the camera towards its target
    camera.view.x = math::lerp(
      camera.view.x,
      scamera.target_position.x,
      0.2
    );
    camera.view.y = math::lerp(
      camera.view.y,
      scamera.target_position.y,
      0.2
    );
  }
}
```

**NB:** I use `math::lerp()` to smooth the camera movements when panning and zooming.

You can see here the power and flexibility of the ECS pattern. This system only cares about the `camera` and `strategic_camera` components. The entity might have a bunch of other components, it’s ok. There might be entities with only a `camera` component and no `strategic_camera` component (or vice-versa), they will be ignored. The system does not know about other systems. Everything is self-contained, and there is no strong coupling.

EnTT gives us the tools, then gets out of our way, what you see here is 99% game logic, no boilerplate.

Finally, we can create our entity:

```cpp
auto &dpy = registry.ctx().get<SDL_DisplayMode>(); // provided when SDL was initialized

auto entity = registry.create();

auto &camera = registry.emplace<camera>(entity);
camera.depth = 0;
camera.view.x = 0;
camera.view.y = 0;
camera.view.w = dpy.w;
camera.view.h = dpy.h;

camera.viewport.x = 0;
camera.viewport.y = 0;
camera.viewport.w = dpy.w;
camera.viewport.h = dpy.h;

auto &strategic_camera = registry.emplace<strategic_camera>(entity);
strategic_camera.target_position.x = 0;
strategic_camera.target_position.y = 0;
strategic_camera.original_size.x = dpy.w;
strategic_camera.original_size.y = dpy.h;
strategic_camera.drag_position = std::nullopt;
strategic_camera.zoom = 1.0;

// the hex namespace contains an implementation of hexagonal coordinates
// see https://www.redblobgames.com/grids/hexagons/ for more information 
strategic_camera.clamp.topleft =
  hex::coords::offset(0, 0)
    .oddr_to_cubic()
    .to_point(map.layout());

strategic_camera.clamp.bottomright =
  hex::coords::offset(map.width(), map.height())
    .oddr_to_cubic()
    .to_point(map.layout());
```

Here is the result:

[View the embedded animation on Gfycat](https://gfycat.com/ifr/frailpleasedcattle)

---

## The scene system

One thing I particularly enjoy with Unity are the Scene assets to implement different levels, or screens.

The ECS pattern provide nothing of the sort, which is why I implemented my own.

Each scene have its set of entities, and 3 hooks:

- `on_enter`: executed when we enter the scene, this is where we create our entities
- `on_update`: executed every frame, useful for GUI rendering with [imgui](https://github.com/ocornut/imgui) or interacting with the client thread (see my [previous devlog](/posts/2023-01-02-multiplayer-multithreading-and-an-actor-model-in-c))
- `on_exit`: executed when we leave the scene, this is where we destroy the entities

It is defined with the following interface:

```cpp
class scene {
  public:
    virtual void on_enter(entt::registry &registry) = 0;
    virtual void on_update(entt::registry &registry) = 0;
    virtual void on_exit(entt::registry &registry) = 0;
};
```

And we create a system to manage the scene objects:

```cpp
class scene_system : final : public system {
  private:
    std::shared_ptr<scene> m_current_scene{nullptr};

  public:
    virtual void run(entt::registry &registry);

    void change_scene(
      entt::registry &registry,
      std::shared_ptr<scene> new_scene
    );
};
```

```cpp
void scene_system::run(entt::registry &registry) {
  if (m_current_scene != nullptr) {
    m_current_scene->on_update(registry);
  }
}

void scene_system::change_scene(
  entt::registry &registry,
  std::shared_ptr<scene> new_scene
) {
  if (m_current_scene != nullptr) {
    m_current_scene->on_exit(registry);
  }

  m_current_scene = new_scene;

  if (m_current_scene != nullptr) {
    m_current_scene->on_enter(registry);
  }
}
```

I then store a pointer to that system in the registry’s context:

```cpp
auto sys = std::make_shared<scene_system>();
// add the system to the game loop
registry.ctx().emplace<std::shared_ptr<scene_system>>(sys);
```

Whenever I need to change the current scene, I can call this snippet:

```cpp
auto scene_manager = registry.ctx().get<std::shared_ptr<scene_system>>();
scene_manager->change_scene(registry, std::make_shared<ingame>());
```

A very dumb example of scene can be implemented like this:

```cpp
class ingame final : scene {
  private:
    std::vector<entt::entity> m_entities;

  public:
    virtual void on_enter(entt::registry &registry) {
      auto entity = registry.create();
      // setup entity components
      m_entities.push_back(entity);

      // ...
    }

    virtual void on_update(entt::registry &registry) {
      // ...
    }

    virtual void on_exit(entt::registry &registry) {
      registry.destroy(m_entities.begin(), m_entities.end());
      m_entities.clear();
    }
};
```

---

## Sorting components and the render system

Let’s take a look again at two components provided by my engine:

```cpp
struct camera {
  int depth;
  math::rect view;      // portion of the world to render
  math::rect viewport;  // portion of the screen to render to

  // a few utility functions:
  math::vec2 center() const; // viewport's center
  math::vec2 scale() const;  // view/viewport size ratio
  math::vec2 offset() const; // view's position from the world origin

  math::vec2 world_to_screen(math::vec2 world_pos) const;
};

struct sprite {
  SDL_Texture *texture;
  SDL_Rect source;       // portion of the texture to render
  int layer;
  int order_in_layer;
};
```

The render system will use those components to draw on the screen. But we need to draw in a specific order (defined by the camera’s depth and the sprite’s layer and order in layer).

To achieve this, we need to sort the entities. Thankfully, EnTT gives us the tools for that:

```cpp
registry.sort<sprite>([](const auto &lhs, const auto &rhs) {
  if (lhs.layer == rhs.layer) {
    return lhs.order_in_layer < rhs.order_in_layer;
  }

  return lhs.layer < rhs.layer;
});

registry.sort<camera>([](const auto &lhs, const auth &rhs) {
  return lhs.depth < rhs.depth;
});
```

The sort method will sort the components in-place (without allocations). If you have a lot of entities, this operation can be costly (in my game, with 14 thousands entities, it takes 40ms when built in Debug mode on Windows with sanitizers enabled, almost 3 frames!). This is why I only do it when creating entities and not at every frame.

**NB:** The ordering function must be a [strict weak ordering](https://www.boost.org/sgi/stl/StrictWeakOrdering.html). If it’s not, the sorting function might need to perform more checks than needed, impacting the performance. A previous buggy version of my ordering function resulted in a 2.5s long sorting.

Once sorted, the render system can hint at EnTT which ordering it needs when iterating over the entities:

```cpp
// automatically use the camera component ordering
auto cameras = registry.view<camera>();

// explicitly use the sprite component ordering
auto sprites = registry.view<transform, sprite>().use<sprite>();
```

Finally, the render system’s `run` method looks like this:

```cpp
void render_system::run(entt::registry &registry) {
  auto renderer = registry.ctx().get<SDL_Renderer *>();

  SDL_SetRenderDrawColor(renderer, 25, 25, 25, 255);
  SDL_RenderClear(renderer);

  auto shaders = registry.view<shader>();
  auto cameras = registry.view<camera>();
  auto sprites = registry.view<transform, sprite>().use<sprite>();

  for (auto camera_entity : cameras) {
    auto &camera = cameras.get<camera>(camera_entity);

    SDL_Rect viewport = {
      .x = int(std::floor(camera.viewport.x)),
      .y = int(std::floor(camera.viewport.y)),
      .w = int(std::floor(camera.viewport.w)),
      .h = int(std::floor(camera.viewport.h))
    };
    SDL_RenderSetViewport(renderer, &viewport);

    for (auto sprite_entity : sprites) {
      auto &transform = sprites.get<components::transform>(sprite_entity);
      auto &sprite = sprites.get<components::sprite>(sprite_entity);

      auto screen_pos = camera.world_to_screen(transform.position);
      auto cam_scale = camera.scale();

      SDL_FRect dest = {
        .x = screen_pos.x,
        .y = screen_pos.y,
        .w = sprite.source.w * transform.scale.x * cam_scale.x,
        .h = sprite.source.h * transform.scale.y * cam_scale.y
      };

      if (shaders.contains(sprite_entity)) {
        auto &shader = shaders.get<components::shader>(sprite_entity);
        // render texture with shader
      }
      else {
        SDL_RenderCopyF(renderer, sprite.texture, &sprite.source, &dest);
      }
    }
  }

  SDL_RenderSetViewport(renderer, nullptr);

  ImGui_ImplSDLRenderer_NewFrame();
  ImGui_ImplSDL2_NewFrame();
  ImGui::NewFrame();

  // execute commands in the `ui_command_buffer` (more on that later)

  ImGui::Render();
  ImGui_ImplSDLRenderer_RenderDrawData(ImGui::GetDrawData());

  SDL_RenderPresent(renderer);
}
```

**NB:** Currently, I only have a `sprite` component, but when later on I will include shapes, circles, rectangles, etc… to draw primitives on the screen, they may share the `layer` and `order_in_layer` properties. At that moment, I might create a dedicated component for those properties, to be used in the render system as well.

---

## Bonus: EnTT editor with imgui

Debugging at runtime is an important feature. The Unity Inspector is a great tool for this, I’d like something similar to be able to inspect and modify the data of my entities and components while the game is running.

Fortunately, it exists!

{{< cta-link
  url="https://github.com/Green-Sky/imgui_entt_entity_editor"
  title="GitHub - Green-Sky/imgui_entt_entity_editor: A drop-in entity editor for EnTT with Dear ImGui"
  description="A drop-in, single-file entity editor for EnTT, with ImGui as graphical backend. Originally developed for MushMachine…"
>}}

It’s a header-only library, which adds a framework to build inspectors for your components, for example:

```cpp
#pragma once

#include <imgui-entt-editor.hpp>

struct camera {
  int depth;
  math::rect view;
  math::rect viewport;

  // ...
};

namespace MM {
  template<>
  void ComponentEditorWidget<camera>(
    entt::registry &registry,
    entt::registry::entity_type entity
  ) {
    auto &cam = registry.get<camera>(entity);
    ImGui::DragInt("depth", &cam.depth, 1);

    ImGui::Text("View");
    ImGui::Indent();
    ImGui::DragFloat("x##view_x", &cam.view.x, 0.1f);
    ImGui::DragFloat("y##view_y", &cam.view.y, 0.1f);
    ImGui::DragFloat("w##view_w", &cam.view.w, 0.1f);
    ImGui::DragFloat("h##view_h", &cam.view.h, 0.1f);
    ImGui::Unindent();

    ImGui::Text("Viewport");
    ImGui::Indent();
    ImGui::DragFloat("x##viewport_x", &cam.viewport.x, 0.1f);
    ImGui::DragFloat("y##viewport_y", &cam.viewport.y, 0.1f);
    ImGui::DragFloat("w##viewport_w", &cam.viewport.w, 0.1f);
    ImGui::DragFloat("h##viewport_h", &cam.viewport.h, 0.1f);
    ImGui::Unindent();
  }
}
```

Using this library, I defined a “debug system”:

```cpp
class debug_system final : public engine::system {
  private:
    using Editor = MM::EntityEditor<entt::entity>;
    Editor m_editor;

    std::set<Editor::ComponentTypeID> m_components_filter;
    entt::entity m_current_entity;

    bool m_open{false};

  public:
    template <typename Component>
    void register_component(const std::string &display_name) {
      m_editor.registerComponent<Component>(display_name);
    }

    virtual void run(entt::registry &registry) {
      // gather inputs

      if (/* backspace released */) {
        m_open = !m_open;
      }

      if (m_open) {
        // more on that later
        auto &add_ui_command = registry.ctx().get<ui_command_request>();
        add_ui_command([&](entt::registry &registry) {
          ImGui::Begin("Inspector");

          ImGui::Columns(2);
          ImGui::BeginChild("Entity List");
          m_editor.renderEntityList(registry, m_components_filter);
          ImGui::EndChild();
          ImGui::NextColumn();

          ImGui::BeginChild("Entity Editor");
          m_editor.renderEditor(registry, m_current_entity);
          ImGui::EndChild();
          ImGui::NextColumn();

          ImGui::End();
        });
      }
    }
};
```

When initializing the engine, I emplace in the registry’s context the pointer to the debug system:

```cpp
registry.ctx().emplace<std::shared_ptr<debug_system>>(m_debug_system);
```

And finally, I can register my components like this:

```cpp
auto debug_system = registry.ctx().get<std::shared_ptr<debug_system>>();
debug_system->register_component<camera>("Camera");
```

And the result is:

![](img/image-cf8492bf6a067edb.png)

---

## Conclusion

EnTT is really a great library, I really enjoy the “gives you the tools and get out of your way” philosophy. I can’t recommend it enough 🙂

There is a Gitter channel where the author is active if you need help, people are nice there so feel free to join:

{{< cta-link
  url="https://gitter.im/skypjack/entt"
  title="skypjack/entt"
  description="Pointer stability creates holes (that are filled as soon as possible), so in theory it can affect linear iterations…"
>}}

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
