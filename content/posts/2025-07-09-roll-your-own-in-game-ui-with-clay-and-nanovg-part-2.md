---
title: "Roll your own in-game UI with Clay and NanoVG (part 2)"
date: "2025-07-09T15:57:22Z"
description: "Introduction   In the previous part of this series, we laid out some foundations that will..."
tags: ["gamedev", "cpp", "programming", "tutorial"]
author: "David Delassus"
images: ["img/image-0dbe064a6520fcab.png"]
---

## Introduction

In the [previous part](/posts/2025-07-08-roll-your-own-in-game-ui-with-clay-and-nanovg-part-1) of this series, we laid out some foundations that will shine in the next part when we are gonna implement a text input component.

But first, we need to get to the core of this series, which is integrating the actual libraries that will allow us to design our in-game UI:

 - **Clay:** https://github.com/nicbarker/clay
 - **NanoVG:** https://github.com/memononen/nanovg

> **DISCLAIMER:** It seems NanoVG is not actively maintained. It still works really well, but if you somehow manage to stumble on bugs in the library, you might need to fix them yourself. Take that into considerations before using it.

---

## Table of Contents

 - [Part 1: GLFW, EnTT, and an input system](/posts/2025-07-08-roll-your-own-in-game-ui-with-clay-and-nanovg-part-1)
 - **Part 2: Writing a NanoVG renderer for Clay**
 - Part 3: A text input component with stb_textedit

---

<!--
MARK: 1. init
-->

### 1. Initialization and teardown

**Clay** is a header-only library, which means you need to have a compilation unit in your project that includes its implementation.

**NanoVG** is not a header-only, but its backend (OpenGL in our case) is.

So let's first create a C file in our project with the following content:

```c
#include <glad/glad.h>
#include <GLFW/glfw3.h>

#define NANOVG_GL3_IMPLEMENTATION
#include <nanovg.h>
#include <nanovg_gl.h>

#define CLAY_IMPLEMENTATION
#include <clay.h>
```

> **NB:** It's important for this file to be a C file and not a C++ file. Clay uses designated field initializers in a lot of its code (which is C, not C++). In C++23 (since C++20 I think, not sure, don't care), not listing all of the initializers in the correct order triggers a warning, and I usually compile with `-Wall -Werror` at least.

We will also use this file to write some extension functions. A few features of Clay are not publicly exposed.

Clay does not do any dynamic allocations. We provide it with some memory at the start, and that's all it will use. It does provide us with the function `Clay_MinMemorySize()` that returns the minimal required size.

That "minimum size" is actually calculated from a maximum number of elements, which defaults to `8192`, plenty enough for even complex applications. Should you want to increase that, you can use the `Clay_SetMaxElementCount()` function before initializing Clay.

As for NanoVG, the creation of the "context" is done by its backend (in our case, the OpenGL backend), and it's a dynamic allocation. You have to call the destructor yourself. Which is why, as good C++ manners, we'll wrap it up in a `std::unique_ptr` with a deleter.

For simplicity, we'll create a structure to hold the 2 together:

```cpp
#define NANOVG_GL3
#include <nanovg.h>
#include <nanovg_gl.h>

#include <clay.h>


namespace engine::ui {
  struct nvg_context_deleter {
    void operator()(NVGcontext* self) {
      if (self != nullptr) {
        nvgDeleteGL3(self);
      }
    }
  };

  using nvg_context = std::unique_ptr<NVGcontext, nvg_context_deleter>;

  struct context {
    std::vector<std::byte> layout_memory;
    Clay_Arena layout_arena;

    nvg_context renderer_context;

    context(int width, int height) {
      layout_memory.resize(static_cast<size_t>(Clay_MinMemorySize()));
      layout_arena = Clay_CreateArenaWithCapacityAndMemory(
        layout_memory.size(),
        layout_memory.data()
      );

      Clay_Initialize(
        layout_arena,
        Clay_Dimensions{
          .width  = static_cast<float>(width),
          .height = static_cast<float>(height)
        },
        Clay_ErrorHandler{
          .errorHandlerFunction = [](Clay_ErrorData err) {
            auto msg = std::string_view{
              err.errorText.chars,
              static_cast<size_t>(err.errorText.length)
            };
            std::print("[UI] ERROR: {}", msg));
          },
          .userData = nullptr,
        }
      );

      rendering_context = nvg_context{
        nvgCreateGL3(NVG_ANTIALIAS | NVG_STENCIL_STROKES)
      };
    }
  };
}
```

We can then update our game loop to initialize and teardown our UI context:

```diff
+  auto& ui_ctx = registry.ctx().emplace<engine::ui::context>(1366, 768);
   init(registry);
```

```diff
   teardown(registry);
+  registry.ctx().erase<engine::ui::context>();
```

<!--
MARK: 2. input
-->

### 2. Input management

Every frame, we need to give to Clay some inputs:

- the size of the framebuffer, so that responsive layout can be computed properly)
- the position of the mouse and the state of the left mouse button, so that you can build interactive components
- the state of the mouse wheel, so that you can scroll

To fetch those values, we will setup a few input controls using the input system we implemented in part 1. A new method on our `context` structure can do the trick:

```cpp
namespace engine::ui {
  using namespace entt::literals;

  constexpr auto KEY_CLICK_ACTION       = "ui.action.click"_hs;
  constexpr auto KEY_MOUSEMOVE_ACTION   = "ui.action.mousemove"_hs;
  constexpr auto KEY_MOUSESCROLL_ACTION = "ui.action.mousescroll"_hs;
  constexpr auto KEY_DELTATIME_ACTION   = "ui.action.deltatime"_hs;

  void context::setup_input_controls() {
    input::controls::main().define(
      KEY_CLICK_ACTION,
      input::controls::trigger{
        .ignore_capture = true,
        .bindings       = input::mouse_button(GLFW_MOUSE_BUTTON_LEFT),
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

And we call it during initialization:

```diff
   auto& ui_ctx = registry.ctx().emplace<engine::ui::context>(1366, 768);
+  ui_ctx.setup_input_controls();
   init(registry);
```

Now that our input controls are defined, we can implement a `begin_render()` method:

```cpp
namespace engine::ui {
  void context::begin_render(entt::registry& registry /* will be used in part 3 */) {
    // first, we read the values of our inputs
    auto click_action       = input::controls::main().read_trigger(KEY_CLICK_ACTION);
    auto mousemove_action   = input::controls::main().read_passthrough_point(KEY_MOUSEMOVE_ACTION);
    auto mousescroll_action = input::controls::main().read_passthrough_point(KEY_MOUSESCROLL_ACTION);
    auto deltatime_action   = input::controls::main().read_passthrough_time(KEY_DELTATIME_ACTION);

    // we query the current framebuffer size,
    // this can be the window size, or the size of your FBO
    int w, h;
    glfwGetFramebufferSize(glfwGetCurrentContext(), &w, &h);

    // pass the inputs to Clay
    Clay_SetLayoutDimensions(Clay_Dimensions{
      .width  = static_cast<float>(w),
      .height = static_cast<float>(h)
    });

    Clay_SetPointerState(
      Clay_Vector2{
        .x = mousemove_action.value.x,
        .y = mousemove_action.value.y,
      },
      click_action.active
    );

    Clay_UpdateScrollContainers(
      true,
      Clay_Vector2{
        .x = mousescroll_action.value.x,
        .y = mousescroll_action.value.y,
      },
      static_cast<float>(deltatime_action.value)
    );

    // tell Clay we are ready to write some UI code
    Clay_BeginLayout();
  }

  void context::end_render() {
    // compile the layout into an array of render commands,
    // those render commands will be handled by the renderer using NanoVG
    auto cmds = Clay_EndLayout();

    // TODO: renderer
  }
}
```

Those new methods can then be called in the game-loop:

```diff
     begin_frame(registry);
     engine::input::controls::main().update(registry);
     update(registry, delta_time);
     render(registry);

+    ui_ctx.begin_render(registry);
+    // your Clay code
+    ui_ctx.end_render();

     input_state.key_buffer.clear();
     input_state.char_buffer.clear();

     end_frame(registry);
```

<!--
MARK: 3. renderer
-->
### 3. Implementing the renderer

All the Clay functions (which are called using the macros `CLAY`, `CLAY_TEXT`, etc...) create elements in the library's arena (using the memory we gave Clay during initialization).

In the `end_render()` method, we call the `Clay_EndLayout()` function, which compiles all those elements into an array of render commands.

Clay provides the following render commands:

| Command Type | Role |
| --- | --- |
| `CLAY_RENDER_COMMAND_TYPE_RECTANGLE` | Draw a filled rectangle |
| `CLAY_RENDER_COMMAND_TYPE_BORDER` | Draw an outlined rectangle |
| `CLAY_RENDER_COMMAND_TYPE_TEXT` | Draw text |
| `CLAY_RENDER_COMMAND_TYPE_IMAGE` | Draw a texture |
| `CLAY_RENDER_COMMAND_TYPE_SCISSOR_START` | Start clipping content |
| `CLAY_RENDER_COMMAND_TYPE_SCISSOR_END` | Stop clipping content |
| `CLAY_RENDER_COMMAND_TYPE_CUSTOM` | Render our own custom elements (we will use this for the text input in part 3) |

For each of those commands, we will implement a static function, this will make the render loop and switch more concise.

<!--
MARK: 3.1 rect
-->
#### 3.1. Draw a filled rectangle

Clay will give us the "bounding box" in which to draw the rectangle, as well as its background color and eventual corner radius.

```cpp
static void render_command_rectangle(NVGcontext* ctx, const Clay_RenderCommand& cmd) {
  nvgSave(ctx); // save the current state of the NanoVG context
  nvgBeginPath(ctx);
  nvgRoundedRectVarying(
    ctx,
    cmd.boundingBox.x,
    cmd.boundingBox.y,
    cmd.boundingBox.width,
    cmd.boundingBox.height,
    cmd.renderData.rectangle.cornerRadius.topLeft,
    cmd.renderData.rectangle.cornerRadius.topRight,
    cmd.renderData.rectangle.cornerRadius.bottomRight,
    cmd.renderData.rectangle.cornerRadius.bottomLeft
  );
  // Clay gives us the color as floats, but still in the 0-255 range
  // NanoVG expects bytes in the 0-255 range
  // a simple static cast is enough
  nvgFillColor(
    ctx,
    nvgRGBA(
      static_cast<uint8_t>(cmd.renderData.rectangle.backgroundColor.r),
      static_cast<uint8_t>(cmd.renderData.rectangle.backgroundColor.g),
      static_cast<uint8_t>(cmd.renderData.rectangle.backgroundColor.b),
      static_cast<uint8_t>(cmd.renderData.rectangle.backgroundColor.a)
    )
  );
  nvgFill(ctx);
  nvgRestore(ctx); // restore the previous state of the NanoVG context
}
```

As you can see, NanoVG provides an API very similar to HTML5 canvas. This will make the actual rendering quite easy.

<!--
MARK: 3.2. border
-->
#### 3.2. Draw an outlined rectangle

The code will be almost identical, except that we will use `stroke` instead of `fill`:

```cpp
static void render_command_border(NVGcontext* ctx, const Clay_RenderCommand& cmd) {
  nvgSave(ctx);
  nvgBeginPath(ctx);
  nvgRoundedRectVarying(
    ctx,
    cmd.boundingBox.x,
    cmd.boundingBox.y,
    cmd.boundingBox.width,
    cmd.boundingBox.height,
    cmd.renderData.border.cornerRadius.topLeft,
    cmd.renderData.border.cornerRadius.topRight,
    cmd.renderData.border.cornerRadius.bottomRight,
    cmd.renderData.border.cornerRadius.bottomLeft
  );
  nvgStrokeColor(
    ctx,
    nvgRGBA(
      static_cast<uint8_t>(cmd.renderData.border.color.r),
      static_cast<uint8_t>(cmd.renderData.border.color.g),
      static_cast<uint8_t>(cmd.renderData.border.color.b),
      static_cast<uint8_t>(cmd.renderData.border.color.a)
    )
  );
  nvgStrokeWidth(ctx, cmd.renderData.border.width.left);
  nvgStroke(ctx);
  nvgRestore(ctx);
}
```

> **NB:** While Clay supports distinct border widths for the same element, I don't think NanoVG does out of the box. So for simplicity, we use the `left` border width for everything and ignore the others.

<!--
MARK: 3.3. text
-->
#### 3.3. Draw text

Clay provides a `fontId` field, which contains whatever identifier we want that can be used to use a specific font for the text rendering.

On the other side, NanoVG provides a function `nvgFontFaceId()` which allows us to define which font to use for the text rendering, by its identifier.

But NanoVG needs the font in its font atlas first. Under the hood, it will use `stb_truetype` to load said font. Let's add a method to our `engine::ui::context` class:

```cpp
namespace engine::ui {
  using font_id_type = uint16_t;

  font_id_type context::add_font_to_atlas(const std::filesystem::path& path) {
    // TODO: load font from filesystem using your preferred method
    // We'll assume you initialized the following variables with your
    // font data:
    void* asset_content_data;
    size_t asset_content_size;

    auto font_id = nvgCreateFontMem(
      rendering_context.get(),
      path.filename().string().c_str(),
      reinterpret_cast<uint8_t*>(asset_content_data),
      static_cast<int>(asset_content_size),
      false
    );

    return static_cast<font_id_type>(font_id);
  }
}
```

You can then load your favorite font in the `init()` function:

```diff
 void init(entt::registry& registry) {
+  auto& ui_ctx = registry.ctx().get<engine::ui::context>();
+  ui_ctx.add_font_to_atlas("/path/to/your/font.ttf");
 }
```

We load only one font, its identifier will be `0`, which is also the default value of the `fontId` field in Clay. I'll live things up to you on how to keep track of your font identifiers.

Now that we have a font loaded, we can implement our rendering function:

```cpp
static void render_command_text(NVGcontext* ctx, const Clay_RenderCommand& cmd) {
  nvgSave(ctx);
  nvgFontFaceId       (ctx, static_cast<int>  (cmd.renderData.text.fontId));
  nvgFontSize         (ctx, static_cast<float>(cmd.renderData.text.fontSize));
  nvgTextLetterSpacing(ctx, static_cast<float>(cmd.renderData.text.letterSpacing));
  nvgTextLineHeight   (ctx, static_cast<float>(cmd.renderData.text.lineHeight));
  nvgTextAlign        (ctx, NVG_ALIGN_LEFT | NVG_ALIGN_TOP);
  nvgFillColor(
    ctx,
    nvgRGBA(
      static_cast<uint8_t>(cmd.renderData.text.textColor.r),
      static_cast<uint8_t>(cmd.renderData.text.textColor.g),
      static_cast<uint8_t>(cmd.renderData.text.textColor.b),
      static_cast<uint8_t>(cmd.renderData.text.textColor.a)
    )
  );
  nvgText(
    ctx,
    cmd.boundingBox.x,
    cmd.boundingBox.y,
    cmd.renderData.text.stringContents.chars,
    cmd.renderData.text.stringContents.chars + cmd.renderData.text.stringContents.length
  );
  nvgRestore(ctx);
}
```

Clay needs to be able to measure text in order to properly calculate the bounding boxes. It requires us to provide it a measuring function, which we can implement with NanoVG:

```cpp
Clay_Dimensions context::measure_text(
  Clay_StringSlice text,
  Clay_TextElementConfig* config
) {
  auto* ctx = rendering_context.get();

  nvgSave(ctx);

  nvgFontFaceId       (ctx, static_cast<int>  (config->fontId));
  nvgFontSize         (ctx, static_cast<float>(config->fontSize));
  nvgTextLetterSpacing(ctx, static_cast<float>(config->letterSpacing));
  nvgTextLineHeight   (ctx, static_cast<float>(config->lineHeight));

  float bounds[4];
  auto advance = nvgTextBounds(ctx, 0, 0, text.chars, text.chars + text.length, bounds);

  nvgRestore(ctx);

  return Clay_Dimensions{
    .width  = advance,
    .height = bounds[3] - bounds[1]
  };
}
```

> **NB:** We don't use the `bounds` for the width, because those could be `0` or even a negative number (for kerning support in non-monospace fonts). For example a string containing only spaces would have a width of `0`. However, in part 3 when we will implement the text input component, we do need a proper width for every character, including spaces. Thus, we use the `advance` instead, which represent the amount of pixels we need to move forward for the next character.

Now, we can give the text measuring function to Clay:

```diff
+      Clay_SetMeasureTextFunction(
+        [](auto text, auto config, auto user_data) {
+          auto* self = static_cast<context*>(user_data);
+          self->measure_text(text, config);
+        },
+        this
+      );

       rendering_context = nvg_context{
         nvgCreateGL3(NVG_ANTIALIAS | NVG_STENCIL_STROKES)
       };
```

<!--
MARK: 3.4. texture
-->
#### 3.4. Draw a texture

Similarly, Clay provides a field `imageData` which is a `void*` pointer. I choose to put a texture identifier in that pointer, using `reinterpret_cast`.

NanoVG requires the texture to be loaded into an atlas as well, which means we need yet another method on our `context` class:

```cpp
namespace engine::ui {
  using texture_id_type = int;

  texture_id_type context::add_texture_to_atlas(const std::filesystem::path& path) {
    // TODO: load image from filesystem using your preferred method
    // We'll assume you initialized the following variables with your
    // image data:
    void* asset_content_data;
    size_t asset_content_size;

    auto tex_id = nvgCreateImageMem(
      rendering_context.get(),
      0,
      reinterpret_cast<uint8_t*>(asset_content_data),
      static_cast<int>(asset_content_size)
    );

    return static_cast<texture_id_type>(tex_id);
  }
}
```

Just like NanoVG used `stb_truetype` to parse the font, it will use `stb_image` to parse the texture.

We can now implement the rendering function:

```cpp
static void render_command_image(NVGcontext* ctx, const Clay_RenderCommand& cmd) {
  nvgSave(ctx);
  nvgBeginPath(ctx);
  nvgRoundedRectVarying(
    ctx,
    cmd.boundingBox.x,
    cmd.boundingBox.y,
    cmd.boundingBox.width,
    cmd.boundingBox.height,
    cmd.renderData.image.cornerRadius.topLeft,
    cmd.renderData.image.cornerRadius.topRight,
    cmd.renderData.image.cornerRadius.bottomRight,
    cmd.renderData.image.cornerRadius.bottomLeft
  );
  nvgFillPaint(
    ctx,
    nvgImagePattern(
      ctx,
      cmd.boundingBox.x,
      cmd.boundingBox.y,
      cmd.boundingBox.width,
      cmd.boundingBox.height,
      0.0f,
      static_cast<int>(reinterpret_cast<intptr_t>(cmd.renderData.image.imageData)),
      static_cast<float>(cmd.renderData.image.backgroundColor.a) / 255.0f // alpha is in [0;1] range
    )
  );
  nvgFill(ctx);
  nvgRestore(ctx);
}
```

<!--
MARK: 3.5. clipping
-->
#### 3.5. Clipping

This part is so straightforward, I will just paste the code:

```cpp
static void render_command_scissor_start(NVGcontext* ctx, const Clay_RenderCommand& cmd) {
  nvgScissor(
    ctx,
    cmd.boundingBox.x,
    cmd.boundingBox.y,
    cmd.boundingBox.width,
    cmd.boundingBox.height
  );
}

static void render_command_scissor_end(NVGcontext* ctx) {
  nvgResetScissor(ctx);
}
```

Notice how in this case I do not use `nvgSave(ctx)` and `nvgRestore(ctx)`? This would reset the scissor if I did.

<!--
MARK: 3.6. render loop
-->
#### 3.6. The actual render loop

We are now almost done. We need to loop over the render commands array, and call the proper rendering functions:

```cpp
static void render_command_array(NVGcontext* ctx, const Clay_RenderCommandArray& cmds) {
  int ww, wh;
  glfwGetWindowSize(glfwGetCurrentContext(), &ww, &wh);

  int fbw, fbh;
  glfwGetFramebufferSize(glfwGetCurrentContext(), &fbw, &fbh);

  auto device_pixel_ratio = static_cast<float>(fbw) / static_cast<float>(ww);

  nvgBeginFrame(
    ctx,
    static_cast<float>(ww),
    static_cast<float>(wh),
    device_pixel_ratio
  );

  for (int32_t i = 0; i < cmds.length; ++i) {
    const auto& cmd = cmds.internalArray[i];

    switch (cmd.commandType) {
      case CLAY_RENDER_COMMAND_TYPE_RECTANGLE:
        render_command_rectangle(ctx, cmd);
        break;

      case CLAY_RENDER_COMMAND_TYPE_BORDER:
        render_command_border(ctx, cmd);
        break;

      case CLAY_RENDER_COMMAND_TYPE_TEXT:
        render_command_text(ctx, cmd);
        break;

      case CLAY_RENDER_COMMAND_TYPE_IMAGE:
        render_command_image(ctx, cmd);
        break;

      case CLAY_RENDER_COMMAND_TYPE_SCISSOR_START:
        render_command_scissor_start(ctx, cmd);
        break;

      case CLAY_RENDER_COMMAND_TYPE_SCISSOR_END:
        render_command_scissor_end(ctx);
        break;

      case CLAY_RENDER_COMMAND_TYPE_CUSTOM:
        // TODO: part 3
        break;

      default:
        break;
    }
  }

  nvgEndFrame(ctx);
}
```

If you only use Clay and NanoVG, this is all you need. But if you are rendering your game scene, you will notice that NanoVG changes some settings on the OpenGL context.

We need to save those settings before using NanoVG and restore them afterwards. To do this, I wrote a simple helper structure:

```cpp
namespace engine::graphics::internal {
  struct state {
    GLint viewport[4];
    GLint scissor[4];

    GLint array_buffer;
    GLint element_array_buffer;

    GLint current_program;
    GLint bound_vao;
    GLint bound_tex;

    GLint blend_src_rgb;
    GLint blend_src_a;

    GLint blend_dst_rgb;
    GLint blend_dst_a;

    GLint blend_eq_rgb;
    GLint blend_eq_a;

    GLboolean blend_enabled;
    GLboolean cull_face_enabled;
    GLboolean depth_test_enabled;
    GLboolean scissor_test_enabled;
    GLboolean depth_mask_enabled;

    void fetch() {
      glGetIntegerv(GL_VIEWPORT,    viewport);
      glGetIntegerv(GL_SCISSOR_BOX, scissor);

      glGetIntegerv(GL_ARRAY_BUFFER_BINDING,         &array_buffer);
      glGetIntegerv(GL_ELEMENT_ARRAY_BUFFER_BINDING, &element_array_buffer);

      glGetIntegerv(GL_CURRENT_PROGRAM,      &current_program);
      glGetIntegerv(GL_VERTEX_ARRAY_BINDING, &bound_vao);
      glGetIntegerv(GL_TEXTURE_BINDING_2D,   &bound_tex);

      glGetIntegerv(GL_BLEND_SRC_RGB,   &blend_src_rgb);
      glGetIntegerv(GL_BLEND_SRC_ALPHA, &blend_src_a);

      glGetIntegerv(GL_BLEND_DST_RGB,   &blend_dst_rgb);
      glGetIntegerv(GL_BLEND_DST_ALPHA, &blend_dst_a);

      glGetIntegerv(GL_BLEND_EQUATION_RGB,   &blend_eq_rgb);
      glGetIntegerv(GL_BLEND_EQUATION_ALPHA, &blend_eq_a);

      glGetBooleanv(GL_BLEND,           &blend_enabled);
      glGetBooleanv(GL_CULL_FACE,       &cull_face_enabled);
      glGetBooleanv(GL_DEPTH_TEST,      &depth_test_enabled);
      glGetBooleanv(GL_SCISSOR_TEST,    &scissor_test_enabled);
      glGetBooleanv(GL_DEPTH_WRITEMASK, &depth_mask_enabled);
    }

    void restore() const {
      glViewport(viewport[0], viewport[1], viewport[2], viewport[3]);
      glScissor(scissor[0], scissor[1], scissor[2], scissor[3]);

      glBindBuffer(GL_ARRAY_BUFFER, array_buffer);
      glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, element_array_buffer);

      glUseProgram(current_program);
      glBindVertexArray(bound_vao);
      glBindTexture(GL_TEXTURE_2D, bound_tex);

      glBlendFuncSeparate(blend_src_rgb, blend_dst_rgb, blend_src_a, blend_dst_a);
      glBlendEquationSeparate(blend_eq_rgb, blend_eq_a);

      if (blend_enabled) {
        glEnable(GL_BLEND);
      }
      else {
        glDisable(GL_BLEND);
      }

      if (cull_face_enabled) {
        glEnable(GL_CULL_FACE);
      }
      else {
        glDisable(GL_CULL_FACE);
      }

      if (depth_test_enabled) {
        glEnable(GL_DEPTH_TEST);
      }
      else {
        glDisable(GL_DEPTH_TEST);
      }

      if (scissor_test_enabled) {
        glEnable(GL_SCISSOR_TEST);
      }
      else {
        glDisable(GL_SCISSOR_TEST);
      }

      glDepthMask(depth_mask_enabled);
    }
  };
}
```

We can amend the rendering loop:

```diff
 static void render_command_array(NVGcontext* ctx, const Clay_RenderCommandArray& cmds) {
+  auto gl_state = engine::graphics::internal::state{};
+  gl_state.fetch();
```

```diff
   nvgEndFrame(ctx);
+  gl_state.restore();
 }
```

#### 3.7. Calling the render loop

We can now call our renderer in the `end_render()` method:

```diff
 void context::end_render() {
   auto cmds = Clay_EndLayout();
+  render_command_array(rendering_context.get(), cmds);
 }
```

<!--
MARK: 4. input
-->
### 4. Resolving conflicting inputs

Let's say we have the following UI:

```cpp
CLAY({ .id = CLAY_ID("outer container"), /* ... */ }) {
  CLAY({ .id = CLAY_ID("sidebar"), /* ... */ }) {
    auto& input_state = registry.ctx().get<engine::input::state>();
    input_state.capture_mouse |= Clay_Hovered();

    // ...
  }
}
```

Since we render the UI **after** our call to:

```cpp
engine::input::controls::main().update(registry);
```

The `capture_mouse` flag won't be set at the right moment. And inputs will go through the UI into the game scene.

We need to process the inputs before. But in order to process the inputs, we have to compute the UI a first time, so that we know where the UI elements are.

To do this, we can add 2 methods to our `context` class:

```cpp
void context::begin_prerender(entt::registry& registry) {
  auto& input_state = registry.ctx().get<input::state>();

  input_state.capture_keyboard = false;
  input_state.capture_mouse    = false;

  Clay_BeginLayout();
}

void context::end_prerender() {
  Clay_EndLayout();
}
```

And we update our game loop like this:

```diff
     begin_frame(registry);
+
+    ui_ctx.begin_prerender(registry);
+    show_ui(registry, true);
+    ui_ctx.end_prerender();
+
     engine::input::controls::main().update(registry);
     update(registry, delta_time);
     render(registry);

     ui_ctx.begin_render(registry);
     show_ui(registry, false);
     ui_ctx.end_render();

     input_state.key_buffer.clear();
     input_state.char_buffer.clear();

     end_frame(registry);
```

And we move our UI code into the new `show_ui` function:

```cpp
void show_ui(entt::registry& registry, bool prerender) {
  CLAY({ .id = CLAY_ID("outer container"), /* ... */ }) {
    CLAY({ .id = CLAY_ID("sidebar"), /* ... */ }) {
      if (prerender) {
        auto& input_state = registry.ctx().get<engine::input::state>();
        input_state.capture_mouse |= Clay_Hovered();
      }

      // ...
    }
  }
}
```

Side effects (like actions triggered when clicking on buttons) should not be triggered in the "prerender" phase.

<!--
MARK: 5. layout builder
-->
### 5. Layout Builder API and buttons

While we are technically done with the NanoVG renderer for Clay, there is still a lot of manual work to implement typical UI elements (like buttons).

We can abstract away the Clay API behind a "Layout Builder".

For example, we might want to apply different styles on buttons in there 3 states: `normal`, `hovered`, and `active`.

To do this, we can define a class that will expose a simpler API:

```cpp
namespace engine::ui {

namespace engine::ui {
  class layout_builder {
    public:
      // we first define some structures to hold our "styles"
      struct element_config {
        Clay_ElementId id;

        Clay_ElementDeclaration style;
      };

      struct text_config {
        Clay_String text;

        Clay_TextElementConfig style;
      };

      // a button has a style for the container in each state
      // and for the label in each state
      struct button_config {
        Clay_ElementId id;

        Clay_String label;

        struct {
          struct {
            Clay_ElementDeclaration normal;
            Clay_ElementDeclaration hovered;
            Clay_ElementDeclaration active;
          } container;

          struct {
            Clay_TextElementConfig normal;
            Clay_TextElementConfig hovered;
            Clay_TextElementConfig active;
          } label;
        } style;
      };

    public:
      // pass the registry and prerender flag so that it can abstract away the
      // input logic
      layout_builder(entt::registry& registry, bool prerender);

      // we will use this on an element to capture the mouse
      void capture_mouse();

      // this will remplace the CLAY macro
      void begin_element(element_config cfg);
      void end_element();

      // this will remplace the CLAY_TEXT macro
      void text(text_config cfg);

      // this will be our new highlevel component
      bool button(button_config cfg);

    private:
      entt::registry& m_registry;
      bool            m_prerender;
  };
}
```

The constructor is quite straightforward to implement:

```cpp
namespace engine::ui {
  layout_builder::layout_builder(entt::registry& registry, bool prerender)
    : m_registry{registry}, m_prerender{prerender}
  {}
}
```

The `capture_mouse` method will use the `m_prerender` field, so that we can hide the `if` statement from our `show_gui` method:

```cpp
namespace engine::ui {
  void layout_builder::capture_mouse() {
    if (m_prerender) {
      auto& input_state = m_registry.ctx().get<input::state>();
      input_state.capture_mouse |= Clay_Hovered();
    }
  }
}
```

<!--
MARK: 5.1. Elements
-->
#### 5.1. API to create generic elements

The `CLAY(...) { ... }` macro actually expands to something like this:

```cpp
Clay__OpenElement();
Clay__ConfigureOpenElement(/* ... */);
/* ... */
Clay__CloseElement();
```

We can abstract this away with our `begin_element()` and `end_element()` methods:

```cpp
namespace engine::ui {
  void layout_builder::begin_element(element_config cfg) {
    Clay__OpenElement();

    auto decl = cfg.style;
    decl.id   = cfg.id;

    Clay__ConfigureOpenElement(decl);
  }


  void layout_builder::end_element() {
    Clay__CloseElement();
  }
}
```

<!--
MARK: 5.2. text
-->
#### 5.2. API to create text elements

Similarly, the `CLAY_TEXT(label, config)` macro expands to this:

```cpp
Clay__OpenTextElement(label, Clay__StoreTextElementConfig(config));
```

Which translates directly to our `text()` method:

```cpp
namespace engine::ui {
  void layout_builder::text(text_config cfg) {
    Clay__OpenTextElement(cfg.text, Clay__StoreTextElementConfig(cfg.style));
  }
}
```

<!--
MARK: 5.3. button
-->
#### 5.3. API to create buttons

The `Clay_Hovered()` function tells us if the currently open element has the mouse pointer hover it. But we have no such function to tell us if the left mouse button was clicked or not.

Normally, you would have to give a function pointer to `Clay_OnHover()`, which
will be called the next time we call `Clay_SetPointerState()`, therefore on the
next frame.

I was not satisfied with it. But fortunately, in the compilation unit where we defined `CLAY_IMPLEMENTATION`, we have access to many internal functions, allowing us to write "extensions".

Let's write a `ClayX_Clicked()` and a `ClayX_Pressed()` function. Notice the `ClayX` prefix to identify quickly that this is not part of the official Clay API.

> **NB:** Those **must** be in the C file that defines `CLAY_IMPLEMENTATION`.

```c
bool ClayX_Pressed(void) {
  if (Clay_Hovered()) {
    Clay_Context* ctx = Clay_GetCurrentContext();

    if (
      ctx->pointerInfo.state == CLAY_POINTER_DATA_PRESSED_THIS_FRAME ||
      ctx->pointerInfo.state == CLAY_POINTER_DATA_PRESSED
    ) {
      return true;
    }
  }

  return false;
}


bool ClayX_Clicked(void) {
  if (Clay_Hovered()) {
    Clay_Context* ctx = Clay_GetCurrentContext();

    if (ctx->pointerInfo.state == CLAY_POINTER_DATA_RELEASED_THIS_FRAME) {
      return true;
    }
  }

  return false;
}
```

Then, in a header file, we can declare those functions:

```cpp
extern "C" {
  bool ClayX_Pressed(void);
  bool ClayX_Clicked(void);
}
```

Now, we can move on to implementing our `button()` method on the layout builder. The logic is quite simple, depending on the pointer state, we will choose what style to use, and then we return wether the button was
clicked or not, to allow the user to trigger side effects:

```cpp
namespace engine::ui {
  bool layout_builder::button(button_config cfg) {
    Clay__OpenElement();

    // remember, we don't want to trigger side effects in the prerender phase
    auto clicked = !m_prerender && ClayX_Clicked();

    auto container_decl = cfg.style.container.normal;
    auto label_decl     = cfg.style.label.normal;

    if (ClayX_Pressed()) {
      container_decl = cfg.style.container.active;
      label_decl     = cfg.style.label.active;
    }
    else if (Clay_Hovered()) {
      container_decl = cfg.style.container.hovered;
      label_decl     = cfg.style.label.hovered;
    }

    container_decl.id = cfg.id;

    Clay__ConfigureOpenElement(container_decl);
    Clay__OpenTextElement(cfg.label, Clay__StoreTextElementConfig(label_decl));
    Clay__CloseElement();

    return clicked;
  }
}
```

And voilà!

### 6. Showcase

Our new Layout Builder API can be used in our `show_gui()` function:

```cpp
void show_gui(entt::registry& registry, bool prerender) {
  auto lb = engine::ui::layout_builder{registry, prerender};

  auto root_style = decltype(engine::ui::layout_builder::element_config::style){};
  root_style.layout.sizing.width  = CLAY_SIZING_GROW(0, 0);
  root_style.layout.sizing.height = CLAY_SIZING_GROW(0, 0);
  root_style.layout.padding       = CLAY_PADDING_ALL(16);
  root_style.layout.childGap      = 16;

  auto sidebar_style = decltype(engine::ui::layout_builder::element_config::style){};
  sidebar_style.layout.sizing.width    = CLAY_SIZING_FIXED(200);
  sidebar_style.layout.sizing.height   = CLAY_SIZING_GROW(0, 0);
  sidebar_style.layout.padding         = CLAY_PADDING_ALL(16);
  sidebar_style.layout.childGap        = 16;
  sidebar_style.layout.layoutDirection = CLAY_TOP_TO_BOTTOM;
  sidebar_style.backgroundColor        = {64, 64, 32, 255};

  auto button_style = decltype(engine::ui::layout_builder::button_config::style){};
  button_style.container.normal.layout.sizing.width     = CLAY_SIZING_GROW(0, 0);
  button_style.container.normal.layout.sizing.height    = CLAY_SIZING_FIXED(32);
  button_style.container.normal.layout.childAlignment.x = CLAY_ALIGN_X_CENTER;
  button_style.container.normal.layout.childAlignment.y = CLAY_ALIGN_Y_CENTER;

  button_style.container.hovered = button_style.container.normal;
  button_style.container.active  = button_style.container.normal;

  button_style.container.normal.backgroundColor  = {32, 0, 0, 255};
  button_style.container.hovered.backgroundColor = {64, 0, 0, 255};
  button_style.container.active.backgroundColor  = {128, 0, 0, 255};

  button_style.label.normal.fontSize  = 16;
  button_style.label.normal.textColor = {255, 255, 255, 255};

  button_style.label.hovered = button_style.label.normal;
  button_style.label.active  = button_style.label.normal;

  lb.begin_element({ .id = CLAY_ID("OuterContainer"), .style = root_style });
    lb.begin_element({ .id = CLAY_ID("SideBar"), .style = sidebar_style});
      lb.capture_mouse();

      if (lb.button({ .id = CLAY_ID("Item 1"), .label = CLAY_STRING("Item 1"), .style = button_style })) {
        // Item 1 clicked!
      }

      if (lb.button({ .id = CLAY_ID("Item 2"), .label = CLAY_STRING("Item 2"), .style = button_style })) {
        // Item 2 clicked!
      }

      if (lb.button({ .id = CLAY_ID("Item 3"), .label = CLAY_STRING("Item 3"), .style = button_style })) {
        // Item 3 clicked!
      }
    lb.end_element();
  lb.end_element();
}
```

Which looks like this (don't mind the horrible placeholder colors):

![Clay and NanoVG ingame UI](img/image-0dbe064a6520fcab.png)

---

## Conclusion

This concludes the second part of this series. I hope you enjoyed it, feel free to give your remarks, tips, or critics in the comments :)

In the next part, we'll get to use `stb_textedit` and write a text input component, so stay tuned!
