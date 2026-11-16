---
title: "I’m embedding my game assets into the executable, here’s how I did it…"
date: "2022-12-25T21:35:20Z"
description: "Warmonger Dynasty — Devlog #0 — The asset pipeline"
tags: ["gamedev", "cpp", "programming", "cmake"]
author: "David Delassus"
images: ["img/image-465cbb51ad1d7d51.png"]
---

I’m a huge fan of the 4X strategy game genre. Civilization, Endless Space, or Total War, those are amongst my favorite games.

I’m also a software developer who dabbled with game development as a hobby with a few engines and also from scratch. Although, I never released anything worthwhile because I lack skills in the artistic fields (graphics, music, story telling, …), yeah solo gamedev is **insanely hard**.

Fortunately for me, there are tons of assets on the Unity Asset Store, or even on <https://opengameart.org> for me to get things started. And now, with DALL-E or Stable Diffusion, I can even generate some placeholder sprites/textures. So why not make a game in that genre that I love?

This is how **Warmonger Dynasty** was born:

![](img/image-465cbb51ad1d7d51.png)

In order to make it even simpler, I decided to make it from scratch with the following stack:

- C\+\+20
- [CMake](https://cmake.org/) build system
- [entt](https://github.com/skypjack/entt/) ECS (Entity Component System) header-only library
- [SDL2](https://www.libsdl.org/) (with the extensions SDL2\_image, SDL2\_mixer, …)
- [dear imgui](https://github.com/ocornut/imgui) with the SDL\_Renderer backend

I have 3 goals with this project:

- finish it
- play it with my friends
- make it a static binary

The last point is quite important to me, I want to be able to distribute the game as a single executable, no DLLs, no separate assets, etc…

Compiling SDL2 and other third-parties dependencies is quite easy, but embedding the assets in the executable is a bit trickier. This is the subject of this article, but do not read it as a “you should do it like this”. In fact, you probably shouldn’t.

I decided to vendor all my dependencies, which makes my root `CMakeLists.txt` looks like:

```
cmake_minimum_required(VERSION 3.22.0)
project(warmonger-dynasty CXX)

find_package(OpenGL REQUIRED)

set(BUILD_SHARED_LIBS OFF)
set(SDL_SHARED OFF)
set(SDL_STATIC ON)

set(SDL2MIXER_OPUS OFF)
set(SDL2MIXER_FLAC OFF)
set(SDL2MIXER_MOD_MODPLUG OFF)
set(SDL2MIXER_MIDI_FLUIDSYNTH OFF)
set(SDL2MIXER_SAMPLES OFF)

set(SDL2NET_SAMPLES OFF)

add_subdirectory(vendor/SDL2)

set(SDL2_LIBRARY SDL2-static)
set(SDL2_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/vendor/SDL2/include")

add_subdirectory(vendor/SDL_image)
add_subdirectory(vendor/SDL_mixer)
add_subdirectory(vendor/SDL_net)
add_subdirectory(vendor/imgui)
add_subdirectory(vendor/fmt)          # std::format will only land in GCC 13 in 2023
add_subdirectory(sources/assets)      # our asset library, see later
add_subdirectory(sources/trollworks)  # my home-made game engine based on entt/SDL2
add_subdirectory(sources/game)        # my game's specific logic
```

I have other third-party header-only libraries in the `vendor/` folder:

- [entt](https://github.com/skypjack/entt/) ECS
- [nlohmann/json](https://github.com/nlohmann/json) JSON parser
- [cereal](https://uscilab.github.io/cereal/) (de)serialization library

---

## 📦 Creating the assets archive

I have multiple kind of assets in the game:

- sprites and textures in the PNG or BMP format
- music and sound effects in the WAV format
- GLSL vertex/fragment shaders source code
- JSON metadata

The first step is to put them all in a TAR archive (it’s quite a simple format to parse, we’ll see how afterwards).

Using CMake, we can write the following `CMakeLists.txt`:

```
cmake_minimum_required(VERSION 3.22.0)
project(asset CXX)

file(GLOB_RECURSE ASSETS
  ${PROJECT_SOURCE_DIR}/data/*
)

add_custom_command(
  OUTPUT ${PROJECT_BINARY_DIR}/assets.tar
  COMMAND ${CMAKE_COMMAND} -E tar cvf ${PROJECT_BINARY_DIR}/assets.tar .
  DEPENDS ${ASSETS}
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/data
)
```

CMake provides a simple `tar` implementation, which makes this step portable on all platforms.

---

## 🧩 Embedding the archive

In order to access the TAR archive, I will need a pointer to the data embedded in the executable.

We can do this by generating a static char array, its content will be the bytes of the archive:

```cpp
#include <cstdint>

static const uint8_t archive_bytes[] = {
  #include <libasset-data.inc>
};
```

Now, we need to generate that `libasset-data.inc` file as a list of bytes (in hexadecimal format). This can be done with the utility `xxd` (hexdump), easily installable on both Windows, Mac OS and Linux. We now add the following step to our `CMakeLists.txt` :

```
find_program(HEXDUMP_COMMAND NAMES xxd)

add_custom_command(
  OUTPUT ${PROJECT_BINARY_DIR}/libasset-data.inc
  COMMAND ${HEXDUMP_COMMAND} -i < ${PROJECT_BINARY_DIR}/assets.tar > ${PROJECT_BINARY_DIR}/libasset-data.inc
  DEPENDS ${PROJECT_BINARY_DIR}/assets.tar
)
```

---

## 📚 Building the asset library

Now that we can generate our static char array with the content of our assets archive, we can implement an API to access its data.

First, we need to complete our `CMakeLists.txt`:

```
file(GLOB_RECURSE LIBASSET_SOURCES
  ${PROJECT_SOURCE_DIR}/src/*.cpp
)

file(GLOB_RECURSE LIBASSET_HEADERS
  ${PROJECT_SOURCE_DIR}/include/*.hpp
)

add_library(${PROJECT_NAME} STATIC
  ${LIBASSET_HEADERS}
  ${LIBASSET_SOURCES}
  ${PROJECT_BINARY_DIR}/libasset-data.inc
)
```

Our API will provide an asset manager, which will be in charge of loading and caching the resources. Therefore, we need to link the library against the appropriate third-party libraries:

```
if(WIN32)
  target_link_libraries(${PROJECT_NAME}
    SDL2-static
    SDL2_image
    SDL2_mixer
    opengl32
    glu32
  )
elseif(APPLE)
  target_link_libraries(${PROJECT_NAME}
    SDL2-static
    SDL2_image
    SDL2_mixer
    opengl
    glu
  )
elseif(UNIX)
  target_link_libraries(${PROJECT_NAME}
    SDL2-static
    SDL2_image
    SDL2_mixer
    opengl
    glu
  )
else()
  message(FATAL_ERROR "Platform not supported")
endif()
```

Not forgetting the include paths:

```
target_include_directories(${PROJECT_NAME}
  PRIVATE "${PROJECT_SOURCE_DIR}/include"
  PRIVATE "${PROJECT_BINARY_DIR}"
  PRIVATE "${CMAKE_SOURCE_DIR}/vendor/SDL2/include"
  PRIVATE "${CMAKE_SOURCE_DIR}/vendor/SDL_image"
  PRIVATE "${CMAKE_SOURCE_DIR}/vendor/SDL_mixer/include"
  PRIVATE "${CMAKE_SOURCE_DIR}/vendor/entt/include"
  PRIVATE "${CMAKE_SOURCE_DIR}/vendor/nlohmann/include"
)
```

And the compile/linking options:

```
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  target_compile_definitions(${PROJECT_NAME} PUBLIC
    DEBUG
    _USE_MATH_DEFINES
  )

  target_compile_options(${PROJECT_NAME} PUBLIC
    -g -Wall -std=c++20 -Wno-misleading-indentation
  )

  target_link_options(${PROJECT_NAME} PUBLIC -static-libstdc++)
else()
  target_compile_definitions(${PROJECT_NAME} PUBLIC
    _USE_MATH_DEFINES
  )

  target_compile_options(${PROJECT_NAME} PUBLIC
    -O2 -Wall -Werror -std=c++20 -Wno-misleading-indentation
  )

  target_link_options(${PROJECT_NAME} PUBLIC -static-libstdc++)
endif()
```

A few notes here:

**1 \- Using** `-static-libstdc++`

It makes sure that, once distributed, my executable will load the version of the standard library that was used for building.

I encountered some weird problem, when running the final executable on another computer (or even in Git Bash), it would exit with code 127, which means that a DLL or the program was not found.

Tinkering with [Dependency Walker](http://dependencywalker.com) I finally found out that a different libstdc\+\+ DLL was being loaded or missing.

I then decided to link it statically as well (maybe there is a better option? feel free to comment).

**2 \- Using** `-Wno-misleading-indentation`

It seems that GCC (at the time of writing, I’m on the 11.x version) does not like really long lines in a file (and my static char array’s definition is really long, 50MB long in fact).

See [this bug report](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=89549) for more information.

**3 \- Building/linking this library could not be slower**

I have 50MB worth of assets at the time of writing this article. It takes a couple minutes to compile the file containing the static char array, and over 30 seconds to link the static library into the final executable.

This alone might be a good example of why this method is not suitable, but oh well, it works as intended 🙂

---

## 📑 Parsing the TAR archive

As I said earlier, TAR is a very simple format. I based my parser on the following [OSDev wiki page](https://wiki.osdev.org/Tar).

The `tar.hpp` header:

```cpp
#pragma once

#include <vector>
#include <string>

#define TAR_HEADER_ALIGNMENT  512

namespace libasset {
  enum class tar_entry_type {
    normal,
    hardlink,
    symlink,
    chardev,
    blockdev,
    dir,
    pipe
  };

  struct tar_header {
    char filename[100];
    char mode[8];
    char uid[8];
    char gid[8];
    char size[12];
    char mtime[12];
    char checksum[8];
    char typeflag[1];
    char linkname[100];

    char ustar_indicator[6];
    char ustar_version[2];

    char user[32];
    char group[32];

    char device_major[8];
    char device_minor[8];

    char filename_prefix[155];

    const void *get_data() const;
    const size_t get_size() const;
    tar_entry_type get_type() const;
  };

  class tar_file {
    private:
      std::vector<const tar_header *> m_headers;

    public:
      tar_file(const void *addr);

      const tar_header *find_header(const std::string &filepath) const;
  };
}
```

The `tar.cpp` implementation:

```cpp
#include <libasset/tar.hpp>

#include <cstring>
#include <cstddef>
#include <cstdint>


namespace libasset {
  static size_t parse_size(const char *octal_size) {
    size_t size = 0;
    int digit_count = 1;

    for (int digit_idx = 11; digit_idx > 0; digit_idx--, digit_count *= 8) {
      size += ((octal_size[digit_idx - 1] - '0') * digit_count);
    }

    return size;
  }

  const void *tar_header::get_data() const {
    return (void *) ((uintptr_t) this + TAR_HEADER_ALIGNMENT);
  }

  const size_t tar_header::get_size() const {
    return parse_size(size);
  }

  tar_entry_type tar_header::get_type() const {
    switch (typeflag[0]) {
      case '1': return tar_entry_type::hardlink;
      case '2': return tar_entry_type::symlink;
      case '3': return tar_entry_type::chardev;
      case '4': return tar_entry_type::blockdev;
      case '5': return tar_entry_type::dir;
      case '6': return tar_entry_type::pipe;
      case '0':
      default:
        return tar_entry_type::normal;
    }
  }

  tar_file::tar_file(const void *addr) {
    size_t offset = 0;

    while (true) {
      const tar_header *header = (const tar_header *) ((uintptr_t) addr + offset);

      if (header->filename[0] == '\0') {
        break;
      }

      size_t size = header->get_size();
      m_headers.push_back(header);

      offset += ((size / TAR_HEADER_ALIGNMENT) + 1) * TAR_HEADER_ALIGNMENT;
      if (size % TAR_HEADER_ALIGNMENT != 0) {
        offset += TAR_HEADER_ALIGNMENT;
      }
    }
  }

  const tar_header *tar_file::find_header(const std::string &filepath) const {
    auto archive_path = "./" + filepath;

    for (auto header : m_headers) {
      auto header_path = header->filename;

      if (strncmp(header_path, archive_path.c_str(), 100) == 0) {
        return header;
      }
    }

    return nullptr;
  }
}
```

---

## 🔌 The Asset Manager interface

We now are able to get the content of files within the embedded TAR archive from their path within the archive.

We’ll then expose those data via an Asset Manager, which will be in charge of loading and caching the resources:

```cpp
#pragma once

#include <string_view>
#include <string>
#include <vector>

#include <entt/entt.hpp>
#include <nlohmann/json.hpp>

#include <libasset/tar.hpp>
#include <libasset/bitmap.hpp>
#include <libasset/texture.hpp>
#include <libasset/music.hpp>
#include <libasset/sprite.hpp>
#include <libasset/shader.hpp>
#include <libasset/soundtrack.hpp>

#define LIBASSET_VERSION    1


namespace libasset {
  class asset_manager {
    private:
      tar_file m_archive;
      bitmap_cache m_bitmap_cache;
      texture_cache m_texture_cache;
      music_cache m_music_cache;
      sprite_cache m_sprite_cache;
      shader_cache m_shader_cache;
      soundtrack_cache m_soundtrack_cache;

    public:
      asset_manager();

      void setup(entt::registry &registry);
      void teardown(entt::registry &registry);

      // get raw text
      const std::string_view get_text(const std::string &resource_path);

      // parse JSON files
      nlohmann::json get_json(const std::string &resource_path);

      // load a BMP image into an SDL_Surface
      SDL_Surface *get_bitmap(const std::string &resource_path);

      // load a PNG, or BMP, or JPG, ... into an SDL_Texture (using SDL2_image)
      SDL_Texture *get_texture(entt::registry &registry, const std::string &resource_path);

      // load a WAV file into a Mix_Music (using SDL2_mixer)
      Mix_Music *get_music(entt::registry &registry, const std::string &resource_path);

      // read a JSON file defining some metadata about the sprite (width/height, origin, textures) and load the sub-assets accordingly
      sprite *get_sprite(entt::registry &registry, const std::string &resource_path);

      // read a JSON file defining some metadata (path to vertex/fragment shader source code) and build the GL Program
      shader *get_shader(entt::registry &registry, const std::string &resource_path);

      // read a JSON file defining some metadata (path to the tracks of the soundtrack) and load the sub-assets accordingly
      soundtrack *get_soundtrack(entt::registry &registry, const std::string &resource_path);
  };
}
```

**NB:** The pointers returned are owned by the Asset Manager and are not to be freed by the user (me). I should probably use `std::weak_ptr` here instead, but then SDL functions expecting a raw pointer would require extra work to interface with.

And the base implementation:

```cpp
#include <cstdint>
#include <exception>

#include <libasset.hpp>

static const uint8_t archive_bytes[] = {
  #include <libasset-data.inc>
};

namespace libasset {
  asset_manager::asset_manager() : m_archive(archive_bytes) {}

  void asset_manager::setup(entt::registry &registry) {
    auto manifest = get_json("manifest.json");

    if (manifest["version"].get<int>() != LIBASSET_VERSION) {
      throw std::runtime_error("Invalid libasset manifest version");
    }

    auto assets = manifest["assets"].get<std::vector<std::string>>();
    for (auto asset : assets) {
      auto metadata = get_json(asset + "/metadata.json");
      auto asset_type = metadata["type"].get<std::string>();

      if (asset_type == "sprite") {
        get_sprite(registry, asset);
      }
      else if (asset_type == "shader") {
        get_shader(registry, asset);
      }
      else if (asset_type == "soundtrack") {
        get_soundtrack(registry, asset);
      }
    }
  }

  void asset_manager::teardown(entt::registry &registry) {
    m_shader_cache.clear();
    m_sprite_cache.clear();
    m_texture_cache.clear();
    m_bitmap_cache.clear();
    m_soundtrack_cache.clear();
    m_music_cache.clear();
  }
}
```

---

## ⌛ Loading/Caching assets

The `get_text()` and `get_json()` do not cache the resources (maybe then should, it’s not a bottleneck for now), their implementation is therefore straightforward:

```cpp
const std::string_view asset_manager::get_text(const std::string &resource_path) {
  auto header = m_archive.find_header(resource_path);

  if (header != nullptr) {
    auto data = (const char *) header->get_data();
    auto size = header->get_size();

    return std::string_view(data, size);
  }

  return std::string_view();
}

nlohmann::json asset_manager::get_json(const std::string &resource_path) {
  auto header = m_archive.find_header(resource_path);

  if (header != nullptr) {
    auto first  = (const uint8_t *) header->get_data();
    auto last   = (const uint8_t *) ((uintptr_t) first + header->get_size());
    return nlohmann::json::parse(first, last);
  }

  return nullptr;
}
```

But now, we’ll take a look at a single example, loading and caching `SDL_Texture` structures.

For this, we will rely on the `entt::resource_cache<Type, Loader>` generic type provided by the **entt** library (for more information, see this [wiki page](https://github.com/skypjack/entt/wiki/Crash-Course:-resource-management)):

```cpp
#pragma once

#include <memory>

#include <entt/entt.hpp>
#include <SDL.h>
#include <SDL_image.h>


namespace libasset {
  struct texture_deleter {
    void operator()(SDL_Texture *texture);
  };

  struct texture_loader {
    using result_type = std::shared_ptr<SDL_Texture>;

    result_type operator()(entt::registry &registry, const void *addr, size_t size);
  };

  using texture_cache = entt::resource_cache<SDL_Texture, texture_loader>;
}
```

We define a resource cache with a custom loader, which will manage shared pointers to the `SDL_Texture` structure. Those shared pointers will have a custom deleter to interface properly with SDL.

The loader needs the address and size of the data, as well as a reference to the **entt** registry (for more information, see this [wiki page](https://github.com/skypjack/entt/wiki/Crash-Course:-entity-component-system#the-registry-the-entity-and-the-component)).

The registry is needed because it provides a dependency container, called a [context](https://github.com/skypjack/entt/wiki/Crash-Course:-entity-component-system#context-variables), which in our case holds a reference to the `SDL_Renderer`, needed to create `SDL_Texture` structures.

The implementation relies on [SDL\_RWOps](https://wiki.libsdl.org/SDL2/SDL_RWops) which abstracts away file I/O for SDL libraries:

```cpp
#include <exception>

#include <libasset/texture.hpp>

namespace libasset {
  void texture_deleter::operator()(SDL_Texture *texture) {
    if (texture != nullptr) {
      SDL_DestroyTexture(texture);
    }
  }

  texture_loader::result_type texture_loader::operator()(
    entt::registry &registry,
    const void *addr,
    size_t size
  ) {
    auto renderer = registry.ctx().get<SDL_Renderer *>();
    auto rw = SDL_RWFromConstMem(addr, size);

    if (rw != nullptr) {
      auto texture = IMG_LoadTexture_RW(renderer, rw, 1);

      if (texture != nullptr) {
        return texture_loader::result_type(texture, texture_deleter{});
      }
    }

    return nullptr;
  }
}
```

Now that our resource loader is implemented, our resource cache defined, we can implement the Asset Manager API:

```cpp
SDL_Texture *asset_manager::get_texture(
  entt::registry &registry,
  const std::string &resource_path
) {
  auto header = m_archive.find_header(resource_path);
  if (header != nullptr) {
    auto rsrc_id = entt::hashed_string(resource_path.c_str());
    auto res = m_texture_cache.load(
      rsrc_id,
      registry,
      header->get_data(),
      header->get_size()
    );

    auto tex = res.first->second;
    // tex is an entt::resource<T>
    // calling operator->() manually is the only way to get the pointer out of it
    return tex.operator->();
  }

  return nullptr;
}
```

The resource path is used as identifier for the cache, we get a resource handle, containing our shared pointer, as well as a boolean indicating if the resource was loaded (cache-miss), or not (cache-hit), but we don’t use it in this case.

The code for the other kind of resources is quite similar and will not be covered here.

---

## Conclusion

This concludes the first devlog for my game. There might be more in the future, stay tuned!

I’m having a lot of fun working on this project, but unfortunately I do not have a lot time to allocate to it (between work, and my other projects like [my own programming language Letlang](/posts/2022-11-24-letlang-road-to-v0-1), it’s a hard balance to maintain).

“Slow and steady” as they said 🙂

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
