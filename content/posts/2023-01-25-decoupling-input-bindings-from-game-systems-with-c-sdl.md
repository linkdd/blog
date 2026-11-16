---
title: "Decoupling input bindings from game systems with C++/SDL"
date: "2023-01-25T17:36:26Z"
description: "Warmonger Dynasty Devlog #4: An Input Manager inspired by Unity"
tags: ["gamedev", "cpp", "programming"]
author: "David Delassus"
images: ["img/image-163fabdebc6d832b.png"]
---

---

![](img/image-163fabdebc6d832b.png)

This is the 4th devlog of Warmonger Dynasty, a 4X turn-based strategy game made in C\+\+/SDL. If you want to read the previous entries, take a look at my [reading list](/reading-lists/warmonger-dynasty).

When developing a game, you almost certainly want to read inputs from the player. Be it via the mouse, the keyboard, a gamepad, or a touchscreen.

My game is no exception, and the controls are via the mouse/keyboard. However, I don’t want to hard-code the bindings in my game systems. That would make it hard to make them configurable later on, and even harder to keep track of.

Therefore, I need to design an Input Manager which will allow my game systems to query for inputs without worrying about the specific bindings tied to them.

---

## 3 types of input actions

For my needs, I identified 3 player actions I want to be able to react to:

1. **Trigger:** like a mouse click, I want to know if the action is active (the mouse button is still pressed), if the action has been performed (the mouse button has been pressed this frame), and if the action has been cancelled (the mouse button has been released this frame)
2. **Axis:** like the keyboard’s arrows (or WASD), or a joystick, I want to know the value of the horizontal and vertical axis (between 0 and 1\)
3. **Passthrough:** like the mouse pointer, I want to know where it is on the screen

To hold the values of those actions, we’ll need the following structures:

```
namespace input {
  struct action_trigger {
    bool active;
    bool performed;
    bool cancelled;
  };

  struct action_axis {
    float x;
    float y;
  };

  template <typename T>
  struct action_passthrough {
    T raw;
  };
}
```

---

## The input state

I want a structure to hold the informations my manager is going to query:

```
namespace input {
  struct state {
    const Uint8 *keyboard_state;

    Uint32 mouse_button_mask;
    int mouse_x;
    int mouse_y;
    int mouse_motion_x;
    int mouse_motion_y;
    int mouse_wheel_x;
    int mouse_wheel_y;
  };
}
```

This structure is to be filled by the manager using SDL functions and events:

```
namespace input {
  void manager::frame_begin() {
    m_state.mouse_wheel_x = 0;
    m_state.mouse_wheel_y = 0;
    m_state.mouse_motion_x = 0;
    m_state.mouse_motion_y = 0;
  }

  void manager::process_event(SDL_Event *event) {
    switch (event->type) {
      case SDL_MOUSEWHEEL:
        m_state.mouse_wheel_x = event->wheel.x;
        m_state.mouse_wheel_y = event->wheel.y;
        break;

      case SDL_MOUSEMOTION:
        m_state.mouse_motion_x = event->motion.xrel;
        m_state.mouse_motion_y = event->motion.yrel;
        break;

      default:
        break;
    }
  }

  void manager::update(entt::registry &registry) {
    m_state.keyboard_state = SDL_GetKeyboardState(nullptr);
    m_state.mouse_button_mask = SDL_GetMouseState(
      &m_state.mouse_x,
      &m_state.mouse_y
    );

    // ...
  }
}
```

We’ll look closer at the manager’s definition/implementation later.

---

## An input binding abstraction

Inputs may come from mouse buttons, the keyboard, a gamepad’s button, etc… I need an interface to abstract this:

```
namespace input {
  namespace details {
    class binding_check {
      public:
        virtual ~binding_check() {};

        virtual bool check(entt::registry &registry, const state &state) const {
          return false;
        }
    };
  }
}
```

The `check()` method will check if the binding is active **this frame**. We provide a default implementation which always return false.

Now, we can implement this interface for the various input sources we’ll have (for now, only keyboard and mouse):

```
namespace input {
  namespace details {
    template <Uint8 scancode>
    class key_binding final : public binding_check {
      public:
        virtual bool check(entt::registry &registry, const state &state) const {
          auto &io = registry.ctx().get<ImGuiIO &>();
          return (!io.WantCaptureKeyboard && state.keyboard_state[scancode]);
        }
    };

    template <Uint32 buttonmask>
    class mouse_button_binding final : public binding_check {
      public:
        virtual bool check(entt::registry &registry, const state &state) const {
          auto &io = registry.ctx().get<ImGuiIO &>();
          return (!io.WantCaptureMouse && (state.mouse_button_mask & buttonmask) != 0);
        }
    };
  }
}
```

**NB:** We get the ImGui IO manager to make sure the inputs happened on our game scene and not in the UI (to avoid clicking the map when clicking on a UI button).

The `key_binding` implementation will read from the `keyboard_state` filled by SDL. The template parameter expects one of the `SDL_SCANCODE_*` macro, like:

- `SDL_SCANCODE_LEFT`
- `SDL_SCANCODE_A`
- `SDL_SCANCODE_BACKSPACE`
- …

The `mouse_button_binding` implementation will read from the `mouse_button_mask` filled by SDL. The template parameter expects one of the `SDL_BUTTON_*` macro, like:

- `SDL_BUTTON_LEFT`
- `SDL_BUTTON_RIGHT`

The next step is to allow any binding to be combined with others. We want a way to express shortcuts like “Ctrl\+C”, “W or Up”, “A or Left”, …

For this purpose, we will need 2 more classes:

```
namespace input {
  namespace details {
    class binding_or_combinator final : public binding_check {
      private:
        std::unique_ptr<binding_check> m_a;
        std::unique_ptr<binding_check> m_b;

      public:
        binding_or_combinator(
          std::unique_ptr<binding_check> a,
          std::unique_ptr<binding_check> b
        ) : m_a(std::move(a)), m_b(std::move(b)) {}

        virtual bool check(entt::registry &registry, const state &state) const {
          return m_a->check(registry, state) || m_b->check(registry, state);
        }
    };

    class binding_and_combinator final : public binding_check {
      private:
        std::unique_ptr<binding_check> m_a;
        std::unique_ptr<binding_check> m_b;

      public:
        binding_and_combinator(
          std::unique_ptr<binding_check> a,
          std::unique_ptr<binding_check> b
        ) : m_a(std::move(a)), m_b(std::move(b)) {}

        virtual bool check(entt::registry &registry, const state &state) const {
          return m_a->check(registry, state) && m_b->check(registry, state);
        }
    };
  }
}
```

**NB:** The pointer is needed to avoid the slicing problem (in order to call the correct implementation of `check()`).

Now, the last thing we need regarding the bindings abstraction is some helper functions:

```
namespace input {
  using binding_type = std::unique_ptr<details::binding_check>;

  binding_type nobinding() {
    return std::make_unique<details::binding_check>();
  }

  template <Uint8 scancode>
  binding_type key() {
    return std::make_unique<details::key_binding<scancode>>();
  }

  template <Uint32 buttonmask>
  binding_type mouse_button() {
    return std::make_unique<details::mouse_button_binding<buttonmask>>();
  }
}

input::binding_type operator|(
  input::binding_type a,
  input::binding_type b
) {
  return std::make_unique<input::details::binding_or_combinator>(
    std::move(a),
    std::move(b)
  );
}

input::binding_type operator&(
  input::binding_type a,
  input::binding_type b
) {
  return std::make_unique<input::details::binding_and_combinator>(
    std::move(a),
    std::move(b)
  );
}
```

Operator overloading allows me to write such expressions:

- `input::key<SDL_SCANCODE_LEFT>() | input::key<SDL_SCANCODE_A>()`
- `input::key<SDL_SCANCODE_RIGHT>() | input::key<SDL_SCANCODE_D>()`
- `input::key<SDL_SCANCODE_UP>() | input::key<SDL_SCANCODE_W>()`
- `input::key<SDL_SCANCODE_DOWN>() | input::key<SDL_SCANCODE_S>()`

This should cover the **trigger** actions and the **axis** actions.

For the **passthrough** action, it is a bit more complicated. We may want to pass through different type of values (a `float`, a `vec2`, etc…). For each of those type of values, we may want to read them from a specific input source.

To achieve this, we’ll use a template class to represent a source and how to read from it:

```
#include <variant>

namespace input {
  template <typename T>
  class passthrough_source {
    static_assert(sizeof(T) == 0, "unsupported passthrough data type");

    // specializations must define:
    public:
      using type = // ...

      static void read(
        type source,
        entt::registry &registry,
        const state &state,
        T& out
      ) {}    
  };
}
```

We will then specialize this template for each type of value we want to read, let’s see how I implemented the `vec2` source:

```
namespace input {
  template <>
  class passthrough_source<math::vec2> {
    public:
      // tag types to identify the source
      struct mouse_pointer {};
      struct mouse_delta {};
      struct mouse_wheel {};

      using type = std::variant<
        mouse_pointer,
        mouse_delta,
        mouse_wheel
      >;

    public:
      static void read(
        type source,
        entt::registry &registry,
        const state &state,
        math::vec2& out
      ) {
        std::visit(
          [&](auto &src) { read_source(src, registry, state, out); },
          source
        );
      }

    private:
      static void read_source(
        mouse_pointer source,
        entt::registry &registry,
        const state &state,
        math::vec2& out
      ) {
        auto &io = registry.ctx().get<ImGuiIO &>();
        if (!io.WantCaptureMouse) {
          out.x = state.mouse_x;
          out.y = state.mouse_y;
        }
      }

      static void read_source(
        mouse_delta source,
        entt::registry &registry,
        const state &state,
        math::vec2& out
      ) {
        auto &io = registry.ctx().get<ImGuiIO &>();
        if (!io.WantCaptureMouse) {
          out.x = state.mouse_motion_x;
          out.y = state.mouse_motion_y;
        }
      }

      static void read_source(
        mouse_wheel source,
        entt::registry &registry,
        const state &state,
        math::vec2& out
      ) {
        auto &io = registry.ctx().get<ImGuiIO &>();
        if (!io.WantCaptureMouse) {
          out.x = state.mouse_wheel_x;
          out.y = state.mouse_wheel_y;
        }
      }
  };
```

It becomes easy to extend this feature to other data types, or other input sources. When we want to read from it, we can just call the following snippet:

```
math::vec2 value;

input::passthrough_source<math::vec2>::read(
  input::passtrhough_source<math::vec2>::mouse_pointer,
  registry,
  input_state,
  value
);
```

Yes it’s a bit verbose, but we can use `using` to type less code later if it’s really needed 🙂

For convenience, I’ll define a type alias for each source type:

```
namespace input {
  using passthrough_vec2_source_type = typename passthrough_source<math::vec2>::type;
  // ...
}
```

---

## Configuring the game controls

In the game systems, I want to be able to query inputs via a name, an `std::string` will do the trick for now.

For each named control, we’ll need to store the bindings and the `action_*` structures. So we define a context structures that holds both information:

```
namespace input {
  /* forward */ class manager;

  class controls {
    friend class manager;

    public:
      struct trigger_context {
        action_trigger value;
        binding_type bindings;
      };

      struct axis_context {
        action_axis value;
        binding_type left_bindings;
        binding_type right_bindings;
        binding_type up_bindings;
        binding_type down_bindings;
      };

      template <typename T>
      struct passthrough_context {
        using source_type = typename passthrough_source<T>::type;

        action_passthrough<T> value;
        source_type source;
      };

      // ...
  };
}
```

Then, we’ll use a map to associate those structures to a name:

```
namespace input {
  class controls {
    // ...

    private:
      std::unordered_map<std::string, trigger_context> m_triggers;
      std::unordered_map<std::string, axis_context> m_axis;
      std::unordered_map<std::string, passthrough_context<math::vec2> m_passthrough_vec2;

    public:
      void add_trigger(const std::string &name, binding_type bindings);
      void add_axis(
        const std::string &name,
        binding_type left_bindings,
        binding_type right_bindings,
        binding_type up_bindings,
        binding_type down_bindings
      );
      void add_passthrough_vec2(
        const std::string &name,
        passthrough_vec2_source_type source
      );
  }
}
```

Since we’re using `std::unique_ptr<T>` for the bindings, we need to use `std::move()` in the implementation:

```
namespace input {
  void controls::add_trigger(const std::string &name, binding_type bindings) {
    m_triggers.emplace(name, trigger_context{
      .bindings = std::move(bindings)
    });
  }

  void controls::add_axis(
    const std::string &name,
    binding_type left_bindings,
    binding_type right_bindings,
    binding_type up_bindings,
    binding_type down_bindings
  ) {
    m_axis.emplace(name, axis_context{
      .left_bindings  = std::move(left_bindings),
      .right_bindings = std::move(right_bindings),
      .up_bindings    = std::move(up_bindings),
      .down_bindings  = std::move(down_bindings)
    });
  }

  void controls::add_passthrough_vec2(
    const std::string &name,
    passthrough_vec2_source_type source
  ) {
    m_passthrough_vec2.emplace(name, passthrough_context<math::vec2>{
      .source = source
    });
  }
}
```

This also means that the `controls` class is not copyable. So we should probably delete that constructor just in case:

```
namespace input {
  class controls {
    public:
      controls() = default;
      controls(const controls &) = delete;

    // ...
  };
}
```

Finally, we can define our `manager` class, which will owns the current state and our configured controls:

```
namespace input {
  class manager {
    private:
      controls m_controls;
      state m_state;

    public:
      // the manager is not copyable as well, because of m_controls
      manager() = default;
      manager(const manager &) = delete;

      void configure(std::function<void(controls &> fn) {
        fn(m_controls);
      }

      // see above for the implementation of those methods
      void process_event(SDL_Event *event);
      void frame_begin();
      void update(entt::registry &registry);

      // ...
  };
}
```

When initializing our game loop, we allocate the manager into a shared pointer that we add to the EnTT registry’s context:

```
auto input_manager = std::make_shared<input::manager>();
registry.ctx().emplace<std::shared_ptr<input::manager>>(input_manager);
```

When configuring the controls, we can simply get the input manager from the registry and call the `configure()` method:

```
auto input_manager = registry.ctx().get<std::shared_ptr<input::manager>>();

input_manager->configure([](input::controls &controls) {
  controls.add_trigger(
    "ui_click",
    input::mouse_button<SDL_BUTTON_LEFT>()
  );
  controls.add_passthrough_vec2(
    "pointer",
    input::passthrough_source<math::vec2>::mouse_pointer{}
  );
  controls.add_trigger(
    "debug_toggle_inspector",
    input::key<SDL_SCANCODE_BACKSPACE>()
  );
  controls.add_axis(
    "camera_pan",
    input::key<SDL_SCANCODE_LEFT>()  | input::key<SDL_SCANCODE_A>(),
    input::key<SDL_SCANCODE_RIGHT>() | input::key<SDL_SCANCODE_D>(),
    input::key<SDL_SCANCODE_UP>()    | input::key<SDL_SCANCODE_W>(),
    input::key<SDL_SCANCODE_DOWN>()  | input::key<SDL_SCANCODE_S>()
  );
  controls.add_passthrough_vec2(
    "camera_zoom",
    input::passthrough_source<math::vec2>::mouse_wheel{}
  );
});
```

---

## Querying the manager

The manager will expose the values stored in the `controls` class maps by reference. The user only needs to provide the name of the action:

```
namespace input {
  class manager {
    // ...

    public:
      const action_trigger &read_trigger(const std::string &name) {
        return m_controls.m_triggers.at(name).value;
      }

      const action_axis &read_axis(const std::string &name) {
        return m_controls.m_axis.at(name).value;
      }

      const action_passthrough<math::vec2> &read_passthrough_vec2(const std::string &name) {
        return m_controls.m_passthrough_vec2.at(name).value;
      }
  };
}
```

Bound checking is done by the map’s `at()` method, so if an action does not exist, an `std::out_of_range` exception will be thrown. If this happens, this means I forgot to define the action and I probably want to crash during testing anyway. In Rust I would `.unwrap()` the `Option<T>` as well.

Finally, in my game systems, I can query the input actions. As an example, here is my camera controller system’s implementation:

```
void camera_controller_system::run(entt::registry &registry) {
  auto input_manager = registry.ctx().get<std::shared_ptr<input::manager>>();
  auto pan_action_value = input_manager->read_axis("camera_pan");
  auto zoom_action_value = input_manager->read_passthrough_vec2("camera_zoom");
  auto ui_click_action_value = input_manager->read_trigger("ui_click");
  auto pointer_action_value = input_manager->read_passthrough_vec2("pointer");

  auto view = registry.view<
    components::camera,
    components::strategic_camera
  >();

  for (auto entity : view) {
    auto &camera = view.get<components::camera>(entity);
    auto &scamera = view.get<components::strategic_camera>(entity);

    if (scamera.drag_position.has_value()) {
      if (ui_click_action_value.cancelled) {
        scamera.drag_position = std::nullopt;
      }
      else {
        auto new_pos   = pointer_action_value.raw;
        auto delta_pos = scamera.drag_position.value() - new_pos;
        auto motion = delta_pos * (1.0/scamera.zoom);
        scamera.target_position = scamera.target_position + motion;
        scamera.drag_position = new_pos;
      }
    }
    else if (ui_click_action_value.performed) {
      scamera.drag_position = pointer_action_value.raw;
    }
    else {
      if (zoom_action_value.raw.y != 0) {
        auto zoom_sign = zoom_action_value.raw.y / std::abs(zoom_action_value.raw.y);
        scamera.zoom = math::clamp(
          scamera.zoom + zoom_sign * m_zoom_step,
          m_min_zoom,
          m_max_zoom
        );
      }

      auto pan_direction = math::vec2(pan_action_value.x, pan_action_value.y);
      auto scaled_speed = m_pan_speed * (1.0/scamera.zoom);
      auto motion = pan_direction * scaled_speed;
      scamera.target_position = scamera.target_position + motion;

      auto current_size = math::vec2(camera.view.w, camera.view.h);
      auto new_size = math::lerp(current_size, scamera.original_size * scamera.zoom, 0.2);

      camera.view.w = new_size.x;
      camera.view.h = new_size.y;
    }

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

---

## Updating the action values

So far, our action values are empty. We’ll fix this in the `manager`‘s `update()` method:

```
namespace input {
  void manager::update(entt::registry &registry) {
    // ...

    // update all the trigger actions
    for (auto &[_, action_context] : m_controls.m_triggers) {
      bool was_active = action_context.value.active;
      action_context.value.active    = action_context.bindings->check(registry, m_state);
      action_context.value.performed =  action_context.value.active && !was_active;
      action_context.value.cancelled = !action_context.value.active &&  was_active;
    }

    // update all the axis actions
    for (auto &[_, action_context] : m_controls.m_axis) {
      action_context.value.x = 0;
      action_context.value.y = 0;

      if (action_context.left_bindings->check(registry, m_state)) {
        action_context.value.x -= 1;
      }

      if (action_context.right_bindings->check(registry, m_state)) {
        action_context.value.x += 1;
      }

      if (action_context.up_bindings->check(registry, m_state)) {
        action_context.value.y -= 1;
      }

      if (action_context.down_bindings->check(registry, m_state)) {
        action_context.value.y += 1;
      }
    }

    // update all the passthrough actions
    for (auto &[_, action_context] : m_controls.m_passthrough_vec2) {
      passthrough_source<math::vec2>::read(
        action_context.source,
        registry,
        m_state,
        action_context.value.raw
      );
    }
  }
}
```

---

## Conclusion

This input manager was inspired by Unity’s input system. Despite being a bit verbose, it is very easy to extend and serves its purpose perfectly.

I really got to love `std::variant` and `std::visit()`, they are a powerful tool to write type-safe code without dwelling into black magic templates. Though, I have to admit, I still prefer Rust’s enums and pattern matching, it makes the code easier to read IMHO.

It was a very long time (17 years?) since I got bitten by the slicing problem, so long that I completely forgot about it.

Having worked for years with languages like Python, C#, Javascript (ES6\+), it was really a surprise followed by “ah yes, I remember that C\+\+ is weird sometimes”.

This is the reason why I’m not fond of inheritance, in Rust I would have written a trait and used something like `Box<dyn Trait>` (which is exactly the same as my `std::unique_ptr<T>` but there is no ambiguity about the intent).

Despite all this, I still enjoy C\+\+20 for this project. The STL has come a long way since the days of C\+\+03, and there are more exciting things coming in for C\+\+23.

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
