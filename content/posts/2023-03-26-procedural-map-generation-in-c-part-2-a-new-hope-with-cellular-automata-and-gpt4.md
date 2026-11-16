---
title: "Procedural map generation in C++ — Part 2: A new hope with cellular automata and GPT4"
date: "2023-03-26T21:11:54Z"
description: "Warmonger Dynatsy devlog #7: Cellular automata, simulation and GPT4"
tags: ["programming", "cpp", "gamedev", "procedural-generation", "gpt-4"]
author: "David Delassus"
images: ["img/image-fd9b4ae1c7c3ccd5.jpg"]
---

![](img/image-fd9b4ae1c7c3ccd5.jpg)

---

This is the 7th devlog of Warmonger Dynasty, a 4X turn-based strategy game made in C\+\+/SDL. If you want to read the previous entries, take a look at my [reading list](/reading-lists/warmonger-dynasty).

In my [previous devlog](/posts/2023-02-11-procedural-map-generation-in-c-part-1-the-slow-the-bad-and-the-ugly), we talked about procedural map generation. We tried a combination of Voronoi diagrams, Perlin Noise, and Wave Function Collapse, but unfortunately got no good looking results.

For this part, I decided to try the **GPT4** version of [ChatGPT](https://chat.openai.com) (which costs 20$/month), just to see what the fuss was about. **Spoilers:** it’s very useful if you know how to write your prompt efficiently.

Also, since the generation code was getting more and more complex, I had to write a debugging tool to see the data for each intermediate step. I used Python and Tkinter for this purpose, as well as `matplotlib`, `numpy` and `Pillow`.

---

## Prelude: Code organization

I want to be able to easily add or remove steps to the generation process, and rearrange them.

For this purpose, I created 2 structures, 1 class and 1 abstract class:

- `config`: this structure will hold the parameters of the generation algorithm, such as the seed, width/height of the map, …
- `context`: this structure will hold all the data used by the generation algorithm, such as the height map, heat map, …
- `process`: this class will have a list of tasks to execute, and simply run them sequentially

```cpp
namespace map::gen {
  struct config {
    int seed;
    int width;
    int height;
    // ...
  };

  struct context {
    hex::storage<bool>            landmass_map;
    hex::storage<int>             water_proximity_map;
    hex::storage<size_t>          tectonic_map;
    hex::storage<float>           elevation_map;
    hex::storage<float>           heat_map;
    hex::storage<tw::math::vec2>  wind_map;
    hex::storage<float>           humidity_map;
    hex::storage<biome_type>      biome_map;
    hex::storage<tile_data>       tile_map;

    context(const config &cfg); // constructs the hex::storage<T> instances
  };

  class process {
    private:
      config m_config;
      context m_context;

    public:
      process(const config &cfg);

      void generate() {
        using task_type = void (*)(const config&, context&);
        std::vector<task_type> tasks = {
          // ...
        };

        for (auto &task : tasks) {
          task(m_config, m_context);
        }
      }

      const context &get_context() const;
  };
}
```

In my `map_data` class, I can reuse this easily:

```cpp
void map_data::generate() {
  auto map_gen = gen::process(gen::config{
    .seed = m_config.seed,
    .width = width(),
    .height = height()
  });

  map_gen.generate();
  auto &gen_context = map_gen.get_context();

  for (int col = 0; col < width(); col++) {
    for (int row = 0; row < height(); row++) {
      auto tile_coord = hex::coords::offset(col, row);

      m_tiles.at(tile_coord) = gen_context.tile_map.at(tile_coord);
    }
  }
}
```

---

## Part 1: Landmass generation

The main problem I had with the previous algorithm was that I had no control over the shape of the landmass. Here, as a start, I want 2 continents separated by water.

For this purpose, I knew I wanted a cellular automata to grow land from seed coordinates. I decided to ask **GPT4**.

My prompt included a short explanation of the data structures I am using:

- `hex::coords::cubic` for the cubic hex coordinates
- `hex::coords::offset` for the offset hex coordinates
- `hex::storage<T>` for the tile data storage

The prompt was then finished with the question:

> The first step of the procedural map generation is to generate continents. I want the result of this step to be a `hex::storage<bool>` where `true` means that the tile at those coordinates is land, and `false` means that the tile at those coordinates is water.

GPT4 then proceeded to provide an implementation using Perlin Noise in a function named `generate_continents`. That is my fault, I forgot to say I wanted a cellular automata:

> The problem with Perlin Noise is that I cannot control the shape, size, or number of continents that will be generated. I’d like the `generate_continents` function to accept as parameter the number of continents that needs to be present on the map. The function should then generate “seeds” for every continent, and grow land from those seeds.

Here, it required a bit of back-and-forth as it would stop midway of completing the code, and was not using some of the parameters I wanted (it ignored the seeds). Finally, it provided the following code:

```cpp
// Grow land around land tiles using cellular automata
void grow_land(hex::storage<bool>& continents, int growth_cycles, double growth_chance) {
  std::random_device rd;
  std::mt19937 gen(rd());
  std::uniform_real_distribution<> chance_distr(0.0, 1.0);

  for (int cycle = 0; cycle < growth_cycles; ++cycle) {
    hex::storage<bool> new_continents = continents;

    for (int q = 0; q < continents.height(); ++q) {
      for (int r = 0; r < continents.width(); ++r) {
        hex::coords::offset offset_coords(q, r);
        hex::coords::cubic cubic_coords = hex::coords::offset_to_cubic(offset_coords);

        if (continents.at(offset_coords)) {
          // Spread land to neighbors
          for (const auto& neighbor : cubic_coords.neighbors()) {
            hex::coords::offset neighbor_offset = hex::coords::cubic_to_offset(neighbor);
            if (continents.contains(neighbor_offset) && !continents.at(neighbor_offset)) {
              if (chance_distr(gen) < growth_chance) {
                new_continents.at(neighbor_offset) = true;
              }
            }
          }
        }
      }
    }

    continents = new_continents;
  }
}

// Generate continents using seeds and grow function
hex::storage<bool> generate_continents(int width, int height, int num_continents, int growth_cycles, double growth_chance) {
  hex::storage<bool> continents(width, height);

  // Generate seeds
  std::vector<hex::coords::cubic> seeds = generate_seeds(num_continents, width, height);

  // Set seeds as land
  for (const auto& seed : seeds) {
    hex::coords::offset offset_seed = hex::coords::cubic_to_offset(seed);
    continents.at(offset_seed) = true;
  }

  // Grow land around seeds
  grow_land(continents, growth_cycles, growth_chance);

  return continents;
}
```

This is a very good start. But every tile has the same chance of being turned to land, so I told GPT4:

> The `growth_chance` should decrease the farthest away we are from a seed. Can you use the method `distance()` of the `hex::coords::cubic` class for this?

And it provided this modification:

```cpp
auto nearest_seed_distance = [&](const hex::coords::cubic& coords) {
  int min_distance = std::numeric_limits<int>::max();
  for (const auto& seed : seeds) {
    int distance = coords.distance(seed);
    min_distance = std::min(min_distance, distance);
  }
  return min_distance;
};

// ...

int distance = nearest_seed_distance(neighbor);
double adjusted_growth_chance = max_growth_chance / (1.0 + distance);

if (chance_distr(gen) < adjusted_growth_chance) {
  new_continents.at(neighbor_offset) = true;
}
```

Still not satisfied with the answer, I asked:

> The probability of a tile very far from any seed to be land is too high, can you provide another implementation for the value of `adjusted_growth_chance`? I don’t need you to update the `grow_land` function, only provide the new `adjusted_growth_chance` implementation.

**NB:** I often ask ChatGPT to limit what code it produces, as to avoid it getting confused or stopping midway.

And its answer:

```cpp
double decay_factor = 2.0; // Adjust this value to control the rate of decay
double adjusted_growth_chance = max_growth_chance * std::exp(-distance / decay_factor);
```

With the following explanation:

> By using this exponential decay function, the growth chance will decrease more rapidly as the distance from the nearest seed increases. You can control the rate of decay by adjusting the `decay_factor`. A higher value will result in a slower decay, while a lower value will result in a faster decay.

Great, now let’s get down to refactoring this so that it fits in my project. I intentionally omitted a lot of details regarding my project in order to avoid confusing the bot.

And we get the following result:

![](img/image-5af33ced96d4928c.png)

A bunch of islands with natural looking borders (the mountains are the seed from which the land grew).

From here, I decide to select 2 areas of the map where I will generate a bunch of random seeds, which together will form the final continents. I also want to remove tiles that are entirely surrounded with water.

The final code looks like this:

```cpp
void landmass_task(const config &cfg, context &ctx) {
  std::default_random_engine rd_gen(cfg.seed);

  // set all map as "water"
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord = hex::coords::offset(col, row);
      ctx.landmass_map.at(tile_coord) = false;
    }
  }

  int border_width  = 10;
  int border_height = 5;

  int continent_count = 2;
  int continent_region_width = (cfg.width - border_width * 2) / continent_count;
  int continent_region_height = cfg.height - border_height * 2;

  // generate continents
  for (int continent_id = 0; continent_id < continent_count; continent_id++) {
    int chunk_count  = 5;
    int chunk_height = continent_region_height / chunk_count;

    // generate continent seeds
    auto repeat = (
      cfg.width == 44 ? 1 :
      cfg.width == 74 ? 2 :
      cfg.width == 96 ? 3 :
      /* cfg.width == 106 ? */ 4
    );

    std::vector<hex::coords::offset> chunk_seeds;
    chunk_seeds.reserve(chunk_count * repeat);

    for (; repeat > 0; repeat--) {
      for (int chunk_id = 0; chunk_id < chunk_count; chunk_id++) {
        int col_start   = continent_id * (continent_region_width + border_width) + border_width / 2;
        int col_end     = col_start + continent_region_width;
        int col_padding = continent_region_height * 0.3;

        std::uniform_int_distribution<> rd_col(col_start + col_padding, col_end - col_padding);
        int col = rd_col(rd_gen);

        int row_start = border_height + chunk_height * chunk_id;
        int row_end   = row_start + chunk_height;
        int row       = (row_start + row_end) / 2;

        auto chunk_seed = hex::coords::offset(col, row);
        chunk_seeds.push_back(chunk_seed);
        ctx.landmass_map.at(chunk_seed) = true;
      }
    }

    // grow land from seeds
    auto growth_chance_dist = std::uniform_real_distribution<>(0.0, 1.0);
    auto max_growth_chance  = 0.5;
    int growth_cycles       = continent_region_width * 0.75;
    auto decay_factor       = 2.0;

    auto get_nearest_seed_distance = [&](const hex::coords::cubic &coord) {
      auto min_distance = std::numeric_limits<int>::max();

      for (const auto &seed : chunk_seeds) {
        auto distance = coord.distance(seed.oddr_to_cubic());
        min_distance = std::min(min_distance, distance);
      }

      return min_distance;
    };

    for (auto cycle = 0; cycle < growth_cycles; cycle++) {
      for (int row = 0; row < cfg.height; row++) {
        for (int col = 0; col < cfg.width; col++) {
          auto tile_coord = hex::coords::offset(col, row);
          auto tile_ccoord = tile_coord.oddr_to_cubic();

          for (const auto &neighbor_ccoord : tile_ccoord.neighbors()) {
            auto neighbor_coord = hex::coords::offset::oddr_from_cubic(neighbor_ccoord);

            if (
              ctx.landmass_map.contains(neighbor_coord)
              && !ctx.landmass_map.at(neighbor_coord).value()
            ) {
              auto distance = get_nearest_seed_distance(neighbor_ccoord);
              auto growth_chance = max_growth_chance * std::exp(-distance / decay_factor);

              if (growth_chance_dist(rd_gen) < growth_chance) {
                ctx.landmass_map.at(neighbor_coord) = true;
              }
            }
          }
        }
      }
    }
  }

  // eliminate lone land tiles
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      if (ctx.landmass_map.at(tile_coord).value()) {
        auto tile_ccoord = tile_coord.oddr_to_cubic();

        bool has_land = false;

        for (const auto &neighbor_ccoord : tile_ccoord.neighbors()) {
          auto neighbor_coord = hex::coords::offset::oddr_from_cubic(neighbor_ccoord);
          if (ctx.landmass_map.contains(neighbor_coord)) {
            has_land |= ctx.landmass_map.at(neighbor_coord).value();
          }
        }

        if (!has_land) {
          ctx.landmass_map.at(tile_coord) = false;
        }
      }
    }
  }
}
```

Producing the following result:

![](img/image-65c09ddcab2287ee.png)

Once I developed my debugging tool, I am able to see the full landmass map without zooming out (not the same map):

![](img/image-577209619d901dfa.png)

Because it will be useful for later, I want to know for each tile its distance to the closest water tile. I’m lazy, so I ask GPT4 again:

> Please implement a function which takes as parameters:  
>  — a reference to the `landmass_map` variable   
>  — a `coord` argument of type `hex::coords::offset`  
> And returns the distance to the closest water tile in the `landmass_map`.  
> The class `hex::coords::offset` is not hashable, but there is a template specialization of `std::hash` for `hex::coords::cubic`.

And what it came up with:

```cpp
#include <queue>
#include <unordered_set>
#include <utility>

int distance_to_closest_water(hex::storage<bool>& landmass_map, hex::coords::offset coord) {
  using hex::coords::cubic;
  using hex::coords::offset;

  // Initialize the BFS queue and visited set
  std::queue<std::pair<cubic, int>> bfs_queue;
  std::unordered_set<cubic, std::hash<cubic>> visited;

  // Convert the starting offset coordinate to cubic and add it to the queue with a distance of 0
  cubic start_cubic = coord.to_cubic();
  bfs_queue.push({start_cubic, 0});
  visited.insert(start_cubic);

  while (!bfs_queue.empty()) {
    // Get the next cubic coordinate and its distance from the queue
    auto [current_cubic, current_distance] = bfs_queue.front();
    bfs_queue.pop();

    // Convert the current cubic coordinate to offset and check if it's water
    offset current_offset = offset::from_cubic(current_cubic);
    if (!landmass_map.at(current_offset).value_or(false)) {
      return current_distance;
    }

    // Get the neighboring cubic coordinates and explore them
    for (const auto& neighbor_cubic : current_cubic.neighbors()) {
      // If the neighbor is within the bounds and not visited yet, add it to the queue
      offset neighbor_offset = offset::from_cubic(neighbor_cubic);
      if (landmass_map.contains(neighbor_offset) && visited.find(neighbor_cubic) == visited.end()) {
        bfs_queue.push({neighbor_cubic, current_distance + 1});
        visited.insert(neighbor_cubic);
      }
    }
  }

  // If the algorithm didn't find any water tiles, return -1 (should never happen if the map has water tiles)
  return -1;
}
```

A bit of refactoring later to loop over every tile in the map and put the code in a `task` as well, we get the following data:

![](img/image-7eeb4be22bfcce5b.png)

We see here a small limitation of my visualisation, hex tiles have 6 neighbors, while square tiles (the pixels in the image) only have 4\. But it’s ok, better than nothing 🙂

---

## Part 2: Rising the land

I want natural looking mountain chains. For this purpose, I decided to put mountains on the frontier of tectonic plates.

### Step 1: Tectonic plates

To generate the tectonic plates, I decided to reuse the Voronoi diagrams algorithm I used previously to assign biomes:

```cpp
void tectonic_task(const config &cfg, context &ctx) {
  int quadrant_x_divisor = 6;
  int quadrant_y_divisor = (cfg.height >= 60 ? 6 : 3);

  int quadrant_width  = cfg.width / quadrant_x_divisor;
  int quadrant_height = cfg.height / quadrant_y_divisor;

  std::default_random_engine gen_rd(cfg.seed);
  std::uniform_int_distribution<> rd_x(0, quadrant_width);
  std::uniform_int_distribution<> rd_y(0, quadrant_height);

  std::vector<hex::coords::cubic> seeds;
  seeds.reserve(quadrant_x_divisor * quadrant_y_divisor);

  for (int x = 0; x < quadrant_x_divisor; x++) {
    for (int y = 0; y < quadrant_y_divisor; y++) {
      auto seed_coord = hex::coords::offset(
        rd_x(gen_rd) + x * quadrant_width,
        rd_y(gen_rd) + y * quadrant_height
      );

      seeds.push_back(seed_coord.oddr_to_cubic());
    }
  }

  auto get_closest_seed_index = [&](hex::coords::cubic coord) {
    size_t closest_seed_index = 0;
    int closest_seed_distance = std::numeric_limits<int>::max();

    for (size_t seed_index = 0; seed_index < seeds.size(); seed_index++) {
      auto seed_coord = seeds[seed_index];
      auto distance   = seed_coord.distance(coord);

      if (distance < closest_seed_distance) {
        closest_seed_index    = seed_index;
        closest_seed_distance = distance;
      }
    }

    return closest_seed_index;
  };

  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord  = hex::coords::offset(col, row);
      auto tile_ccoord = tile_coord.oddr_to_cubic();

      ctx.tectonic_map.at(tile_coord) = get_closest_seed_index(tile_ccoord);
    }
  }
}
```

Which gives the following result:

![](img/image-817b515858ca1845.png)

### Step 2: Making a height map

Once again, I can reuse the code I already made in my previous attempt:

```cpp
void elevation_task(const config &cfg, context &ctx) {
  auto noise = tw::math::perlin_noise(cfg.seed);
  int noise_octaves = 1;
  float noise_persistance = 0.5;
  float noise_scale = 3;

  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      auto noise_x = col / (float) cfg.width;
      auto noise_y = row / (float) cfg.height;

      ctx.elevation_map.at(tile_coord) = noise.octave3D_01(
        tw::math::vec3(noise_x, noise_y, 0.34567) * noise_scale,
        noise_octaves,
        noise_persistance
      );
    }
  }
}
```

### Step 3: Rising mountains

As I said, I want to put mountains on the frontier of tectonic plates, but only on land tiles that are not too close to the water.

To find out if a tile is at the frontier of a tectonic plate, I just need to check that at least one neighbor tile is on a different tectonic plate. If it is, then I update the height map to set the corresponding tile’s height to the maximum:

```cpp
void mountains_task(const config &cfg, context &ctx) {
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord  = hex::coords::offset(col, row);
      auto tile_ccoord = tile_coord.oddr_to_cubic();

      auto is_land = ctx.landmass_map.at(tile_coord).value();
      if (is_land) {
        auto tectonic_plate = ctx.tectonic_map.at(tile_coord).value();
        bool is_frontier = false;

        for (const auto &neighbor_ccoord : tile_ccoord.neighbors()) {
          auto neighbor_coord = hex::coords::offset::oddr_from_cubic(neighbor_ccoord);
          if (
            ctx.tectonic_map.contains(neighbor_coord)
            && ctx.tectonic_map.at(neighbor_coord).value() != tectonic_plate
          ) {
            is_frontier = true;
            break;
          }
        }

        if (is_frontier) {
          auto water_proximity = ctx.water_proximity_map.at(tile_coord).value();

          if (water_proximity > 2) {
            ctx.elevation_map.at(tile_coord) = 1;
          }
        }
      }
    }
  }
}
```

This gives the following height map:

![](img/image-219e867305b51b65.png)

And the final result:

![](img/image-68ce924cf0cdb08a.png)

I can control the amount of mountains by control the number of tectonic plates. I could also add a few randomization on the tectonic plates map to make the frontiers less “straight”. Those tweaks are for later, for now I’m satisfied with the result 🙂

---

## Part 3: Make it rain

Here, I want to divide the map in 7 areas, from north to south:

- artic biome
- temperate biome
- tropical biome
- equatorial biome
- tropical biome
- temperate biome
- antarctic biome

In the equatorial biome, I will have either a “red desert” tileset or a “yellow desert” tileset. And I want to select which one based on humidity information.

I could generate a simple noise map for humidity, but I want to challenge myself a bit here 😅

One way to generate humidity maps is to consider the proximity to water, the elevation of the terrain, and the winds that can carry the humidity.

While I already have a heightmap and a water-proximity map, I don’t have a wind map.

To get a wind map, we will need a height map and a heat map as those are the 2 main factors (in our simplified world) that dictates the wind flow.

The heat map will be a Perlin Noise map, exactly like the height map, which is why I’ll skip its implementation here. But I’ll show the result in my debugging tool anyway:

![](img/image-562d00bb226634d0.png)

### Step 1: Wind of Change

For our purpose, the wind map will be a grid of 2D vectors. To generate this, we need to calculate 2 forces for each tile:

- the pressure gradient force: the force that results when there is a difference in pressure across a surface
- the Coriolis force: which causes moving objects to appear to curve as they travel across Earth’s rotating surface

Here is a visualization of the Coriolis force:

![](img/image-c8099786b3b555c9.png)

For our implementation, instead of using the difference in pressure to calculate the pressure gradient force, we will use the difference in elevation and temperature. In other words: wind will flow from high elevation to low elevation, and from hot areas to cold areas.

Again, we explain to GPT4 what we need, it comes up with the following implementation:

```cpp
void generate_wind_map(
  hex::storage<float>& elevation_map,
  hex::storage<float>& heat_map,
  hex::storage<vec2>& wind_map
) {
  using hex::coords::cubic;
  using hex::coords::offset;

  // Constants for tuning the pressure gradient, Coriolis forces, and temperature influence
  const float pressure_gradient_scale = 1.0f;
  const float coriolis_scale = 1.0f;
  const float temperature_scale = 1.0f;

  // Iterate through all tiles in the wind_map
  for (int y = 0; y < wind_map.height(); ++y) {
    for (int x = 0; x < wind_map.width(); ++x) {
      offset coord(x, y);

      vec2 pressure_gradient_force(0.0f, 0.0f);
      cubic current_cubic = coord.to_cubic();

      // Calculate the pressure gradient force based on the elevation and temperature differences
      for (const auto& neighbor_cubic : current_cubic.neighbors()) {
        offset neighbor_offset = offset::from_cubic(neighbor_cubic);

        if (wind_map.contains(neighbor_offset)) {
          float elevation_diff = elevation_map.at(coord).value_or(0.0f) - elevation_map.at(neighbor_offset).value_or(0.0f);
          float temperature_diff = heat_map.at(coord).value_or(0.0f) - heat_map.at(neighbor_offset).value_or(0.0f);
          float combined_diff = elevation_diff + temperature_scale * temperature_diff;

          vec2 direction(neighbor_offset.x - coord.x, neighbor_offset.y - coord.y);
          pressure_gradient_force += direction * combined_diff;
        }
      }

      pressure_gradient_force *= pressure_gradient_scale;

      // Calculate the Coriolis force based on the latitude (y-coordinate)
      vec2 coriolis_force(-coord.y * coriolis_scale, 0.0f);

      // Combine the pressure gradient and Coriolis forces to compute the wind vector
      vec2 wind_vector = pressure_gradient_force + coriolis_force;

      // Store the wind vector in the wind_map
      wind_map.at(coord) = wind_vector;
    }
  }
}
```

This is nice, but there is a subtle error in the implementation of the Coriolis force, let’s tell GPT4:

> In the `generate_wind_map` function, the y-lattitude of the coriolis force is supposed to become negative bellow the equator, but here, the `coord.y` variable only increases from 0 to the map height.

And then, it provides the following correction:

```cpp
// Calculate the relative latitude in the range [-1, 1]
float relative_latitude = 2.0f * (static_cast<float>(coord.y) / wind_map.height()) - 1.0f;

// Calculate the Coriolis force
vec2 coriolis_force(-relative_latitude, 0.0f);
```

After a bit of refactoring, variable renaming, we end up with the following result:

![](img/image-7819972b6771603a.png)

Shout out to `matplotlib` for this visualization!

Here, we can clearly see that the wind indeed flows away from high elevation points (the mountains), in the north the wind goes to the left, and in the south, the wind goes to the right. It’s in the opposite way of what the Wikipedia page about the Coriolis force describe, but it’s a simple minus sign that I can swap later. And who knows? Maybe our game’s world spin in the other direction anyway 😉

### Step 2: Singing in the rain

Now is the time to generate the humidity map. Here is the prompt for GPT4:

> I’d like to iterate multiple times over the tiles of the humidity map. Let’s do this in multiple steps:

> First, set the water tiles to a high humidity, and the land tiles to a low humidity. We can use the `water_proximity_map` to set a higher humidity level to the tile based on its distance to the water.

> Then, we will repeat the following `n` times, where `n` is the number of iteration we want to simulate the humidity begin caried by wind.

> For each tile, we should select a neighboring tile based on the direction of the wind vector for that tile. If the neighbor is in bounds, we should decrease the humidity level of the current tile, and increase the humidity level of the neighbor tile. The length of the wind vector should influence the amount of humidity we transfer.

> Assume I already have a function `get_neighbor` which takes a `hex::coords::cubic` parameter and a `vec2` parameter and returns a `hex::coords::cubic` corresponding to the neighboring tile closest to the wind vector argument.

To which it produced the following code:

```cpp
void generate_humidity_map(
  const hex::storage<bool>& landmass_map,
  hex::storage<int>& water_proximity_map,
  hex::storage<vec2>& wind_map,
  hex::storage<float>& humidity_map,
  int iterations
) {
  using hex::coords::cubic;
  using hex::coords::offset;

  // Initialize humidity_map with low values for land tiles and high values for water tiles
  for (int y = 0; y < humidity_map.height(); ++y) {
    for (int x = 0; x < humidity_map.width(); ++x) {
      offset coord(x, y);
      if (!landmass_map.at(coord).value_or(false)) {
        humidity_map.at(coord) = 1.0f;
      } else {
        int water_distance = water_proximity_map.at(coord).value_or(0);
        humidity_map.at(coord) = 0.1f + std::min(0.9f * (1.0f - static_cast<float>(water_distance) / 10.0f), 0.9f);
      }
    }
  }

  // Iterate n times
  for (int iteration = 0; iteration < iterations; ++iteration) {
    hex::storage<float> new_humidity_map = humidity_map;

    for (int y = 0; y < humidity_map.height(); ++y) {
      for (int x = 0; x < humidity_map.width(); ++x) {
        offset current_offset(x, y);
        cubic current_cubic = current_offset.to_cubic();
        vec2 wind_vector = wind_map.at(current_offset).value_or(vec2(0.0f, 0.0f));

        // Get the neighboring tile based on the wind vector
        cubic neighbor_cubic = get_neighbor(current_cubic, wind_vector);

        // Check if the neighbor tile is within bounds
        offset neighbor_offset = offset::from_cubic(neighbor_cubic);
        if (humidity_map.contains(neighbor_offset)) {
          // Transfer humidity from the current tile to the neighbor tile
          float transfer_amount = 0.1f * wind_vector.length(); // Adjust the factor (e.g., 0.1) as needed
          float current_humidity = humidity_map.at(current_offset).value_or(0.0f);
          float neighbor_humidity = new_humidity_map.at(neighbor_offset).value_or(0.0f);

          float new_current_humidity = std::max(current_humidity - transfer_amount, 0.0f);
          float new_neighbor_humidity = std::min(neighbor_humidity + transfer_amount, 1.0f);

          new_humidity_map.at(current_offset) = new_current_humidity;
          new_humidity_map.at(neighbor_offset) = new_neighbor_humidity;
        }
      }
    }

    humidity_map = new_humidity_map;
  }
}
```

Once again, I can identify a subtle problem in that snippet of code. Let’s tell GPT4:

> If 2 tiles’s wind vector point to the same neighbor tile, one will overwrite the other.

To which, GPT4 proposed the following correction:

```cpp
void generate_humidity_map(
  const hex::storage<bool>& landmass_map,
  hex::storage<int>& water_proximity_map,
  hex::storage<vec2>& wind_map,
  hex::storage<float>& humidity_map,
  int iterations
) {
  using hex::coords::cubic;
  using hex::coords::offset;

  // Initialize humidity_map with low values for land tiles and high values for water tiles
  for (int y = 0; y < humidity_map.height(); ++y) {
    for (int x = 0; x < humidity_map.width(); ++x) {
      offset coord(x, y);
      if (!landmass_map.at(coord).value_or(false)) {
        humidity_map.at(coord) = 1.0f;
      } else {
        int water_distance = water_proximity_map.at(coord).value_or(0);
        humidity_map.at(coord) = 0.1f + std::min(0.9f * (1.0f - static_cast<float>(water_distance) / 10.0f), 0.9f);
      }
    }
  }

  // Iterate n times
  for (int iteration = 0; iteration < iterations; ++iteration) {
    hex::storage<float> new_humidity_map = humidity_map;
    hex::storage<float> transferred_humidity(new_humidity_map.width(), new_humidity_map.height(), 0.0f);

    for (int y = 0; y < humidity_map.height(); ++y) {
      for (int x = 0; x < humidity_map.width(); ++x) {
        offset current_offset(x, y);
        cubic current_cubic = current_offset.to_cubic();
        vec2 wind_vector = wind_map.at(current_offset).value_or(vec2(0.0f, 0.0f));

        // Get the neighboring tile based on the wind vector
        cubic neighbor_cubic = get_neighbor(current_cubic, wind_vector);

        // Check if the neighbor tile is within bounds
        offset neighbor_offset = offset::from_cubic(neighbor_cubic);
        if (humidity_map.contains(neighbor_offset)) {
          // Transfer humidity from the current tile to the neighbor tile
          float transfer_amount = 0.1f * wind_vector.length(); // Adjust the factor (e.g., 0.1) as needed
          float current_humidity = humidity_map.at(current_offset).value_or(0.0f);

          float new_current_humidity = std::max(current_humidity - transfer_amount, 0.0f);
          new_humidity_map.at(current_offset) = new_current_humidity;

          transferred_humidity.at(neighbor_offset) = transferred_humidity.at(neighbor_offset).value_or(0.0f) + transfer_amount;
        }
      }
    }

    // Combine the transferred humidity with the new humidity map
    for (int y = 0; y < new_humidity_map.height(); ++y) {
      for (int x = 0; x < new_humidity_map.width(); ++x) {
        offset coord(x, y);
        float base_humidity = new_humidity_map.at(coord).value_or(0.0f);
        float transfer_humidity = transferred_humidity.at(coord).value_or(0.0f);
        new_humidity_map.at(coord) = std::min(base_humidity + transfer_humidity, 1.0f);
      }
    }

    humidity_map = new_humidity_map;
  }
}
```

Basically, we accumulate the amount of humidity we want to transfer and aggregate it at the end.

This is quite nice, though it requires a bit of refactoring, for example:

- I don’t like its use of the water proximity map to determine the base humidity level
- I prefer using the normalized length of the wind vector instead of this weird multiplication by `0.1f`
- I need to implement the `get_neighbor` function

So the final code looks like this:

```cpp
static hex::coords::cubic get_neighbor(hex::coords::cubic coord, tw::math::vec2 direction) {
  auto a = direction.angle();

  return (
    (a <   30 || a > 330) ? coord.neighbor<0>() :
    (a >=  30 && a <  90) ? coord.neighbor<1>() :
    (a >=  90 && a < 150) ? coord.neighbor<2>() :
    (a >= 150 && a < 210) ? coord.neighbor<3>() :
    (a >= 210 && a < 270) ? coord.neighbor<4>() :
    coord.neighbor<5>()
  );
}

void humidity_task(const config &cfg, context &ctx) {
  const int n_cycles = 10;

  // initialize humidity map, high for water tile, lower the farther away from water
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord = hex::coords::offset(col, row);
      auto water_dist = ctx.water_proximity_map.at(tile_coord).value();

      switch (water_dist) {
        case 0:
          ctx.humidity_map.at(tile_coord) = 1.0f;
          break;

        case 1:
          ctx.humidity_map.at(tile_coord) = 0.7f;
          break;

        case 2:
          ctx.humidity_map.at(tile_coord) = 0.4f;
          break;

        default:
          ctx.humidity_map.at(tile_coord) = 0.1f;
          break;
      }
    }
  }

  for (int cycle = 0; cycle < n_cycles; cycle++) {
    auto next_iteration_map = hex::storage<float>(cfg.width, cfg.height);
    auto transfer_map       = hex::storage<float>(cfg.width, cfg.height);

    for (int row = 0; row < cfg.height; row++) {
      for (int col = 0; col < cfg.width; col++) {
        auto tile_coord  = hex::coords::offset(col, row);
        auto tile_ccoord = tile_coord.oddr_to_cubic();

        auto wind_vector     = ctx.wind_map.at(tile_coord).value();
        auto neighbor_ccoord = get_neighbor(tile_ccoord, wind_vector);
        auto neighbor_coord  = hex::coords::offset::oddr_from_cubic(neighbor_ccoord);

        if (ctx.humidity_map.contains(neighbor_coord)) {
          auto current_humidity  = ctx.humidity_map.at(tile_coord).value();
          auto transfer_amount   = std::min(current_humidity, wind_vector.normalized_length());
          auto neighbor_humidity = transfer_map.at(neighbor_coord).value_or(0.0f);

          next_iteration_map.at(tile_coord) = current_humidity - transfer_amount;
          transfer_map.at(neighbor_coord)   = neighbor_humidity + transfer_amount;
        }
        else {
          next_iteration_map.at(tile_coord) = ctx.humidity_map.at(tile_coord);
        }
      }
    }

    for (int row = 0; row < cfg.height; row++) {
      for (int col = 0; col < cfg.width; col++) {
        auto tile_coord      = hex::coords::offset(col, row);
        auto base_humidity   = next_iteration_map.at(tile_coord).value();
        auto transfer_amount = transfer_map.at(tile_coord).value_or(0.0f);

        next_iteration_map.at(tile_coord) = std::min(base_humidity + transfer_amount, 1.0f);
      }
    }

    ctx.humidity_map = next_iteration_map;
  }
}
```

This will produce the following humidity map:

![](img/image-49c7b8a722441239.png)

Since the world does not wrap around, the edges of the map gets very dry even though they are ocean tiles, I’m not yet sure I care about this.

We can see parts of the land in the south gets a lot of humidity, and areas enclosed by mountains are very dry. Tweaking how the wind map is generated will impact this humidity map.

---

## Part 3: Assigning a biome to every tile

Once again, we will do this in multiple steps:

- we will generate a north pole and a south pole (cold biomes)
- we will then grow desert biomes from seeds taken in a dry land tile
- we will grow tropical biomes from seeds taken in a humid land tile
- the rest will be temperate biomes

### Step 1: Generating the poles

Since we want a north and south pole,

We generate a noise value using `std::sin` for the tile based on its x-coordinate.

Then, based on the noise value and the tile’s y-coordinate, we decide if the tile belong to the north/south pole. If it does we assign it to a cold biome, otherwise, we assign it to a temperate biome (for now).

The noise value allow us to have the biome frontier not be a straight line. The `std::sin` function is the easiest way of achieving this for a decent result.

The code is quite short and straightforward:

```cpp
void cold_biomes_task(const config &cfg, context &ctx) {
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord  = hex::coords::offset(col, row);
      auto latitude    = (float) row / (float) cfg.height;

      auto noise_value = (std::sin(col) + 1.0f) / 40.0f;

      if (
        latitude < (0.1f + noise_value)
        || latitude > (0.9f - noise_value)
      ) {
        ctx.biome_map.at(tile_coord) = biome_type::cold_land;
      }
    }
  }
}
```

Which gives this result:

![](img/image-926ef0f0c37b74cd.png)

### Step 2: Growing a desert biome and a tropical biome

This part is identical for desert biomes and tropical biomes, only the input parameters will change.

The idea here is to select randomly a land tile on each continent, for which the humidity level is within a certain range:

- `[0; 0.2]` for desert biomes
- `[0.5; 1]` for tropical biomes

Using GPT4 once again, it produces the following code that I refactored a bit:

```cpp
std::optional<hex::coords::offset> get_biome_seed(
  const config &cfg,
  const context &ctx,
  float humidity_threshold_low,
  float humidity_threshold_high,
  int start_col,
  int end_col
) {
  std::vector<hex::coords::offset> candidates;

  for (int row = 0; row < cfg.height; row++) {
    for (int col = start_col; col < end_col; col++) {
      auto tile_coord = hex::coords::offset(col, row);
      auto is_land    = ctx.landmass_map.at(tile_coord).value();
      auto water_dist = ctx.water_proximity_map.at(tile_coord).value();
      auto humidity   = ctx.humidity_map.at(tile_coord).value();
      auto has_biome  = ctx.biome_map.at(tile_coord).has_value();

      if (
        !has_biome
        && is_land
        && water_dist > 3
        && humidity >= humidity_threshold_low
        && humidity <= humidity_threshold_high
      ) {
        candidates.push_back(tile_coord);
      }
    }
  }

  if (candidates.empty()) {
    return std::nullopt;
  }

  std::default_random_engine rd_gen(cfg.seed);
  std::uniform_int_distribution<> dist(0, candidates.size() - 1);
  return candidates[dist(rd_gen)];
}
```

The parameters `start_col` and `end_col` are used to determine in which part of the world we should select our seed. I’ll use `(0; width/2)` for the first continent, and `(width/2; width)` for the second continent.

Since we don’t want our seed to be on the coast, we add a check using the water proximity map.

We also don’t want to select a seed that is already part of a biome, hence the check for `std::nullopt`. This is useful since we’ll want to repeat the process a few times to generate multiple biomes.

Next, we will use the `hex::coords::cubic` methods `ring()` and `spiral()`.

The `ring()` method will return the coordinates of the tiles that belong to a ring of a specific radius around the coordinate:

![](img/image-59d0c65f84cd02b4.png)

<https://www.redblobgames.com/grids/hexagons/>

Similarly, the `spiral()` method will return the coordinates for every tile within the ring:

![](img/image-06963de2819a8389.png)

<https://www.redblobgames.com/grids/hexagons/>

We will filter out the coordinates that are outside the bounds of the map, that already have a biome and that are land tiles:

```cpp
std::vector<hex::coords::offset> get_biome_area(
  const config &cfg,
  const context &ctx,
  hex::coords::offset seed_coord,
  int size
) {
  auto seed_ccoord = seed_coord.oddr_to_cubic();
  std::vector<hex::coords::offset> tiles;

  for (auto &inner_ccoord : seed_ccoord.spiral(size - 1)) {
    auto inner_coord = hex::coords::offset::oddr_from_cubic(inner_ccoord);

    if (ctx.landmass_map.contains(inner_coord)) {
      auto is_land   = ctx.landmass_map.at(inner_coord).value();
      auto has_biome = ctx.biome_map.at(inner_coord).has_value();

      if (!has_biome && is_land) {
        tiles.push_back(inner_coord);
      }
    }
  }

  std::default_random_engine rd_gen(cfg.seed);
  std::uniform_real_distribution<> dist(0, 1);

  for (auto &outer_ccoord : seed_ccoord.ring(size)) {
    auto outer_coord = hex::coords::offset::oddr_from_cubic(outer_ccoord);

    if (ctx.landmass_map.contains(outer_coord)) {
      auto is_land   = ctx.landmass_map.at(outer_coord).value();
      auto has_biome = ctx.biome_map.at(outer_coord).has_value();
      auto chance    = dist(rd_gen);

      if (!has_biome && is_land && chance > 0.3) {
        tiles.push_back(outer_coord);
      }
    }
  }

  return tiles;
}
```

We filter out some tiles from the outer ring randomly to give a more natural looking biome.

Finally, the code that assigns the biome types using those 2 functions:

```cpp
void desert_biomes_task(const config &cfg, context &ctx) {
  // generate "red_desert" for first continent
  auto red_desert_seed = get_biome_seed(
    cfg, ctx,
    0, 0.2f,
    0, cfg.width / 2
  );

  if (red_desert_seed.has_value()) {
    auto tiles = get_biome_area(cfg, ctx, red_desert_seed.value(), 4);

    for (auto tile_coord : tiles) {
      ctx.biome_map.at(tile_coord) = biome_type::red_desert;
    }
  }

  // generate "yellow_desert" for second continent
  auto yellow_desert_seed = get_biome_seed(
    cfg, ctx,
    0, 0.2f,
    cfg.width / 2, cfg.width
  );

  if (yellow_desert_seed.has_value()) {
    auto tiles = get_biome_area(cfg, ctx, yellow_desert_seed.value(), 4);

    for (auto tile_coord : tiles) {
      ctx.biome_map.at(tile_coord) = biome_type::yellow_desert;
    }
  }
}
```

The code is identical for tropical biomes, except for the humidity levels and the size of the biome area.

I can then add those tasks to the `process` class as many time as I want, to generate a biome on each continent, for the following result:

![](img/image-e2f52579f01828fe.png)

**NB:** I have a final task to set all the remaining tiles to the temperate biome:

```cpp
void temperate_biomes_task(const config &cfg, context &ctx) {
  for (int row = 0; row < cfg.height; row++) {
    for (int col = 0; col < cfg.width; col++) {
      auto tile_coord = hex::coords::offset(col, row);

      if (!ctx.biome_map.at(tile_coord).has_value()) {
        ctx.biome_map.at(tile_coord) = biome_type::temperate;
      }
    }
  }
}
```

---

## To be continued

This article is already insanely long, and the last part of the world generation algorithm will be probably as long. Let’s summarize a bit what we have done here:

- we generated continents using a cellular automaton to generate chunks of land
- we have built a water proximity map to be used in the following steps
- we generated tectonic plates using Voronoi diagrams
- we produced a height map and a heat map using Perlin Noise
- we have edited the height map to add mountains on the frontiers of tectonic plates
- we made a wind map using the height map and the heat map by simulating a Pressure Gradient Force and the Coriolis Force
- we generated a humidity map by simulating “humidity” being carried by “wind”
- we assign a biome type to every tile using simple rules (north/south for the cold biomes, random dry area for the desert, random wet area for the tropical biomes, temperate for the rest)

To assign a tile type, we will need to add a lot more code, each biome types have different features (swamps, wetlands, oasis, forests, …) that need to be carefully placed by using the height map, the humidity map and the heat map, and probably some cellular automata as well (to grow forests for example).

---

## Conclusion

The “cellular automa” and “simulation” approach gives way better results than my [previous attempt](/posts/2023-02-11-procedural-map-generation-in-c-part-1-the-slow-the-bad-and-the-ugly). I have way more control over the features I want to produce (though many constants need to be moved to the `config` structure).

It’s also way faster, it takes less than a second to generate the map in debug mode.

Regarding GPT4, it’s really a great tool. Instead of spending hours on Google, Reddit, StackOverflow, I can just ask GPT4 for a quick answer that I can verify. And you know how it’s easier to verify a solution than finding a solution.

While GPT4 is not a perfect tool (I always need to refactor the code it produces and I need to carefully write the prompt), it’s certainly a time-saver.

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 5 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for the game is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here → [Devlogs Reading List](/reading-lists/warmonger-dynasty)
