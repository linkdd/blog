---
title: "Trying out SDL3 by writing a C++ Game Engine"
date: "2025-02-27T08:20:14Z"
description: "SDL3 is around the corner! SDL3_image has been released, and we’re waiting for SDL3_ttf, SDL3_mixer and SDL3_net to be released soon as…"
tags: ["gamedev", "programming", "cpp", "game-engine"]
author: "David Delassus"
images: ["img/image-536706396717efaa.png"]
---

![](img/image-536706396717efaa.png)

---

SDL3 is around the corner! SDL3\_image has been released, and we’re waiting for SDL3\_ttf, SDL3\_mixer and SDL3\_net to be released soon as well.

On the menu:

- more consistent naming conventions
- many API improvements (native dialog windows, main callbacks, …)
- integration of old third-party libraries for GPU code
- …

But is it ready for production?

In my opinion, not yet. There are still some performance problem (both CPU-wise and memory-wise) that needs to be ironed out, a few bugs to be fixed, etc. SDL2 has been battled tested for more than a decade, SDL3 is still young, but that’s only a temporary issue.

Also, that’s no reason to not try it out! Let’s try to make a game with SDL3!

Wait… Why make games when you can make a game engine instead?

Making games is hard. Making a game engine is harder. But it’s actually funnier (and it does not require art skills). Let’s do it step by step:

---

## Step 1: The boilerplate

```cpp
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>
#include <entt/entt.hpp>

#define FPS 60


struct global_state {
  bool running;

  SDL_Window* window;
  SDL_Renderer* renderer;

  entt::registry registry;
};


void on_init(global_state& state) {
  // ...
}


void on_event(global_state& state, const SDL_Event& event) {
  // ...
}


void on_update(global_state& state, float delta_time) {
  // ...
}


void on_teardown(global_state& state) {
  // ...
}


int main(int argc, char** argv) {
  if (!SDL_Init(SDL_INIT_VIDEO)) {
    SDL_Log("Init failed: %s", SDL_GetError());
    return 1;
  }

  SDL_Window* win = SDL_CreateWindow("example", 800, 600, 0);
  if (win == nullptr) {
    SDL_Log("Create window failed: %s", SDL_GetError());
    SDL_Quit();
    return 1;
  }

  SDL_Renderer* rdr = SDL_CreateRenderer(win, nullptr);
  if (rdr == nullptr) {
    SDL_Log("Create renderer failed: %s", SDL_GetError());
    SDL_DestroyWindow(win);
    SDL_Quit();
    return 1;
  }

  global_state state = {
    .running  = true,
    .window   = win,
    .renderer = rdr,
  };

  on_init(state);

  Uint64 last_frame_end = SDL_GetTicksNS();

  while (state.running) {
    Uint64 frame_begin = SDL_GetTicksNS();
    float delta_time = (frame_begin - last_frame_end) / SDL_NS_PER_SECOND;

    SDL_Event evt;
    while (!SDL_PollEvent(&evt)) {
      on_event(state, evt);

      switch (evt.type) {
        case SDL_EVENT_QUIT:
          state.running = false;
          break;

        case SDL_EVENT_WINDOW_CLOSE_REQUESTED:
          if (evt.window.windowID == SDL_GetWindowID(win)) {
            state.running = false;
          }
          break;

        default:
          break;
      }
    }

    on_update(state, delta_time);

    last_frame_end = SDL_GetTicksNS();
    Uint64 elapsed = last_frame_end - frame_begin;
    Uint64 target  = SDL_NS_PER_SECOND / FPS;

    if (elapsed < target) {
      SDL_DelayNS(target - elapsed);
    }
  }

  on_teardown(state);

  SDL_DestroyRenderer(rdr);
  SDL_DestroyWindow(win);
  SDL_Quit();

  return 0;
}
```

---

## Step 2: Scene hierarchy, the ECS way

As you can see in the boilerplate above, there is no rendering done. Not yet.

We want to implement a rendering system that takes advantage of EnTT as the Entity/Component/System. A while ago, I wrote an introduction to EnTT which showcases how I used it in my totally not abandoned game that I haven’t touched in years:

{{< cta-link
  url="/posts/2023-01-08-a-short-introduction-to-entity-component-system-in-c-with-entt"
  title="A short introduction to Entity-Component-System in C++ with EnTT"
  description="Warmonger Dynasty Devlog #3: An ECS with EnTT"
>}}

The “render system” showcased in that article isn’t very satisfying. I decided to give it another go here.

The first problem my old implementation had is that there is no hierarchy between entities. All transforms are in “world space”. There is no such thing as “local space”.

Yet, the scene hierarchy is a powerful tool to organize entities. Let’s say you want the camera to follow the player:

- without hierarchy, you have to update both the player entity’s transform, **and** the camera entity’s transform
- with a hierarchy, you put the camera entity as a child of the player entity, and when the player entity’s transform is updated, the camera moves along

But how does one model a hierarchy using an ECS? Let’s look at how Unity does it:

{{< cta-link
  url="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/transforms-concepts.html"
  title="Transform concepts"
  description="You can use the Unity.Transforms namespace to control the position, rotation, and scale of any entity in your project…"
>}}

It defines 2 components that the developer will use, `LocalTransform` and `Parent`. And it provides 2 systems that will populate the components `LocalToWorld` and `Child`.

Let’s create those components for our 2D engine:

```cpp
#include <unordered_set>

#include <SDL3/SDL.h>
#include <entt/entt.hpp>

#include "mat3x3.hpp"
// defines a `mat3x3` type to hold basic 2D transformations:
//  - translation
//  - scaling
//  - rotation
//
// Also provides operator overloading for:
//  - mat3x3 * mat3x3 = mat3x3
//  - mat3x3 * SDL_FPoint = SDL_FPoint
//  - mat3x3 * SDL_FRect = SDL_FRect

struct local_transform {
  SDL_FPoint position;
  SDL_FPoint scale;
  float rotation;
};

struct parent {
  entt::entity entity{entt::null};
};

namespace internal {
  struct local_to_world {
    mat3x3 mat;
  };

  struct children {
    std::unordered_set<entt::entity> entities;
  };
}
```

Whenever you want to make an entity a child of another, you attach the `parent` component to it. Then, a system (that we will implement) will take care of attaching the `children` component to the relevant entities. This way, other systems will be able to walk through the tree from parent to children, or from child to ancestors.

Let’s implement that system. It has 2 tasks:

- add the `children` component to every entity referenced by a `parent` component
- remove from the `children` component every entity that does not have it as parent anymore, or entities that were destroyed

This can be implemented as a free function:

```cpp
void parent_system(entt::registry& registry) {
  auto child_pov  = registry.view<parent>();
  auto parent_pov = registry.view<internal::children>();

  // add to `children` every entity with a `parent`
  for (auto entity : child_pov) {
    auto& c_parent = child_pov.get<parent>(entity);

    if (registry.valid(c_parent.entity)) {
      auto& c_children = registry.get_or_emplace<internal::children>(c_parent.entity);
      c_children.entities.insert(entity);
    }
  }

  // remove from `children`:
  //  - every invalid entities
  //  - entities with the a different `parent`
  //  - entities without a `parent`
  for (auto entity : parent_pov) {
    auto& c_children = parent_pov.get<internal::children>(entity);

    auto to_remove = std::vector<entt::entity>{};
    to_remove.reserve(c_children.entities.size());

    for (auto child : c_children.entities) {
      if (!registry.valid(child)) {
        // the entity was destroyed
        to_remove.push_back(child);
      }
      else if (child_pov.contains(child)) {
        auto& c_parent = child_pov.get<parent>(child);

        if (c_parent.entity != entity) {
          // the entity was assigned to another parent
          to_remove.push_back(child);
        }
      }
      else {
        // the entity does not have a parent anymore
        to_remove.push_back(child);
      }
    }

    for (auto child : to_remove) {
      c_children.entities.remove(child);
    }
  }
}
```

We now have the ability to traverse the hierarchy of entities, but we cannot yet position them correctly in the world. To do that, we need to implement the system that will populate the `local_to_world` components.

Walking through the hierarchy can be done recursively. For every entity with a `local_transform` component, we will compute the transformation matrix. If said entity has a parent, we will need to compute the parent’s transformation matrix and multiply them. And so on with the grand parent’s, and grand grand parents, and… You get the gist.

Because a parent can have many children, it is entirely possible that we already computed the world transformation matrix for a parent. We need to memorize which entity has already been visited, so that we can avoid unnecessary computations (the best optimization is to do less work).

This system can be implemented as a free function:

```cpp
void local_to_world_system(entt::registry& registry) {
  auto view    = registry.view<local_transform>();
  auto visited = std::unordered_set<entt::entity>{};

  // recursive function to traverse the hierarchy
  // `self` is the function itself, for recursion,
  // see: https://en.wikipedia.org/wiki/Fixed-point_combinator
  auto compute_world_matrix = [&](auto& self, entt::entity e) -> mat3x3 {
    if (visited.contains(e)) {
      // we already visited this entity, let's return its world matrix
      auto& c_local_to_world = registry.get<internal::local_to_world>(e);
      return c_local_to_world.mat;
    }

    auto local_matrix = mat3x3::identity();

    // it is possible that the developer referenced a parent entity that
    // has no local transform.
    if (view.contains(e)) {
      // but if it does, let's compute the local matrix
      const auto& c_local_transform = view.get<local_transform>(e);

      local_matrix = mat3x3::from_transform(
        c_local_transform.position,
        c_local_transform.scale,
        c_local_transform.rotation
      );
    }

    auto world_matrix = local_matrix;

    // if the entity has a parent and this parent is a valid entity,
    // let's recurse
    auto* c_parent = registry.try_get<parent>(e);
    if (c_parent != nullptr && registry.valid(c_parent)) {
      auto parent_world_matrix = self(self, c_parent->entity);
      world_matrix = parent_world_matrix * world_matrix;
    }

    // finally update or insert the final component
    registry.emplace_or_replace<internal::local_to_world>(e, world_matrix);
    visited.insert(e);

    return world_matrix;
  };

  for (auto entity : view) {
    compute_world_matrix(compute_world_matrix, entity);
  }
}
```

And voilà! We now have a fully functional scene hierarchy. We just need to call our systems in our update function:

```cpp
void on_update(global_state& state, float delta_time) {
  // update local transforms, or parents...

  parent_system(state.registry);
  local_to_world_system(state.registry);
}
```

---

## Step 3: Drawing things

Now that we know “where” to draw, we need to know “what” to draw.

In my previous implementation, I had many components:

- sprites \& animated sprites
- tilemaps
- shaders
- triangles, quads, polygons and various other geometries
- …

Then, I had an `ordering` layer to determine in which order things should be drawn.

It was tedious, the code for the rendering system was huge, hard to maintain and error-prone. On top of that, it acted as an abstraction layer over SDL while only being a fraction of what SDL provides. Instead, I chose to use the “strategy pattern” and have a single `drawable` component:

```cpp
#include <functional>

#include <SDL3/SDL.h>


struct drawable {
  Sint64 depth;
  SDL_FRect bounding_box;
  std::function<void(SDL_Renderer*, math::mat3x3, float)> draw;

  struct compare {
    bool operator()(const drawable& lhs, const drawable& rhs) const {
      return lhs.depth < rhs.depth;
    }
  };
};
```

**NB:** The component includes a functor, useful for calling `registry.sort()`.

In the above example, we see 3 properties that need clarifying:

- `depth`: in 2D world, this is the “actual” 3rd dimension, it tells the rendering system in which order things should be drawn; on a tilemap it’s often equal to `-y` so that a player can walk “behind” other sprites
- the `draw` strategy receives 3 parameters: the actual SDL renderer, the transformation matrix to use to correctly place things on the screen, and the `delta_time` (for animations)
- `bounding_box`: this is used for culling, more details bellow

As said above, the best optimization is to do less work. Not drawing what is not seen anyway, aka: culling, is one such optimization. Why send a draw call to the GPU for something that is not in view?

To implement culling, we need some notion of a “camera”. A camera takes a portion of the world space, and renders it to a portion of the screen space:

![](img/image-e821882cafc781ed.png)

The “world” will need to be rendered for each camera, and then finally drawn on the screen. It is possible that 2 viewports overlaps. For example, in an RTS, we have the main camera that fills the screen, and the camera for the minimap that fills only a small corner:

![](img/image-4677b895fa39c7d6.jpg)

Age of Empire 2

It would be a shame to have the minimap rendered first, and then hidden behind the main camera. To solve this, we need a 3rd dimension for cameras as well, we’ll call this the camera “Z index”.

This gives us the following component:

```cpp
struct camera {
  Sint64 z_index;
  SDL_FRect view;
  SDL_FRect viewport;

  struct compare {
    bool operator()(const camera& lhs, const camera& rhs) const {
      return lhs.z_index > rhs.z_index;
    }
  };
};
```

Now that we have a camera, we can use its “view” to determine if an entity is seen by said camera. This is actually where the `bounding_box` property comes in.

This property defines the “2D volume” an entity use in the world. It is relative to the entity’s `local_transform` component, and as such, it must be translated to world space. It must also include the “volume” of all the child entities. To satisfy this property, we walk through the scene hierarchy recursively, and calculate the union of the entity’s bounding box and its children bounding box.

Here is what the union of 2 rectangles look like:

![](img/image-bbd261c1ad7cf436.png)

<https://learn.microsoft.com/en-us/windows/win32/gdi/rectangle-operations>

We will need a system to compute this world space bounding box:

```cpp
namespace internal {
  struct bounding_box {
    SDL_FRect rect;
  };
}


void bounding_box_system(entt::registry& registry) {
  auto view    = registry.view<drawable>();
  auto visited = std::unordered_set<entt::entity>{};
  visited.reserve(view.size());

  auto compute_bbox = [&](auto& self, entt::entity e) -> SDL_FRect {
    if (visited.contains(e)) {
      auto& c_bbox = registry.get<internal::bounding_box>(e);
      return c_bbox.rect;
    }

    auto bbox = SDL_FRect{0.0f, 0.0f, 0.0f, 0.0f};

    // child entities may have no drawables, but their children might have.
    // let's check if the component exists and assume a default empty rect
    // for the bounding box

    auto* c_drawable  = registry.try_get<drawable>(e);
    auto* c_transform = registry.try_get<internal::local_to_world>(e);
    auto* c_children  = registry.try_get<internal::children>(e);

    if (c_drawable != nullptr && c_transform != nullptr) {
      // we have a drawable, let's compute the actual local bounding box
      bbox = c_transform->mat * c_drawable->bounding_box;
    }

    if (c_children != nullptr) {
      // we have children, let's get the union of their bounding box
      for (auto child : c_children->entities) {
        // get child bounding box (recursively)
        auto child_bbox = self(self, child);

        // bbox U child_bbox --> bbox
        SDL_GetRectUnionFloat(&bbox, &child_bbox, &bbox);
      }
    }

    // add the component to the entity
    registry.emplace_or_replace<internal::bounding_box>(e, bbox);
    visited.insert(e);

    return bbox;
  };

  for (auto entity : view) {
    compute_bbox(compute_bbox, entity);
  }
}
```

We are almost there. We need to render the world for each camera, and then render the actual camera on the screen. To do this, we need a texture to draw to. Draw calls need to be sorted by their depth, we can use a priority queue. Therefore, each camera will have a queue and a texture associated to it, this can be done with an internal component:

```cpp
#include <queue>


namespace internal {
  struct draw_call {
    drawable drawable;
    mat3x3 world_transform;

    struct compare {
      bool operator()(const draw_call& lhs, const draw_call& rhs) const {
        return components::drawable::compare{}(lhs.drawable, rhs.drawable);
      }
    };
  };

  struct draw_target {
    using draw_queue_type = std::priority_queue<
      draw_call,
      std::vector<draw_call>,
      draw_call::compare
    >;

    draw_queue_type queue;
    SDL_Texture *target;
  };
}
```

We will need to automatically add this `internal::draw_target` component whenever a `camera` component is added (and also remove it when the entity is destroyed, or the component removed). Fortunately, EnTT provides just that:

```cpp
static void construct_draw_target(entt::registry& registry, entt::entity entity) {
  registry.emplace<internal::draw_target>(entity);
}

static void destroy_draw_target(entt::registry& registry, entt::entity entity) {
  registry.remove<internal::draw_target>(entity);
}

void camera_setup_system(entt::registry& registry) {
  registry.on_construct<camera>().connect<&construct_draw_target>();
  registry.on_destroy<camera>().connect<&destroy_draw_target>();
}
```

The render system will need to sort cameras by their Z index in order to render them in the correct order. Then, for each camera, we will iterate over all `drawable` components and push their draw strategy in the camera’s queue. Because it is a priority queue, draw calls will always be in the correct order (according to their `depth`).

But, if we sort the `drawable` components, we ensure that the draw calls are inserted in order in the queue as well.

We can use `registry.sort<T>(compare_fn)` for this. By default, this method uses insertion sort, which is faster than quicksort when the data is mostly sorted.

We also need to actually create the `draw_target` texture. We create one with the size of the camera view, and recreate it whenever the view changes.

The render process for a single camera is as follows:

- clear the render target
- translate the camera `view` from the entity’s local space to world space
- check if the drawable entity’s bounding box intersects with the world space view
- compute the “world space to screen space” transformation matrix for the drawable entity
- push the draw call in the camera draw queue
- once all drawable entities have been iterated over, execute each draw call in queue
- finally, render the camera texture to the screen at `viewport`

And once again, our system can be implemented as a free function:

```cpp
static SDL_Texture* prepare_draw_target_texture(
  SDL_Renderer* renderer,
  SDL_Texture* target,
  SDL_FRect view
) {
  if (target != nullptr) {
    float w, h;
    SDL_GetTextureSize(target, &w, &h);

    if (w != view.w || h != view.h) {
      SDL_DestroyTexture(target);
      target = nullptr;
    }
  }

  if (target == nullptr) {
    target = SDL_CreateTexture(
      renderer,
      SDL_PIXELFORMAT_RGBA8888,
      SDL_TEXTUREACCESS_TARGET,
      view.w, view.h
    );
  }

  return target;
}
      

void render_system(
  entt::registry& registry,
  SDL_Renderer* renderer,
  float delta_time
) {
  registry.sort<camera>(camera::compare{});
  registry.sort<drawable>(drawable::compare{});

  auto camera_entities = registry.view<
    camera,
    internal::local_to_world,
    internal::draw_target
  >();
  // iterate over entities using the `camera` component ordering
  camera_entities.use<camera>();

  auto drawable_entities = registry.view<
    drawable,
    internal::local_to_world,
    internal::bounding_box
  >();
  // iterate over entities using the `drawable` component ordering
  drawable_entities.use<drawable>();

  for (auto e_camera : camera_entities) {
    auto& c_camera           = camera_entities.get<camera>(e_camera);
    auto& c_camera_transform = camera_entities.get<internal::local_to_world>(e_camera);
    auto& c_draw_target      = camera_entities.get<internal::draw_target>(e_camera);

    c_draw_target.target = prepare_draw_target_texture(
      renderer,
      c_draw_target.target,
      c_camera.view
    );
    if (c_draw_target.target == nullptr) {
      continue;
    }

    SDL_SetRenderTarget(renderer, c_draw_target.target);

    // clear target
    auto bak = SDL_Color{};
    SDL_GetRenderDrawColor(renderer, &bak.r, &bak.g, &bak.b, &bak.a);
    SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
    SDL_RenderClear(renderer);
    SDL_SetRenderDrawColor(renderer, bak.r, bak.g, bak.b, bak.a);

    // translate view from local space to world space
    auto view_local_pos = SDL_FPoint{c_camera.view.x, c_camera.view.y};
    auto view_world_pos = c_camera_transform.mat * view_local_pos;
    auto world_view = SDL_FRect{
      .x = view_world_pos.x - c_camera.view.w / 2.0f,
      .y = view_world_pos.y - c_camera.view.h / 2.0f,
      .w = c_camera.view.w,
      .h = c_camera.view.h,
    };

    for (auto e_drawable : drawable_entities) {
      auto& c_drawable           = drawable_entities.get<drawable>(e_drawable);
      auto& c_drawable_transform = drawable_entities.get<internal::local_to_world>(e_drawable);
      auto& c_bounding_box       = drawable_entities.get<internal::bounding_box>(e_drawable);

      if (SDL_HasRectIntersectionFloat(&world_view, &c_bounding_box.rect)) {
        // entity's bounding box intersects with the camera view

        // compute world space to screen space transformation matrix
        auto M_offset = mat3x3::translation({-view_world_pos.x, -view_world_pos.y});
        auto M_center = mat3x3::translation({c_camera.view.w / 2.0f, c_camera.view.h / 2.0f});
        auto M        = (M_center * M_offset) * c_drawable_transform.mat;

        // push draw call
        c_draw_target.queue.emplace(internal::draw_call{
          .drawable = c_drawable,
          .world_transform = M,
        });
      }
    }

    // execute draw calls
    while (!c_draw_target.queue.empty()) {
      auto draw_call = c_draw_target.queue.top();
      c_draw_target.queue.pop();

      draw_call.drawable.draw(renderer, draw_call.world_transform, delta_time);
    }

    // render camera texture to screen
    // NB: SDL will automatically stretch the texture to the viewport
    SDL_SetRenderTarget(renderer, nullptr);
    SDL_RenderTexture(renderer, c_draw_target.target, nullptr, &c_camera.viewport);
  }
}
```

All that remains, is to call the systems in our update function:

```cpp
void on_init(global_state& state) {
  // ...

  camera_setup_system(state.registry);
}

void on_update(global_state& state, float delta_time) {
  // update local transforms, parents, or drawables...

  parent_system(state.registry);
  local_to_world_system(state.registry);
  bounding_box_system(state.registry);
  render_system(state.registry, state.renderer, delta_time);
}
```

And voilà!

We now have the basics for actual game making.

---

## Stay Tuned!

This article is the first of a series, by the end of that series, we will have a game (hopefully).

The next part will be about a very basic homemade 2D collision detection system. It will be less about SDL3, but I’m waiting for SDL3\_mixer to come out as stable in order to make an audio system.

As for the game I’m gonna implement, I might try myself at a remake of an old time favorite NES game of mine: **Kickle Cubicle**.

[Watch the embedded video on YouTube](https://www.youtube.com/watch?v=XEI1TMrX0cY)

I even found the spritesheets online (not sure about the license though):

{{< cta-link
  url="https://www.spriters-resource.com/nes/kicklecubicle/"
  title="NES - Kickle Cubicle - The Spriters Resource"
  description="NES - Kickle Cubicle - The #1 source for video game sprites on the internet!"
>}}
