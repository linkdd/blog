---
title: "Procedural Map Generation in C++ — Part 1: The slow, the bad and the ugly"
date: "2023-02-11T15:49:15Z"
description: "Warmonger Dynasty Devlog #6 : Procedural Map Generation Part 1"
tags: ["gamedev", "programming", "cpp", "procedural-generation", "devlog"]
author: "David Delassus"
images: ["img/image-d2792991b54fa020.jpg"]
---

![](img/image-d2792991b54fa020.jpg)

---

This is the 6th devlog of Warmonger Dynasty, a 4X turn-based strategy game made in C\+\+/SDL. If you want to read the previous entries, take a look at my [reading list](/reading-lists/warmonger-dynasty).

In my previous devlogs, I talked more about the engine than the game. I felt it was an interesting topic so that you can appreciate more the next devlogs. It is now time to tackle the game 🙂

Every 4X strategy game needs a map, you can have a handcrafted map like the Total War Warhammer series, or a random one like Civilization.

Let’s start by saying that I’m not very good at level design, or art. Therefore, I choosed to go with a random one.

**NB:** For the art, I’m using assets bought on the Unity Store from [David Baumgart](https://assetstore.unity.com/publishers/13841).

---

## Part 1 : The map data structure

![](img/image-dd94b6153be4de3f.png)

Because [hexagons are the bestagons](https://www.youtube.com/watch?v=thOifuHs6eY), I’m using a hexagonal grid for the map, with pointy-top hexagons.

Therefore, I implemented, thanks to [this great article](https://www.redblobgames.com/grids/hexagons/), a hexagonal coordinate system and storage:

- `hex::coords::cubic` : Cubic coordinates ( `q;r;s` ), used for path finding, movement/vision range, field of view, and other cool algorithms
- `hex::coords::offset` : Offset coordinates ( `x;y` ), used for indexing tiles
- `hex::storage<T>` : To store tile data (of type `T`) and access it through offset coordinates

I’ll skip the implementation details since the code is heavily inspired by the article I linked above (it provides a C\+\+, Python and Javascript implementation and links to other implementations in C#, Java, Rust, etc…).

This allows me to easily create tilemaps like this:

```cpp
auto tilemap = hex::storage<float>(width, height);

for (int row = 0; row < height; row++) {
  for (int col = 0; col < width; col++) {
    auto coord = hex::coords::offset(col, row);
    tilemap.at(coord) = 0.5;
  }
}

auto some_coord = hex::coords::offset(x, y);
if (tilemap.contains(some_coord) {
  std::optional<float> &tile_data = tilemap.at(some_coord);
  // ...
}
```

---

## Part 2 : Splitting the map into biomes

![](img/image-2923a6f56a04145d.png)

I want to use different tilesets on different parts of the map, in order to have a bit of diversity. Therefore, I need to divide the map into areas, and assign an area index to each tile of the map.

To do this, I chose to use a [Voronoi diagram](https://en.wikipedia.org/wiki/Voronoi_diagram). The algorithm is quite simple.

### Step 1, divide the map into “quadrants”

![](img/image-d658704219e7673b.png)

This is done with those 4 lines of code:

```cpp
class biome_generator {
  private:
    int m_width;
    int m_height;

  // ...
};

void biome_generator::gen_biome_seeds() {
  int quadrant_x_divisor = 6;
  // map smaller than 60 tiles of height have less quadrants
  // this ensures that the areas have a decent size
  int quadrant_y_divisor = (m_height >= 60 ? 6 : 3);

  int quadrant_width  = m_width / quadrant_x_divisor;
  int quadrant_height = m_height / quadrant_y_divisor;

  // ...
}
```

### Step 2, choose a random point in each quadrant

![](img/image-572eecaaba193c59.png)

Using the `<random>` header from C\+\+11, and a `seed` parameter:

```cpp
class biome_generator {
  private:
    int m_seed;

    std::vector<SDL_Point> m_biome_seeds;

  // ...
};

void biome_generator::gen_biome_seeds() {
  // ...

  std::default_random_engine gen_rd(m_seed);
  std::uniform_int_distribution<> rd_x(0, quadrant_width);
  std::uniform_int_distribution<> rd_y(0, quadrant_height);

  for (int x = 0; x < quadrant_x_divisor; x++) {
    for (int y = 0; y < quadrant_y_divisor; y++) {
      m_biome_seeds.push_back(SDL_Point{
        .x = rd_x(gen_rd) + x * quadrant_width,
        .y = rd_y(gen_rd) + y * quadrant_height
      });
    }
  }
}
```

### Step 3: Assign to each tile the index of the biome

The biome index correspond to the index of the seed in the `biome_seeds` vector. To every tile, we will assign the index of the closest seed.

Let’s define a function to find the closest seed first:

```cpp
using biome_index_type = size_t;

biome_index_type biome_generator::get_closest_biome_seed(
  hex::coords::offset tile_coord
) {
  biome_index_type closest_seed_index = 0;
  int closest_seed_dist = INT32_MAX;

  for (
    biome_index_type seed_index = 0;
    seed_index < m_biome_seeds.size();
    seed_index++
  ) {
    auto seed = m_biome_seeds[seed_index];

    // the distance() function uses cubic coordinates
    // oddr_to_cubic() converts offset coordinates to cubic coordinates
    // "oddr" -> odd rows are offsetted to the right
    auto coord = hex::coords::offset(seed.x, seed.y).oddr_to_cubic();
    auto dist = coord.distance(tile_coord.oddr_to_cubic());

    if (dist < closest_seed_dist) {
      closest_seed_index = seed_index;
      closest_seed_dist  = dist;
    }
  }

  return closest_seed_index;
}
```

And then assign a biome index to every tile:

```cpp
class biome_generator {
  private:
    hex::storage<biome_index_type> m_biome_index_map;

  // ...
};

void biome_generator::gen_biome_index_map() {
  for (int row = 0; row < m_height; row++) {
    for (int col = 0; col < m_width; col++) {
      auto tile_coord = hex::coords::offset(col, row);
      auto tile_biome = get_closest_biome_seed(tile_coord);

      m_biome_index_map.at(tile_coord) = tile_biome;
    }
  }
}
```

---

## Part 2: Assigning a biome type to every biome

Now that our map is divided into multiple areas, we need to choose a type of biome for every area.

### Step 1: Make some noise

We could choose randomly, but this mean that we could have situations like a volcanic biome right next to a cold land biome. Not very realistic.

The second option is to use a **noise function**. Like most game developers, we will use Perlin Noise here.

![](img/image-75b393b3edb29b8a.png)

Once again, I’ll skip the details of implementing the Perlin Noise function (mostly because it feels like black magic to me). My implementation is inspired by this header-only library that I recommend:

{{< cta-link
  url="https://github.com/Reputeless/PerlinNoise"
  title="GitHub - Reputeless/PerlinNoise: Header-only Perlin noise library for modern C++ (C++17/C++20)"
  description="siv::PerlinNoise is a header-only Perlin noise library for modern C++ (C++17/20). The implementation is based on Ken…"
>}}

With this implementation, I can generate values between 0 and 1, I’ll use this to generate 3 noise maps:

- a height map
- a heat map
- a humidity map

Each tile will have its own height/heat/humidity value:

```cpp
class biome_generator {
  private:
    hex::storage<float> m_height_map;
    hex::storage<float> m_heat_map;
    hex::storage<float> m_humidity_map;

  // ...
};

void biome_generator::gen_noise_maps() {
  auto noise = math::perlin_noise(m_seed);
  int noise_octaves = 1;
  float noise_persistance = 0.5;
  float noise_scale = 3;

  for (int row = 0; row < m_height; row++) {
    for (int col = 0; col < m_width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      auto noise_x = col / (float) m_width;
      auto noise_y = row / (float) m_height;

      auto tile_height = noise.octave3D_01(
        math::vec3(noise_x, noise_y, 0.34567) * noise_scale,
        noise_octaves,
        noise_persistance
      );
      auto tile_heat = noise.octave3D_01(
        math::vec3(noise_x, noise_y, 0.45678) * noise_scale,
        noise_octaves,
        noise_persistance
      );
      auto tile_humidity = noise.octave3D_01(
        math::vec3(noise_x, noise_y, 0.56789) * noise_scale,
        noise_octaves,
        noise_persistance
      );

      m_height_map.at(tile_coord)   = tile_height;
      m_heat_map.at(tile_coord)     = tile_heat;
      m_humidity_map.at(tile_coord) = tile_humidity;
    }
  }
}
```

### Step 2: Averages

Then, we compute the average height, heat and humidity of each biome. To achieve this, we can keep track of the value during generation in a vector:

```cpp
#include <algorithm>
#include <numeric>

static float average(const std::vector<float> &vals) {
  auto sum = std::reduce(
    vals.begin(),
    vals.end(),
    (float) 0,
    std::plus<float>()
  );

  return sum / vals.size();
}

class biome_data {
  private:
    std::vector<float> m_height_nodes;
    std::vector<float> m_heat_nodes;
    std::vector<float> m_humidity_nodes;

    float m_average_height;
    float m_average_heat;
    float m_average_humidity;

  public:
    void add_node(float height, float heat, float humidity) {
      m_height_nodes.push_back(height);
      m_heat_nodes.push_back(heat);
      m_humidity_nodes.push_back(humidity);
    }

    void compute_averages() {
      m_average_height = average(m_height_nodes);
      m_average_heat = average(m_heat_nodes);
      m_average_humidity = average(m_humidity_nodes);
    }

    float average_height() const { return m_average_height; }
    float average_heat() const { return m_average_heat; }
    float average_humidity() const { return m_average_humidity; }
};
```

Using this class, we modify the `gen_noise_maps` function:

```cpp
class biome_generator {
  private:
    std::unordered_map<biome_index_type, biome_data> m_biomes_data;

  // ...
};

void gen_noise_maps() {
  // ...

  for (int row = 0; row < m_height; row++) {
    for (int col = 0; col < m_width; col++) {
      // ...

      auto &tile_biome_data = m_biomes_data[tile_biome];
      tile_biome_data.add_node(tile_height, tile_heat, tile_humidity);
    }
  }
}
```

Our `biome_generator` constructor now looks like this:

```less
biome_generator::biome_generator(int seed, int width, int height)
  : m_seed(seed),
    m_width(width),
    m_height(height),
    m_biome_map(width, height),
    m_height_map(width, height),
    m_heat_map(width, height),
    m_humidity_map(width, height)
{
  gen_biome_seeds();

  // i actually merged those 2 together
  // but for clarity, let's keep them separate
  gen_biome_index_map();
  gen_noise_maps();

  for (auto &[_, biome_data] : m_biomes_data) {
    biome_data.compute_averages();
  }

  gen_biome_map();
}
```

### Step 3: Choosing the right biome type

First, let’s define an enumeration:

```kotlin
enum class biome_type {
  temperate,
  tropical,
  red_desert,
  yellow_desert,
  cold_land,
  volcanic
};
```

I could write a bunch of `if`/`else` using the average height/heat/humidity of the biome and return the correct `biome_type` value. But it’s:

- hard to maintain
- hard to tweak
- hard to read

Instead, I want to map ranges of values to a biome type. Something like this:

![](img/image-6cdb31d4af946c2d.png)

**NB:** I omitted the `height` in this diagram for clarity. Also the values depicted in that diagram are not the ones I used in the game.

I’ve not yet found a better to serialize this as JSON than the following:

```json
[
  {
    "biome": "volcanic",
    "requires": {
      "height": 0.7,
      "heat": 0.8,
      "humidity": 0
    }
  },
  {
    "biome": "cold_land",
    "requires": {
      "height": 0.7,
      "heat": 0,
      "humidity": 0
    }
  },
  {
    "biome": "volcanic",
    "requires": {
      "height": 0,
      "heat": 0.9,
      "humidity": 0
    }
  },
  {
    "biome": "tropical",
    "requires": {
      "height": 0,
      "heat": 0.5,
      "humidity": 0.5
    }
  },
  {
    "biome": "yellow_desert",
    "requires": {
      "height": 0,
      "heat": 0.5,
      "humidity": 0.4
    }
  },
  {
    "biome": "red_desert",
    "requires": {
      "height": 0,
      "heat": 0.5,
      "humidity": 0
    }
  },
  {
    "biome": "temperate",
    "requires": {
      "height": 0,
      "heat": 0.3,
      "humidity": 0
    }
  },
  {
    "biome": "cold_land",
    "requires": {
      "height": 0,
      "heat": 0,
      "humidity": 0
    }
  }
]
```

**NB:** Yes they can overlap. Yes I need at least one where each requirements are 0\. Yes there are probably better solutions, but this one works fine for now.

Using this data structure, we take all the entries where the average values of `height`/`heat`/`humidity` are greater (or equal) than the entry’s `requires`.

Then we do a lexicographic sort of the filtered entries, and take the last one, which is the closest to the average values.

The code looks like this:

```cpp
#include <nlohmann/json.hpp>
#include <entt/entt.hpp>

class biome_selector {
  private:
    struct selector_config {
      biome_type biome;
      float min_height;
      float min_heat;
      float min_humidity;
    };

    std::vector<selector_config> selectors;

  public:
    static void load(const nlohmann::json &data) {
      biome_selector instance;

      // deserialization

      entt::locator<biome_selector>::emplace(instance); // singleton
    }

    static biome_type get_biome_type(float height, float heat, float humidity) {
      auto &instance = entt::locator<biome_selector>::value();
      decltype(instance.selectors) matches;

      std::copy_if(
        instance.selectors.begin(),
        instance.selectors.end(),
        std::back_inserter(matches),
        [&](const auto &selector) {
          return (
            height >= selector.min_height
            && heat >= selector.min_heat
            && humidity >= selector.min_humidity
          );
        }
      );

      std::sort(
        matches.begin(),
        matches.end(),
        [&](const auto &lhs, const auto &rhs) {
          if (lhs.min_height == rhs.min_height) {
            if (lhs.min_heat == rhs.min_heat) {
              return lhs.min_humidity < rhs.min_humidity;
            }

            return lhs.min_heat < rhs.min_heat;
          }

          return lhs.min_height < rhs.min_height;
        }
      );

      auto &selector = matches.back();
      return selector.biome;
    }
};
```

Finally, we can use this in our `biome_generator` class:

```cpp
void biome_generator::gen_biome_map() {
  for (int row = 0; row < m_height; row++) {
    for (int col = 0; col < m_width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      auto biome_index = m_biome_index_map.at(tile_coord).value();
      auto &biome_data = m_biomes_data.at(biome_index);
      m_biome_map.at(tile_coord) = biome_selector::get_biome_type(
        biome_data.average_height(),
        biome_data.average_heat(),
        biome_data.average_humidity()
      );
    }
  }
}

biome_type biome_generator::get_biome_type(
  hex::coords::offset tile_coord
) const {
  return m_biome_map.at(tile_coord).value();
}
```

This gives the following result:

[View the embedded animation on Gfycat](https://gfycat.com/ifr/thatjoyfulgreatargus)

---

## Part 3 : Choosing the right tile

I tried reusing the height/heat/humidity information to choose a tile type within a biome, but it led to not very natural maps, and it was hard to tweak due to the number of tiles:

![](img/image-0537bd14ad0afa07.png)

Basically, I want to be able to define what kind of tile can be adjacent to what kind of tile. For example, I can have plains adjacent to hills, hills adjacent to highlands, and highlands adjacent to mountains, but mountains should not be adjacent to plains.

Based on those rules, I want the algorithm to choose a tile type that fits within its neighbors.

Fortunately, this algorithm already exists: **Wave Function Collapse**

![](img/image-8e0483c7b748a34a.gif)

You can read more about this algorithm on this repository:

{{< cta-link
  url="https://github.com/mxgmn/WaveFunctionCollapse"
  title="GitHub - mxgmn/WaveFunctionCollapse: Bitmap & tilemap generation from a single example with the…"
  description="This program generates bitmaps that are locally similar to the input bitmap. Local similarity means that (C1) The…"
>}}

### Step 1: Encoding the adjacency rules

In the original implementation, the adjacency rules are inferred from an example image. But here, we will skip that part and define the rules in a JSON asset.

I have one enumeration per biome to hold the tile types:

```cpp
enum class temperate_tile_type {
  // ...
};

enum class tropical_tile_type {
  // ...
};

enum class red_desert_tile_type {
  // ...
};

enum class yellow_desert_tile_type {
  // ...
};

enum class cold_land_tile_type {
  // ...
};

enum class volcanic_tile_type {
  // ...
};

using tile_type = std::variant<
  temperate_tile_type,
  tropical_tile_type,
  red_desert_tile_type,
  yellow_desert_tile_type,
  cold_land_tile_type,
  volcanic_tile_type
};
```

And some boilerplate code to map string names in the JSON file to the enum values.

The JSON has the following structure:

```json
{
  "<biome type>": {
    "<tile type>": [
      "<tile type>",
      "<tile type>"
    ]
  }
}
```

For each biome type, I define the valid adjacent tiles for each tile type.

For clarity, I made myself a few matrices in a spreadsheet so that I can visualize that structure:

![](img/image-6594e51598d6da7f.png)

Tropical biome adjacency matrix

![](img/image-81888c122be39e69.png)

Red Desert biome adjacency matrix

Similar to the previous part, I encapsulate this information in a singleton class:

```cpp
using valid_tile_types = std::vector<tile_type>;

namespace details {
  template <typename TileType>
  struct tile_selection_rules {
    static_assert(sizeof(TileType) == 0, "unsupported tile type");
  };

  template <>
  struct tile_selection_rules<temperate_tile_type> {
    valid_tile_types dirt;
    // boilerplate: same field for each tile type
  };

  // boilerplate: same template specialization for each biome type
}

class tile_selection_rules {
  private:
    details::tile_selection_rules<temperate_tile_type> m_temperate;
    details::tile_selection_rules<tropical_tile_type> m_tropical;
    details::tile_selection_rules<red_desert_tile_type> m_red_desert;
    details::tile_selection_rules<yellow_desert_tile_type> m_yellow_desert;
    details::tile_selection_rules<cold_land_tile_type> m_cold_land;
    details::tile_selection_rules<volcanic_tile_type> m_volcanic;

  public:
    static void load(const nlohmann::json &data) {
      tile_selection_rules instance;

      // boilerplate: deserialization

      entt::locator<tile_selection_rules>::emplace(instance);
    }

    static const valid_tile_types &get_valid_neighbors(temperate_tile_type type) {
      auto &instance = entt::locator<tile_selection_rules>::value();

      switch (type) {
        case temperate_tile_type::dirt:
          return instance.m_temperate.dirt;

        // boilerplate: same things for other tile types
      }

      return {};
    }

    // boilerplate: same things for toher biome types
    static const valid_tile_types &get_valid_neighbors(tropical_tile_type type);
    static const valid_tile_types &get_valid_neighbors(red_desert_tile_type type);
    static const valid_tile_types &get_valid_neighbors(yellow_desert_tile_type type);
    static const valid_tile_types &get_valid_neighbors(cold_land_tile_type type);
    static const valid_tile_types &get_valid_neighbors(volcanic_tile_type type);
};
```

Lots of boilerplate here, but I did not want to put too much effort in this part of the code, because I plan to add modding to the game later on, and this will likely go into the trash, replaced by a Lua API probably.

Anyway, this code allows me to easily get the valid adjacent tiles:

```cpp
tile_type type = /* ... */;

auto &valid_neighbors = std::visit(
  [](auto t) -> const valid_tile_types& {
    return tile_selection_rules::get_valid_neighbors(t);
  },
  type
);
```

### Step 2: The algorithm in a nutshell

Wave Function Collapses is basically a constraints based algorithm, similar to a Sudoku solver.

![](img/image-e20a70a8bbc93120.png)

We have a set of candidates, when there is one remaining option for a cell, we “collapse” it and then update the available candidates on the rest of the grid. We repeat that process until all cells are collapsed.

Let’s define a structure to store information about our candidates:

```cpp
struct tile_candidate {
  bool collapsed;
  std::vector<tile_type> options;
  hex::coords::offset coord;

  int entropy() const {
    return options.size();
  }

  template <typename URDG>
  void collapse(URDG &rd_gen) {
    collapsed = true;

    std::vector<tile_type> collapsed_options;
    std::sample(
      options.begin(),
      options.end(),
      std::back_inserter(collapsed_options),
      1,
      rd_gen
    );

    options = collapsed_options;
  }
};
```

Then, let’s define a class that will encapsulate our algorithm:

```cpp
class tile_generator {
  private:
    int m_seed;
    int m_width;
    int m_height;

    // we will use this to get the biome type of a tile
    const biome_generator &m_biome_generator;

    // we will use this when choosing a tile to collapse
    std::default_random_engine m_rd_gen;

    std::vector<tile_candidate> m_tile_candidates;
    hex::storage<tile_type> m_tilemap;

  public:
    tile_generator(
      int seed,
      int width,
      int height,
      const biome_generator &biome_generator
    ) : m_seed(seed),
        m_width(width),
        m_height(height),
        m_biome_generator(biome_generator),
        m_rd_gen(m_seed),
        m_tilemap(width, height)
    {
      // ...
    }

  // ...
};
```

I’ll also create a function which returns the available tiles for a biome, I’ll skip its implementation as it’s only boilerplate again:

```cpp
const std::vector<tile_type> &biome_tiles(biome_type biome);
```

First, we need to initialize our candidates:

```cpp
void tile_generator::initialize_candidates() {
  for (int row = 0; row < m_height; row++) {
    for (int col = 0; col < m_width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      auto biome = m_biome_generator.get_biome_type(tile_coord);
      m_tile_candidates.push_back(tile_candidate{
        .collapsed = false,
        .options = biome_tiles(biome),
        .coord = tile_coord
      });
    }
  }
}
```

Every iteration of the algorithm, we will collapse one of the tile with the least entropy, and then propagate the constraint through the map (removing options from other candidates).

To do this, we’ll need a function to get one of the tile with the least entropy:

```cpp
tile_candidate* tile_generator::get_least_entropy_candidate() {
  std::vector<tile_candidate*> non_collapsed_candidates;

  // complexity is O(n)
  for (auto &candidate : m_tile_candidates) {
    if (!candidate.collapsed) {
      non_collapsed_candidates.push_back(&candidate);
    }
  }

  // complexity is O(n*log(n))
  std::sort(
    non_collapsed_candidates.begin(),
    non_collapsed_candidates.end(),
    [](tile_candidate *a, tile_candidate *b) {
      return a->entropy() < b->entropy();
    }
  );

  // now, tiles with the least entropy are at the front of the list
  // we want the index of the first tile with a bigger entropy
  // so that we have an upper bound for the random choice

  auto least_entropy = non_collapsed_candidates.front()->entropy();
  size_t not_least_entropy_idx = 0;

  for (auto candidate : non_collapsed_candidates) {
    not_least_entropy_idx++;

    if (candidate->entropy() != least_entropy) {
      break;
    }
  }

  // we want [0; not_least_entropy_idx) and not [0; not_least_entropy_idx]
  // semi-closed range [) vs closed range []
  std::uniform_int_distribution<> rd_idx(0, not_least_entropy_idx - 1);
  auto idx = rd_idx(m_rd_gen);
  return non_collapsed_candidates.at(idx);
}
```

**NB:** I could have copy the tiles with the least entropy in a new `std::vector`, then use `std::shuffle()` on that vector, and return the front element. But the shuffle function is linear in complexity as well. Since this function will be called once per iteration, it’s already expensive.

Also, you might be concerned by “what if all the tiles are collapsed?”. This function would not be called in that case, so I can skip the check.🙂

Now, for each iteration, I can collapse a tile:

```cpp
auto least_entropy_candidate = get_least_entropy_candidate();
least_entropy_candidate->collapse(m_rd_gen);
```

Once collapsed, we need to propagate the constraints by going through the grid and update the available options for each non-collapsed candidates:

```cpp
void tile_generator::propagate_constraints() {
  std::vector<tile_candidate> updated_candidates;
  updated_candidates.reserve(m_tiles_candidates.size());

  // complexity is O(n)
  for (auto &candidate : m_tile_candidates) {
    if (candidate.collapsed) {
      // nothing to do here
      updated_candidates.push_back(candidate);
    }
    else {
      auto biome = m_biome_generator.get_biome_type(candidate.coord);
      // we start with all the options available
      auto updated_options = biome_tiles(biome);

      // we get the coordinates of the neighbords
      auto cubic_coord = candidate.coord.oddr_to_cubic();
      auto n0 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<0>());
      auto n1 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<1>());
      auto n2 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<2>());
      auto n3 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<3>());
      auto n4 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<4>());
      auto n5 = hex::coords::offset::oddr_from_cubic(cubic_coord.neighbor<5>());

      // and we filter the options based on the neighbors tiles
      filter_options_for_neighbor(n0, biome, updated_options);
      filter_options_for_neighbor(n1, biome, updated_options);
      filter_options_for_neighbor(n2, biome, updated_options);
      filter_options_for_neighbor(n3, biome, updated_options);
      filter_options_for_neighbor(n4, biome, updated_options);
      filter_options_for_neighbor(n5, biome, updated_options);

      updated_candidates.emplace_back(tile_candidate{
        .collapsed = false,
        .options = updated_options,
        .coord = candidate.coord
      });
    }
  }

  m_tile_candidates = updated_candidates;
}
```

The magic happens in the `filter_options_for_neighbor()` function, it will mutate the `updated_options` list, leaving only the valid options in it (and thus, decreasing the entropy of the tile candidate).

**NB:** An hexagonal tile have 6 neighbors, the cubic coordinates allow us to compute them very easily.

Let’s implement our filter function:

```rust
void tile_generator::filter_options_for_neighbor(
  hex::coords::offset neighbor_coord,
  biome_type biome,
  std::vector<tile_type> &updated_options
) {
  if (
    // make sure the coordinates are not out of bounds
    m_tilemap.contains(neighbor_coord)
    // make sure we are comparing with a tile in the same biome
    && biome == m_biome_generator.get_biome_type(neighbor_coord)
  ) {
    // remove all the options that do not satisfy the constraints
    std::remove_if(
      updated_options.begin(),
      updated_options.end(),
      [&](tile_type &option) {
        auto neighbor_index = neighbor_coord.row * m_width + neighbor_coord.col;
        auto &neighbor = m_tile_candidates.at(neighbor_index);

        valid_tile_types valid_options;

        // aggregates all the valid options
        for (auto &neighbor_option : neighbor.options) {
          auto &valid_neighbor_options = std::visit(
            [](auto option) -> const valid_tile_types& {
              return tile_selection_rules::get_valid_neighbors(option);
            },
            neighbor_option
          );

          std::copy(
            valid_neighbor_options.begin(),
            valid_neighbor_options.end(),
            std::back_inserter(valid_options)
          );
        }

        // remove the option if it is not found in the valid ones
        return std::find(
          valid_options.begin(),
          valid_options.end(),
          option
        ) == valid_options.end();
      }
    );
  }
}
```

### Step 3: Tying things together

Now, our algorithm looks like this:

```cpp
void tile_generator::collapse_grid() {
  // we collapse one tile per iteration
  // so we have at most n iterations where n is the number of tiles
  // also, the complexity is O(n)
  for (int iteration = 0; iteration < m_tile_candidates.size(); iteration++) {
    auto least_entropy_candidate = get_least_entropy_candidate();
    least_entropy_candidate->collapse(m_rd_gen);

    propagate_constraints();
  }

  for (auto &candidate : m_tile_candidates) {
    // for now, we skip the need for backtracking by choosing a default tile
    if (candidate.options.empty()) {
      m_tilemap.at(candidate.coord) = temperate_tile_type::ocean;
    }
    else {
      m_tilemap.at(candidate.coord) = candidate.options.front();
    }
  }
}
```

In our current implementation, it is possible to have a tile where every single option is invalid. In a Sudoku solver, we’d need to backtrack until the moment we made a guess, and invalidate that option. It would be the same here, but for now, let’s use a default tile.

And here is a few results of the algorithm:

![](img/image-5e461d6a35429400.png)![](img/image-2bdeb4cfceced908.png)

---

## Problem 1: The slow

Our implementation is very naïve, and therefore very slow, here is a few numbers:

- in debug mode: one iteration takes **2** to **3** seconds (thank you Windows for the bloat)
- in release mode: one iteration takes **40** to **50** milliseconds
- the smallest map size have **1144** tiles
- the biggest map size have **6996** tiles

This means that:

- the generation for the smallest map takes at most **1 hour** in debug mode, and **1 minute** in release mode
- the generation for the biggest map takes at most **6 hours** in debug mode, and **6 minutes** in release mode

This is very very slow.

---

## Problem 2: The bad

The current implementation does not support:

- backtracking, which means some seed might generate “unsolvable” maps
- weighted adjacency, at the moment, all options have the same probability of being chosen, which leads to lone tiles that do not make a lot sense, and feels unnatural

It could be nice to reuse as well the noise maps from the previous step to influence the tile selection when collapsing a candidate.

---

## Problem 3: The ugly

Let’s be honest, the result looks bad. Really bad. Compare them to a Civilization VI map:

![](img/image-e2c32ff831a9ed40.jpg)

I need a whole new algorithm if I want to achieve the same quality.

---

## Conclusion

I learned a lot during this process:

- Voronoi diagrams are fun and easy
- noise function for map generation are not enough to get a good, natural looking, map
- optimizations in Wave Function Collapse are **NOT** premature

While I had fun implementing this, I might throw the code to the trash and rethink the whole process, the results are just not what I am looking for.

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 1 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for the game is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here → [Devlogs Reading List](/reading-lists/warmonger-dynasty)
