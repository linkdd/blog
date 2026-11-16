---
title: "Roll your own in-game UI with Clay and NanoVG (part 1)"
date: "2025-07-08T13:36:15Z"
description: "Introduction   Most games need a UI. Most game engines provide a framework to build it. For..."
tags: ["gamedev", "cpp", "programming", "tutorial"]
author: "David Delassus"
---

## Introduction

Most games need a UI. Most game engines provide a framework to build it. For example, Unity has UI Toolkit. But what if you're making your game from scratch with just C++ and OpenGL, what are you left with?

Most people will mention the venerable **[Dear ImGui](https://github.com/ocornut/imgui/)**. It is a fine library and I use it to write the *Debug UI* which includes a scene editor, a profiler, etc...

But for an *in-game UI*, it has that "developer UI style" that does not quite feel good in my opinion. You can style it extensively of course, but we'll explore in this series an alternative.

Have you heard of **[Clay](https://github.com/nicbarker/clay)**? It is a UI layout library, written in C, that take inspirations from CSS Flexbox. Have a little taste:

```cpp
void my_layout() {
  Clay_ElementDeclaration root_config;
  root_config.id                   = CLAY_ID("OuterContainer");
  root_config.layout.sizing.width  = CLAY_SIZING_GROW(0, 0);
  root_config.layout.sizing.height = CLAY_SIZING_GROW(0, 0);
  root_config.layout.padding       = CLAY_PADDING_ALL(16);
  root_config.layout.childGap      = 16;

  Clay_ElementDeclaration sidebar_config;
  sidebar_config.id                   = CLAY_ID("SideBar");
  sidebar_config.layout.sizing.width  = CLAY_SIZING_FIXED(300);
  sidebar_config.layout.sizing.height = CLAY_SIZING_GROW(0, 0);
  sidebar_config.layout.padding       = CLAY_PADDING_ALL(16);
  sidebar_config.layout.childGap      = 16;
  sidebar_config.backgroundColor      = {255, 255, 255, 255};

  Clay_ElementDeclaration item_config;
  item_config.layout.sizing.width  = CLAY_SIZING_GROW(0, 0);
  item_config.layout.sizing.height = CLAY_SIZING_FIXED(32);
  item_config.layout.padding       = CLAY_PADDING_ALL(8);
  item_config.backgroundColor      = {255, 0, 0, 255};

  Clay_TextElementConfig item_label_config;
  item_label_config.fontSize  = 16;
  item_label_config.textColor = {255, 255, 255, 255};

  CLAY(root_config) {
    CLAY(sidebar_config) {
      my_item(
        CLAY_ID("Item 1"),
        CLAY_STRING("Item 1"),
        item_config,
        item_label_config
      );
      my_item(
        CLAY_ID("Item 2"),
        CLAY_STRING("Item 2"),
        item_config,
        item_label_config
      );
      my_item(
        CLAY_ID("Item 3"),
        CLAY_STRING("Item 3"),
        item_config,
        item_label_config
      );
    }
  }
}

void my_item(
  Clay_ElementId id,
  Clay_String label,
  Clay_ElementDeclaration base_config,
  Clay_TextElementConfig label_config
) {
  Clay_ElementDeclaration element_config = base_config;
  element_config.id = id;

  CLAY(element_config) {
    CLAY_TEXT(label, CLAY_TEXT_CONFIG(label_config));
  }
}
```

As you can see, we can separate easily the "styling" from the UI declaration. We can use normal functions as more "high level components". It's quite straightforward.

Clay does not come with a renderer, its job is simply layout. You have to roll your own, and this is what this series of article will be about.

---

## Table of Content

 - **Part 1: GLFW, EnTT, and an input system**
 - [Part 2: Writing a NanoVG renderer for Clay](/posts/2025-07-09-roll-your-own-in-game-ui-with-clay-and-nanovg-part-2)
 - Part 3: A text input component with stb_textedit

---

### 1. Setting up the project

I will go over this step quickly, as there is a lot of resources online on how to do this.

In this series, I will use **[GLFW](https://glfw.org/)** to create a window and handle inputs, and **[EnTT](https://github.com/skypjack/entt)** for state management.

In my game, I use **EnTT** quite a lot, for its Entity Component System, but not only, as it supports:

- resource management
- observer pattern & event/signal dispatching
- ...

I often use it as an arena, using the `entt::registry`'s "context", to store shared state and access it from anywhere, allowing me to pass only the registry itself.

I won't go into details of the build system (I use CMake with `FetchContent` and build statically) and will omit error handling. If you are reading this, I assume you already have a game, or a prototype, or a work-in-progress project. And there are plenty of resources online to get up and running, so I'll just paste a placeholder code:

```cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp>

#include <entt/entt.hpp>

int main() {
  auto registry = entt::registry{};

  glfwInit(); // error handling omitted

  glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
  glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 6);
  glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

  auto window = glfwCreateWindow(1366, 768, "RYO-UI", nullptr, nullptr);
  // error handling omitted

  glfwMakeContextCurrent(window);
  glfwSwapInterval(1);

  gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);
  // error handling omitted

  glfwSetWindowUserPointer(window, &registry);
  glfwSetFramebufferSizeCallback(window, configure_framebuffer);

  init(registry);

  auto prev_frame_time = glfwGetTime();

  while (!glfwWindowShouldClose(window)) {
    auto current_time = glfwGetTime();
    auto delta_time   = current_time - prev_frame_time;
    prev_frame_time   = current_time;

    begin_frame(registry);
    update(registry, delta_time);
    render(registry);
    end_frame(registry);
  }

  teardown(registry);

  glfwDestroyWindow(window);
  glfwTerminate();

  return 0;
}


void configure_framebuffer(GLFWwindow* win, int width, int height) {
  glViewport(0, 0, width, height);

  // optional: configure your FBO, RBO, etc... here
}


void begin_frame(entt::registry& registry) {
  glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
  glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
}


void end_frame(entt::registry& registry) {
  glfwSwapBuffers(glfwGetCurrentContext());
  glfwPollEvents();
}


void init(entt::registry& registry) {
  // your game initialization:
  //  - loading assets
  //  - generating maps
  //  - ...
}


void teardown(entt::registry& registry) {
  // your game teardown
}


void update(entt::registry& registry, double delta_time) {
  // your game logic
}


void render(entt::registry& registry) {
  // your game scene rendering pipeline
}
```

Standard stuff so far.

### 2. An input system

Your game will have to handle inputs, but so does your UI. And the two must not have conflicts. A click in your UI should not "go through" and do something in your scene.

This is why the first step of this series is to work on the input system.

In fact, couple years ago, I wrote about this [here](/posts/2023-01-25-decoupling-input-bindings-from-game-systems-with-c-sdl), it was for SDL, but I ported it to GLFW with some minor improvements.

#### 2.1. Input Actions

The idea is to define "actions", and associate "bindings" to them. Then the input system plugs itself in GLFW to update its internal state. Whenever you want, you can read the value of any defined "action".

There are 3 types of actions:

- **trigger:** like a mouse click, it has 3 states:
    - *active:* the action is currently being done
    - *performed:* the action started this frame
    - *cancelled:* the action ended this frame
- **axis:** like a joystick or the keyboard arrow keys, it exposes a `glm::vec2`
- **passthrough:** like the mouse position, or current delta time, it exposes the raw value of the underlying *source*

We will use the following structures to store those states:

```cpp
namespace engine::input {
  struct action_trigger {
    bool active;
    bool performed;
    bool cancelled;
  };

  struct action_axis {
    glm::vec2 value;
  };

  struct action_passthrough_point {
    glm::vec2 value;
  };

  struct action_passthrough_time {
    double value;
  };
}
```

But to be able to compute their state, we first need to get inputs from GLFW.

#### 2.2. The internal input state

We need to keep track of:

- which keys or mouse buttons are currently pressed
- the current mouse position, motion and scroll
- the current time and delta time

And in the future, for the UI text inputs, we will need a buffer of the keys that have been pressed since the last frame, as well as a buffer of unicode codepoints that were typed since the last frame.

```cpp
namespace engine::input {
  struct state {
    // used by "triggers" to determine which action is active
    std::unordered_map<int, bool> keys;
    std::unordered_map<int, bool> mouse_buttons;

    // will be used by our UI text inputs
    std::vector<std::pair<uint16_t, uint16_t>> key_buffer;
    std::vector<uint32_t>                      char_buffer;

    // used by "passthrough" actions
    glm::vec2 mouse_position;
    glm::vec2 mouse_motion;
    glm::vec2 mouse_wheel;

    double current_time;
    double delta_time;

    // used to ignore actions, usually set when computing the UI
    bool capture_keyboard{false};
    bool capture_mouse{false};

    // GLFW callback handlers
    static void on_key(GLFWwindow* window, int key, int scancode, int action, int mods);
    static void on_char(GLFWwindow* window, unsigned int codepoint);
    static void on_mouse_button(GLFWwindow* window, int button, int action, int mods);
    static void on_mouse_move(GLFWwindow* window, double xpos, double ypos);
    static void on_mouse_scroll(GLFWwindow* window, double xoffset, double yoffset);
  };
}
```

We will store that structure in the registry's context, and then setup the  GLFW callbacks:

```diff
-  glfwSetWindowUserPointer(window, &registry);
-  glfwSetFramebufferSizeCallback(window, configure_framebuffer);
+  auto& input_state = registry.ctx().emplace<engine::input::state>();
+
+  glfwSetWindowUserPointer(window, &registry);
+
+  glfwSetFramebufferSizeCallback(window, configure_framebuffer);
+  glfwSetKeyCallback            (window, engine::input::state::on_key);
+  glfwSetCharCallback           (window, engine::input::state::on_char);
+  glfwSetMouseButtonCallback    (window, engine::input::state::on_mouse_button);
+  glfwSetCursorPosCallback      (window, engine::input::state::on_mouse_move);
+  glfwSetScrollCallback         (window, engine::input::state::on_mouse_scroll);
```

Let's update our game loop as well to feed it the remaining values:

```diff
   while (!glfwWindowShouldClose(window)) {
-    auto current_time = glfwGetTime();
-    auto delta_time   = current_time - prev_frame_time;
-    prev_frame_time   = current_time;
+    input_state.current_time = glfwGetTime();
+    input_state.delta_time   = input_state.current_time - prev_frame_time;
+    prev_frame_time          = input_state.current_time;

     begin_frame(registry);
     update(registry, delta_time);
     render(registry);
+
+    input_state.key_buffer.clear();
+    input_state.char_buffer.clear();
+
     end_frame(registry);
   }
```

Now, we can implement the callbacks:

```cpp
namespace engine::input {
  void state::on_key(GLFWwindow* window, int key, int scancode, int action, int mods) {
    (void)scancode;

    auto* registry = static_cast<entt::registry*>(glfwGetWindowUserPointer(window));
    auto& state    = registry->ctx().get<engine::input::state>();

    state.keys[key] = (action == GLFW_PRESS || action == GLFW_REPEAT);

    if (action == GLFW_PRESS || action == GLFW_REPEAT) {
      state.key_buffer.emplace_back(key, mods);
    }
  }


  void state::on_char(GLFWwindow* window, unsigned int codepoint) {
    auto* registry = static_cast<entt::registry*>(glfwGetWindowUserPointer(window));
    auto& state    = registry->ctx().get<engine::input::state>();

    state.char_buffer.push_back(codepoint);
  }


  void state::on_mouse_button(GLFWwindow* window, int button, int action, int mods) {
    (void)mods;

    auto* registry = static_cast<entt::registry*>(glfwGetWindowUserPointer(window));
    auto& state    = registry->ctx().get<engine::input::state>();

    state.mouse_buttons[button] = (action == GLFW_PRESS || action == GLFW_REPEAT);
  }

  void state::on_mouse_move(GLFWwindow* window, double xpos, double ypos) {
    auto* registry = static_cast<entt::registry*>(glfwGetWindowUserPointer(window));
    auto& state    = registry->ctx().get<engine::input::state>();

    auto mpos = glm::vec2{xpos, ypos};

    // if you are using a FBO, or rendering to an ImGui window,
    // you might need to scale and translate the `mpos` vector

    state.mouse_motion   = mpos - state.mouse_position;
    state.mouse_position = mpos;
  }

  void state::on_mouse_scroll(GLFWwindow* window, double xoffset, double yoffset) {
    auto* registry = static_cast<entt::registry*>(glfwGetWindowUserPointer(window));
    auto& state    = registry->ctx().get<engine::input::state>();

    state.mouse_wheel = glm::vec2{xoffset, yoffset};
  }
}
```

#### 2.3. Input bindings

Bindings are in essence, simple functions that accept the internal state as parameter, and returns `true` or `false` whether they detected the key or mouse button being pressed.

They become powerful once we can combine them using `|` (OR) and `&` (AND).

This is the interface we will define:

```cpp

namespace engine::input::internal {
  class binding_check {
    public:
      virtual ~binding_check() = default;

      virtual bool check(const state& state, bool ignore_capture) const {
        (void)state;
        (void)ignore_capture;
        return false;
      }
  };
}
```

> **NB:** The `ignore_capture` parameter will be used by the UI to ignore the value of `capture_mouse` and `capture_keyboard`. The UI will define input actions, but will also "capture" the mouse and keyboard to avoid clicks or typing text in the UI going through to the game scene. Without that flag, the UI actions would keep toggling between "active" and "inactive" every frame.

The OR and AND combinators will take as parameters 2 other bindings. In order to be able to call the correct virtual method, we need to store them as pointers or references to avoid [object slicing](https://en.wikipedia.org/wiki/Object_slicing).

If we were to use references, this would force the user to keep track of every "sub binding" and their lifetime. To simplify the API, we'll use a pointer, an `std::unique_ptr` in fact, so that it gets destroyed automatically (thank you RAII):

```cpp
namespace engine::input::internal {
  class binding_or_combinator final : public binding_check {
    public:
      binding_or_combinator(
        std::unique_ptr<binding_check> lhs,
        std::unique_ptr<binding_check> rhs
      ) : m_lhs{std::move(lhs)},
          m_rhs{std::move(rhs)}
      {}

      bool check(const state& state, bool ignore_capture) const override {
        return m_lhs->check(state, ignore_capture) || m_rhs->check(state, ignore_capture);
      }

    private:
      std::unique_ptr<binding_check> m_lhs;
      std::unique_ptr<binding_check> m_rhs;
  };

  class binding_and_combinator final : public binding_check {
    public:
      binding_and_combinator(
        std::unique_ptr<binding_check> lhs,
        std::unique_ptr<binding_check> rhs
      ) : m_lhs{std::move(lhs)},
          m_rhs{std::move(rhs)}
      {}

      bool check(const state& state, bool ignore_capture) const override {
        return m_lhs->check(state, ignore_capture) && m_rhs->check(state, ignore_capture);
      }

    private:
      std::unique_ptr<binding_check> m_lhs;
      std::unique_ptr<binding_check> m_rhs;
  };
}
```

Now that we have our combinators, we still need a check for a key and a check for a mouse button. This can be done with 2 more inheritances:

```cpp
namespace engine::input::internal {
  class key_binding final : public binding_check {
    public:
      key_binding(int key) : m_key{key} {}

      bool check(const state& state, bool ignore_capture) const override {
        if (state.keys.contains(m_key) {
          auto pressed = state.keys.at(m_key);
          return (ignore_capture || !state.capture_keyboard) && pressed;
        }

        return false;
      }

    private:
      int m_key;
  };

  class mouse_button_binding final : public binding_check {
    public:
      mouse_button_binding(int button) : m_button{button} {}

      bool check(const state& state, bool ignore_capture) const override {
        if (state.mouse_buttons.contains(m_button)) {
          auto pressed = state.mouse_buttons.at(m_key);
          // if rendering inside an ImGui window, or using a FBO,
          // you might want to check that the button was pressed **inside** your viewport
          return (ignore_capture || !state.capture_mouse) && pressed;
        }

        return false;
      }

    private:
      int m_button;
  };
}
```

To finalize our API, we only need some factories and operator overloading:

```cpp
namespace engine::input {
  using binding_type = std::unique_ptr<internal::binding_check>;

  binding_type key(int key) {
    return std::make_unique<internal::key_binding>(key);
  }

  binding_type mouse_button(int button) {
    return std::make_unique<internal::mouse_button_binding>(button);
  }
}

engine::input::binding_type operator|(
  engine::input::binding_type lhs,
  engine::input::binding_type rhs
) {
  return std::make_unique<engine::input::internal::binding_or_combinator>(
    std::move(lhs), std::move(rhs)
  );
}

engine::input::binding_type operator&(
  engine::input::binding_type lhs,
  engine::input::binding_type rhs
) {
  return std::make_unique<engine::input::internal::binding_and_combinator>(
    std::move(lhs), std::move(rhs)
  );
}
```

We can now create complex bindings quite easily:

```cpp
using namespace engine::input;

auto copy = (key(GLFW_KEY_LEFT_CONTROL) | key(GLFW_KEY_RIGHT_CONTROL)) & key(GLFW_KEY_C);
```

#### 2.4. Input Passthrough

The "passthrough" kind of action is a simple function that returns a value from the internal state, and `true` or `false` whether the action was ignored.

We need to identify the "source" of the action and the type of value we are passing through:

 - for mouse events, we are passing through a `glm::vec2`, we have 3 sources:
    - position
    - motion
    - scroll
 - for time events, we are passing through a `double`, we have 2 sources:
    - current time
    - delta time

As you could see earlier, we got 2 type of actions for each type of value we can passthrough: `action_passthrough_point` and `action_passthrough_time`.

Similarly, we'll have 2 helper classes to read those values. Each will expose a sum-type to identify the source.

Let's begin with the `passthrough_point_source`:

```cpp
namespace engine::input {
class passthrough_point_source {
    public:
      struct mouse_pointer {};
      struct mouse_delta {};
      struct mouse_wheel {};

      using type = std::variant<mouse_pointer, mouse_delta, mouse_wheel>;

    public:
      static bool read(const state& state, bool ignore_capture, glm::vec2& out, type source) {
        return std::visit(
          [&](auto& src) { return read(state, ignore_capture, out, src); },
          source
        );
      }

    private:
      static bool read(const state& state, bool ignore_capture, glm::vec2& out, mouse_pointer) {
        if (ignore_capture || !state.capture_mouse) {
          out = state.mouse_position;
          return true;
        }

        return false;
      }

      static bool read(const state& state, bool ignore_capture, glm::vec2& out, mouse_delta) {
        if (ignore_capture || !state.capture_mouse) {
          out = state.mouse_motion;
          return true;
        }

        return false;
      }

      static bool read(const state& state, bool ignore_capture, glm::vec2& out, mouse_wheel)  {
        if (ignore_capture || !state.capture_mouse) {
          out = state.mouse_wheel;
          return true;
        }

        return false;
      }
  };
}
```

> **NB:** The reason behind using `std::variant` and `std::visit` instead of an `enum class` might be a case of YAGNI (You Ain't Gonna Need It). In the future, I may identify a source that requires some parameters. An `enum class` cannot hold data, but a sum-type can. This pattern is heavily influenced by my previous experience with functional languages such as Haskell (or even Rust to some extent). Feel free to adapt the code to your likings.

And now, the `passthrough_time_source`:

```cpp
namespace engine::input {
  class passthrough_time_source {
    public:
      struct current_time {};
      struct delta_time {};

      using type = std::variant<current_time, delta_time>;

    public:
      static bool read(const state& state, double& out, type source) {
        return std::visit(
          [&](auto& src) { return read(state, out, src); },
          source
        );
      }

    private:
      static bool read(const state& state, double& out, current_time) {
        out = state.current_time;
        return true;
      }

      static bool read(const state& state, double& out, delta_time) {
        out = state.delta_time;
        return true;
      }
  };
}
```

#### 2.5. Defining control maps

We now have all the tools to map bindings to actions, what I call a "control map".

Each "control" will be uniquely identified, using `entt::id_type`, which can be computed from a compile-time hashed string:

```cpp
using namespace entt::literals;

constexpr entt::id_type MY_INPUT_ACTION = "my_input_action"_hs;
```

The code is quite straightforward:

```cpp
namespace engine::input {
  class controls {
    public:
      // first, we define the arguments to be used for our `define` methods
      struct trigger {
        using action_type = action_trigger;

        bool ignore_capture{false};

        binding_type bindings;
      };

      struct axis {
        using action_type = action_axis;

        bool ignore_capture{false};

        binding_type left_bindings;
        binding_type right_bindings;
        binding_type up_bindings;
        binding_type down_bindings;
      };

      struct passthrough_point {
        using action_type = action_passthrough_point;

        bool ignore_capture{false};

        passthrough_point_source::type source_type;
      };

      struct passthrough_time {
        using action_type = action_passthrough_time;

        passthrough_time_source::type source_type;
      };

    public:
      // will be defined later
      static controls& main();
      void update(entt::registry& registry);
      // ---

      // the API to add controls to our control maps
      void define(entt::id_type id, trigger control) {
        m_triggers.emplace(
          id,
          std::make_pair(
            std::move(control),
            trigger::action_type{}
          )
        );
      }

      void define(entt::id_type id, axis control) {
        m_axes.emplace(
          id,
          std::make_pair(
            std::move(control),
            axis::action_type{}
          )
        );
      }

      void define(entt::id_type id, passthrough_point control) {
        m_passthrough_points.emplace(
          id,
          std::make_pair(
            std::move(control),
            passthrough_point::action_type{}
          )
        );
      }

      void define(entt::id_type id, passthrough_time control) {
        m_passthrough_times.emplace(
          id,
          std::make_pair(
            std::move(control),
            passthrough_time::action_type{}
          )
        );
      }

      // the API to actually read our actions
      action_trigger read_trigger(entt::id_type id) const {
        if (m_triggers.contains(id)) {
          auto& [_, action] = m_triggers.at(id);
          return action;
        }

        return action_trigger{false, false, false};
      }

      action_axis read_axis(entt::id_type id) const {
        if (m_axes.contains(id)) {
          auto& [_, action] = m_axes.at(id);
          return action;
        }

        return action_axis{glm::vec2{0.0f, 0.0f}};
      }

      action_passthrough_point read_passthrough_point(entt::id_type id) const {
        if (m_passthrough_points.contains(id)) {
          auto& [_, action] = m_passthrough_points.at(id);
          return action;
        }

        return action_passthrough_point{glm::vec2{0.0f, 0.0f}};
      }

      action_passthrough_time read_passthrough_time(entt::id_type id) const {
        if (m_passthrough_times.contains(id)) {
          auto& [_, action] = m_passthrough_times.at(id);
          return action;
        }

        return action_passthrough_time{0.0};
      }

    private:
      template <typename T>
      using control_map = std::unordered_map<
        entt::id_type,
        std::pair<T, typename T::action_type>
      >;

      control_map<trigger> m_triggers;
      control_map<axis> m_axes;
      control_map<passthrough_point> m_passthrough_points;
      control_map<passthrough_time> m_passthrough_times;
  };
}
```

We need `std::move` because the configuration has `binding_type` fields which, if you don't remember, are `std::unique_ptr`, and therefore not copyable.

If you noticed, there is a `static controls& main();` method, making this class a singleton that can be accessed anywhere. It's entirely optional of course, you can choose to not have it if you prefer to avoid singletons. I do prefer it this way though, for the ease of use. I use **EnTT**'s service locator feature to implement it:

```cpp
namespace engine::input {
  controls& controls::main() {
    if (!entt::locator<controls>::has_value()) {
      entt::locator<controls>::emplace();
    }

    return entt::locator<controls>::value();
  }
}
```

#### 2.6. The last straw: computing our action states

This is the last part of our input system, and it all happens in the `update()` method of the `controls` class. In this method, we go through all our control maps, and use the internal input state to set the action states:

```cpp
namespace engine::input {
  void controls::update(entt::registry& registry) {
    auto& state = registry.ctx().get<engine::input::state>();

    for (auto& [id, control] : m_triggers) {
      auto& [trigger, action] = control;

      bool was_active  = action.active;
      action.active    = trigger.bindings->check(state, trigger.ignore_capture);
      action.performed = action.active && !was_active;
      action.cancelled = !action.active && was_active;
    }

    for (auto& [id, control] : m_axes) {
      auto& [axis, action] = control;

      action.value.x = 0.0f;
      action.value.y = 0.0f;

      if (axis.left_bindings->check(state, axis.ignore_capture)) {
        action.value.x += 1.0f;
      }

      if (axis.right_bindings->check(state, axis.ignore_capture)) {
        action.value.x -= 1.0f;
      }

      if (axis.up_bindings->check(state, axis.ignore_capture)) {
        action.value.y += 1.0f;
      }

      if (axis.down_bindings->check(state, axis.ignore_capture)) {
        action.value.y -= 1.0f;
      }
    }

    for (auto& [id, control] : m_passthrough_points) {
      auto& [point, action] = control;

      if (!passthrough_point_source::read(state, point.ignore_capture, action.value, point.source_type)) {
        action.value = glm::vec2{0.0f, 0.0f};
      }
    }

    for (auto& [id, control] : m_passthrough_times) {
      auto& [time, action] = control;

      if (!passthrough_time_source::read(state, action.value, time.source_type)) {
        action.value = 0.0;
      }
    }
  }
}
```

All we need to do now, is call that method in our game loop:

```diff
     begin_frame(registry);
+    engine::input::controls::main().update(registry);
     update(registry, delta_time);
     render(registry);

     input_state.key_buffer.clear();
     input_state.char_buffer.clear();

     end_frame(registry);
```

### 3. Usage

Our input system is now complete, and can be used quite easily. You can use it to define your game controls, for example:

```cpp
void init(entt::registry& registry) {
  engine::input::controls::main().define(
    "pan"_hs,
    engine::input::controls::axis{
      .left_bindings  = engine::input::key(GLFW_KEY_LEFT)  | engine::input::key(GLFW_KEY_A),
      .right_bindings = engine::input::key(GLFW_KEY_RIGHT) | engine::input::key(GLFW_KEY_D),
      .up_bindings    = engine::input::key(GLFW_KEY_UP)    | engine::input::key(GLFW_KEY_W),
      .down_bindings  = engine::input::key(GLFW_KEY_DOWN)  | engine::input::key(GLFW_KEY_S)
    }
  );

  engine::input::controls::main().define(
    "point"_hs,
    engine::input::controls::passthrough_point{
      .source_type = engine::input::passthrough_point_source::mouse_pointer{},
    }
  );

  engine::input::controls::main().define(
    "select"_hs,
    engine::input::controls::trigger{
      .bindings = engine::input::mouse_button(GLFW_MOUSE_BUTTON_LEFT)
    }
  );

  engine::input::controls::main().define(
    "target"_hs,
    engine::input::controls::trigger{
      .bindings = engine::input::mouse_button(GLFW_MOUSE_BUTTON_RIGHT)
    }
  );
}
```

And use them:

```cpp
void update(entt::registry& registry) {
  auto click_action = engine::input::controls::main().read_trigger("select"_hs);
  if (click_action.performed) {
    // a click happened
    // and we're sure it was not on a UI element
  }
}
```

In the next part of this series, we'll setup the controls for the UI like this:

```cpp
namespace engine::ui::internal {
  using namespace entt::literals;

  constexpr auto KEY_CLICK_ACTION = "__internal__.ui.action.click"_hs;
  constexpr auto KEY_COPY_ACTION  = "__internal__.ui.action.copy"_hs;
  constexpr auto KEY_CUT_ACTION   = "__internal__.ui.action.cut"_hs;
  constexpr auto KEY_PASTE_ACTION = "__internal__.ui.action.paste"_hs;

  constexpr auto KEY_MOUSEMOVE_ACTION   = "__internal__.ui.action.mousemove"_hs;
  constexpr auto KEY_MOUSESCROLL_ACTION = "__internal__.ui.action.mousescroll"_hs;

  constexpr auto KEY_DELTATIME_ACTION = "__internal__.ui.action.deltatime"_hs;

  void setup_input_controls() {
    input::controls::main().define(
      KEY_CLICK_ACTION,
      input::controls::trigger{
        .ignore_capture = true,
        .bindings       = input::mouse_button(GLFW_MOUSE_BUTTON_LEFT),
      }
    );

    input::controls::main().define(
      KEY_COPY_ACTION,
      input::controls::trigger{
        .ignore_capture = true,
        .bindings       = (input::key(GLFW_KEY_LEFT_CONTROL) | input::key(GLFW_KEY_RIGHT_CONTROL)) & input::key(GLFW_KEY_C),
      }
    );

    input::controls::main().define(
      KEY_CUT_ACTION,
      input::controls::trigger{
        .ignore_capture = true,
        .bindings       = (input::key(GLFW_KEY_LEFT_CONTROL) | input::key(GLFW_KEY_RIGHT_CONTROL)) & input::key(GLFW_KEY_X),
      }
    );

    input::controls::main().define(
      KEY_PASTE_ACTION,
      input::controls::trigger{
        .ignore_capture = true,
        .bindings       = (input::key(GLFW_KEY_LEFT_CONTROL) | input::key(GLFW_KEY_RIGHT_CONTROL)) & input::key(GLFW_KEY_V),
      }
    );

    input::controls::main().define(
      KEY_MOUSEMOVE_ACTION,
      input::controls::passthrough_point{
        .ignore_capture = true,
        .source_type    = input::passthrough_point_source::mouse_pointer{},
      }
    );

    input::controls::main().define(
      KEY_MOUSESCROLL_ACTION,
      input::controls::passthrough_point{
        .ignore_capture = true,
        .source_type    = input::passthrough_point_source::mouse_wheel{},
      }
    );

    input::controls::main().define(
      KEY_DELTATIME_ACTION,
      input::controls::passthrough_time{
        .source_type = input::passthrough_time_source::delta_time{},
      }
    );
  }
}
```

Neat, isn't it? :)

---

## Conclusion

This concludes the first part of this series, which lays the foundation for part 3. I hope you enjoyed it, feel free to give your remarks, tips, or critics in the comments :)

From there, you could choose to deserialize your controls from a JSON asset, allowing the player to remap the bindings if they wishes. I did not implement this yet, and this may be for way later in my game. Though, it should not be complicated. I'll add this snippet if you want to create `entt::id_type` from runtime strings instead of compile-time hashed strings:

```cpp
std::string my_rt_id = "my runtime id";
entt::id_type id = entt::hashed_string{my_rt_id.c_str(), my_rt_id.size()};
```

In the [next part](/posts/2025-07-09-roll-your-own-in-game-ui-with-clay-and-nanovg-part-2), we'll implement a NanoVG renderer for Clay, so stay tuned!
