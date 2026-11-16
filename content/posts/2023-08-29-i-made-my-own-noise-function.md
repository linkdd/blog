---
title: "I made my own noise function"
date: "2023-08-29T22:34:39Z"
description: "Warmonger Dynasty Devlog #9 : Noise function for hexagonal maps"
tags: ["gamedev", "unity", "csharp", "devlog", "math"]
author: "David Delassus"
images: ["img/image-93fa7489290849dd.png"]
---

![](img/image-93fa7489290849dd.png)

---

## 📝 What is a “noise function”?

In Computer Graphics, a “noise function” refers to a mathematical function that produces a seemingly random, but deterministic, output.

We often use those functions to procedurally generate textures, terrains, and other visual elements.

The most commonly used noise functions might be Perlin noise and Simplex noise.

### **Why would you make your own noise function?**

Usually, noise functions take a cartesian 1D, 2D or 3D point as input, and returns a value between 0 and 1\. This means that the input space is a grid, usually the world space (2D or 3D) of your game, or a 1D “grid” along an axis.

In my game, Warmonger Dynasty, the grid is hexagonal, and I use a cubic coordinate system. It is a 2D coordinate system, where each point have 3 components: Q, R and S such that:

![](img/image-33a614e7c8ef3cc1.png)

For more information about this coordinate system, I invite you to read this article:

{{< cta-link
  url="https://www.redblobgames.com/grids/hexagons/"
  title="Red Blob Games: Hexagonal Grids"
  description="Amit's guide to math, algorithms, and code for hexagonal grids in games"
>}}

To understand why that makes a huge difference, let’s see how noise functions are usually implemented.

---

## 📉A simple 1D noise function

The first step is to generate random values at a regular interval. Here, we will generate 5 values at each integer input:

![](img/image-62f00fc9ef177f93.png)

```csharp
public class Noise1D {
  private float[] randomValues;

  Noise1D(int maxVertices) {
    var rndGen = new System.Random();
    randomValues = new float[maxVertices];

    for (int i = 0; i < maxVertices; i++) {
      randomValues[i] = (float)rndGen.NextDouble();
    }
  }

  // ...
}
```

Our noise function should return those values for the integer inputs:

```csharp
public class Noise1D {
  // ...

  public float Get(float x) {
    var ix = (int) x;
    var qx = x - ix;

    if (qx == 0f) {
      var index = ix % randomValues.Length;
      return randomValues[index];
    }
    else {
      // ...
    }
  }
}
```

Here, we cast the input value to an integer and get the fractional part to check if the input was an integer. That integer is then used to get an index in our `randomValues` table.

**NB:** Our function is not defined for negative values of `x`, that is an exercise left to the reader.

However, if the input value is not an integer, then we will perform a linear interpolation of the noise value at that point, based on the noise values of the bounding integers:

```csharp
public class Noise1D {
  // ...

  public float Get(float x) {
    var ix = (int) x;
    var qx = x - ix;

    if (qx == 0f) {
      // ...
    }
    else {
      var lowerBound = Get(ix);
      var upperBound = Get(ix + 1f);
      return Lerp(lowerBound, upperBound, qx);
    }
  }

  private float Lerp(float a, float b, float t) {
    return a * (1f - 1) + b * t;
  }
}
```

And voilà, our noise function:

![](img/image-e6907e5a80e64e66.png)

Instead of giving `qx` directly to our `Lerp()` method, we can use a smooth function, like `cos()` for example:

```cpp
public class Noise1D {
  // ...

  public float Get(float x) {
    // ...
    return Lerp(lowerBound, upperBound, Smooth(qx));
    // ...
  }

  private float Smooth(float t) {
    // 0 <= t <= 1
    return (1f - (float)System.Math.Cos(t * System.Math.PI)) / 2f;
  }
}
```

This should round the noise values around the integer inputs.

---

## 📈A 2D noise function

Instead of generating random values at regular intervals along an axis, we dot it on a 2D grid:

![](img/image-2f49993119913ef4.png)

Given an `x;y` input, we identify the 4 bounding points in our grid, lets name them:

- `c00`: top left
- `c01`: top right
- `c10`: bottom left
- `c11`: bottom right

We will interpolate a value `nx0` on the `c00;c01` axis, and a value `nx1` on the `c10;c11` axis:

```csharp
var nx0 = Lerp(c00_val, c01_val, Smooth(x - c00_val));
var nx1 = Lerp(c10_val, c11_val, Smooth(x - c10_val));
```

This gives us a new vertical axis `nx0;nx1`. Our final noise value will be the interpolation of `y` on that axis:

```csharp
return Lerp(nx0, nx1, Smooth(y - nx0));
```

This concept can then be extended to any number of dimensions you might need.

For more information on this topic, I recommend the following resource:

{{< cta-link
  url="https://www.scratchapixel.com/lessons/procedural-generation-virtual-worlds/procedural-patterns-noise-part-1/introduction.html"
  title="Value Noise and Procedural Patterns: Part 1"
  description="Reading time: 17 mins. To the physicist, unpredictable changes of any quantity V varying in time t are known as noise…"
>}}

This is what I used to learn more about noise and how to ultimately make my own noise function 🙂

---

## The problem with hexagons

Our 2D noise function uses internally a grid in the cartesian coordinate system (`x;y` coordinates) and takes as input a cartesian coordinate as well.

But our map uses the cubic coordinate system (`Q;R;S` coordinates).

Instead of a cartesian grid, we could use a hexagonal map, and for each `x;y` coordinates, we would interpolate the noise value from the 6 vertices of the hexagon bounding our input point. But this would still be noise in a cartesian space.

What we want is to assign a noise value to each hexagon, not each pixel contained in an hexagon.

---

## Coming up with a solution

What we will do is quite simple. To each `Q;R;S` coordinate, we will assign a random gradient (a `Vector3`). The noise value will be calculated based on the gradient of the input coordinate and the gradients of its neighbors.

```csharp
public class QRNoise {
  private System.Random rndGen;
  // ...

  public QRNoise(int seed) {
    rndGen = new System.Random(seed);
    // ...
  }

  public float Get(Hex.Cubic coord) {
    var total = GetGradientContribution(coord, Hex.Cubic.zero);
    foreach (var neighbor in coord.Neighbors()) {
      total += GetGradientContribution(neighbor, coord - neighbor);
    }
    var npoints = 7;
    return (total / npoints + 1f) / 2f; // remap to [0;1] range
  }

  private float GetGradientContribution(Hex.Cubic coord, Hex.Cubic offset) {
    var gradient = GetGradient(coord);
    // dot product:
    return gradient.x * offset.q + gradient.y * offset.r + gradient.z * offset.s;
  }

  private Vector3 GetGradient(Hex.Cubic coord) {
    // ...
  }

  // ...
}
```

We need to assign a random gradient to every possible coordinate. We will do this using a hash table, similar to how Perlin Noise does it.

First, we need to generate a bunch of random gradients:

```csharp
public class QRNoise {
  // ...
  private Vector3[] gradients;

  // ...
  private const int GRADIENTS_SIZE = 32;
  // totally arbitrary value, bigger means more randomness

  public QRNoise(int seed) {
    // ...

    InitializeGradients();
  }

  // ...

  private void InitializeGradients() {
    gradients = new Vector3[GRADIENTS_SIZE];

    for (var i = 0; i < GRADIENTS_SIZE; i++) {
      var gradient = new Vector3(
        (float)rndGen.NextDouble() * 2.0f - 1.0f,
        (float)rndGen.NextDouble() * 2.0f - 1.0f,
        (float)rndGen.NextDouble() * 2.0f - 1.0f
      );
      gradients[i] = gradient.normalized; // we want a magnitude of 1
    }
  }

  private Vector3 GetGradient(Hex.Cubic coord) {
    var h = GetHash(coord) % GRADIENTS_SIZE;
    return gradients[h];
  }

  private int GetHash(Hex.Cubic coord) {
    // ...
  }
}
```

The `GetHash()` function will return consistently an integer for each `Q;R;S` coordinate that we can use to get a gradient.

To implement this function, we will copy what PerlinNoise does. We create an array of integers. We then use the components of our coordinate to get another integer from that array. The final integer will be our hash:

```csharp
using System.Linq;

public class QRNoise {
  // ...
  private int[] permutations;

  private const int PERMUTATIONS_SIZE = 256;
  private const int HASH_PRIME = 521;
  // first prime number after PERMUTATIONS_SIZE * 2

  // ...

  public QRNoise(int seed) {
    // ...
    InitializePermutations();
    // ...
  }

  // ...

  private void InitializePermutations() {
    permutations = Enumerable
      .Range(0, PERMUTATIONS_SIZE)
      .ToList()
      .Shuffle(rndGen) // this is an extension method I made
      .ToArray();
  }

  // ...

  private int GetHash(Hex.Cubic coord) {
    // let's make sure we have positive indices
    // and use a prime number to avoid collisions and distribute values uniformly
    // NB: i'm not sure of the why, let's trust my Google search for now :)
    var q = Mathf.Abs(coord.q) % HASH_PRIME;
    var r = Mathf.Abs(coord.r) % HASH_PRIME;
    var s = Mathf.Abs(coord.s) % HASH_PRIME;

    var a = permutations[s % PERMUTATIONS_SIZE];
    var b = permutations[(r + a) % PERMUTATIONS_SIZE];
    var c = permutations[(q + b) % PERMUTATIONS_SIZE];
    var d = permutations[c % PERMUTATIONS_SIZE];

    return d;
  }
}
```

Once this is done, I am able to generate a noise map for my hexagonal map quite easily:

![](img/image-f1b026e6111df65e.png)

Tweaking the interpolation function will lead to different results, but so far, I’m quite satisfied 🙂

---

Here is the complete code:

```csharp
using System.Linq;
using UnityEngine;

public class QRNoise {
  private System.Random rndGen;
  private int[] permutations;
  private Vector3[] gradients;

  private const int PERMUTATIONS_SIZE = 256;
  private const int HASH_PRIME = 521;
  private const int GRADIENTS_SIZE = 32;

  public QRNoise(int seed) {
    rndGen = new System.Random(seed);

    InitializePermutations();
    InitializeGradients();
  }

  private void InitializePermutations() {
    permutations = Enumerable
      .Range(0, PERMUTATIONS_SIZE)
      .ToList()
      .Shuffle(rndGen)
      .ToArray();
  }

  private void InitializeGradients() {
    gradients = new Vector3[GRADIENTS_SIZE];

    for (var i = 0; i < GRADIENTS_SIZE; i++) {
      var gradient = new Vector3(
        (float)rndGen.NextDouble() * 2.0f - 1.0f,
        (float)rndGen.NextDouble() * 2.0f - 1.0f,
        (float)rndGen.NextDouble() * 2.0f - 1.0f
      );

      gradients[i] = gradient.normalized;
    }
  }

  public float Get(Hex.Cubic coord) {
    var total = GetGradientContribution(coord, Hex.Cubic.zero);

    foreach (var neighbor in coord.Neighbors()) {
      total += GetGradientContribution(neighbor, coord - neighbor);
    }

    var npoints = 7;
    return (total / npoints + 1f) / 2f;
  }

  private float GetGradientContribution(Hex.Cubic coord, Hex.Cubic offset) {
    var gradient = GetGradient(coord);
    return gradient.x * offset.q + gradient.y * offset.r + gradient.z * offset.s;
  }

  private Vector3 GetGradient(Hex.Cubic coord) {
    var h = GetHash(coord) % GRADIENTS_SIZE;
    return gradients[h];
  }

  private int GetHash(Hex.Cubic coord) {
    var q = Mathf.Abs(coord.q) % HASH_PRIME;
    var r = Mathf.Abs(coord.r) % HASH_PRIME;
    var s = Mathf.Abs(coord.s) % HASH_PRIME;

    var a = permutations[s % PERMUTATIONS_SIZE];
    var b = permutations[(r + a) % PERMUTATIONS_SIZE];
    var c = permutations[(q + b) % PERMUTATIONS_SIZE];
    var d = permutations[c % PERMUTATIONS_SIZE];

    return d;
  }
}
```

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 8 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for my game **Warmonger Dynasty** is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here →

{{< cta-link
  url="/reading-lists/warmonger-dynasty"
  title="Warmonger Dynasty Devlogs"
  description=""
>}}
