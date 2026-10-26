# Square Marching Mining Game

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/361164f6-961d-45fb-b76c-511480ba3247" />

A 2D submarine mining game built in **C++ with Raylib**. The terrain is rendered entirely on the GPU using a custom **Square Marching** fragment shader. Explore cavern systems, mine for ores, and fight off creatures that get nastier the deeper you go.

> **Status:** Early development. Core terrain, player movement, mining, and inventory are working. Enemy AI and most gameplay systems are still WIP.

## Features

### Terrain
- **Square Marching on the GPU:** the terrain surface is computed entirely in a custom GLSL fragment shader. No CPU geometry, smooth cave walls, and mining is just a texture update.
- **Chunked world:** the map is split into chunks and only visible ones are rendered each frame.
- **Ore rendering:** ores are baked into the terrain texture and rendered in the same shader pass, with randomized shape offsets and bilinear color blending.
- **Real-time mining:** modifying terrain just patches the chunk texture, which the shader picks up immediately.
- **Collision detection:** player collides with the terrain surface per-chunk using a signed-distance approach.
- **Raycasting:** uses the Amanatides-Woo fast voxel traversal for chunk-aware ray queries.
- **Map loading:** terrain is loaded from PNG files where pixel channels encode density, ore type, and hardness.
- **Adjustable surface level and interpolation:** tweak how smooth or blocky the terrain looks at runtime.

### Player
- **Second-order dynamics movement:** smooth, springy motion without a physics engine. Frequency, damping, and speed are all configurable.
- **Inventory:** slots can be unlocked, items added/removed, active tool switched.
- **Tools:** modular item system for equippable tools:
  - `MiningTool` - mines in a configurable radius with adjustable power and fire rate
  - `BallSpawnerTool` - prototype/debug tool that spawns physics balls
- **Ore tanks:** inventory items that hold collected ore up to a set capacity.

### Rendering
- **Render pipeline:** rendering goes through a list of `RenderPass` stages, each getting the previous pass output as a texture. Easy to add post-processing or reorder layers.
- **Post-processing:** full-screen shader passes for things like the terrain outline effect.
- **Asset manager:** shaders and textures are cached by path so nothing gets uploaded to the GPU twice.
- **Camera:** tracks the player and exposes a world-space screen rect used for frustum culling.

### Engine
- **Component/GameObject system:** entities are `GameObject`s with attached `Component`s. Components get `Awake()`, `Update()`, and cleanup callbacks.
- **`GameManager` singleton:** owns the window, game loop, and render pipeline.
- **Typed event system:** `ActionEvent<Args...>` is a templated observer that lets systems talk to each other without direct dependencies.
- **Input manager:** centralizes Raylib input into a per-frame snapshot.

### UI
- **Inventory HUD:** item slots with icons, rendered with a custom shader.
- **Ore meter:** fill bar per ore tank item, also shader-driven.


## How it works

### The terrain texture format
Every terrain chunk is backed by a single RGBA32F texture on the GPU, where each texel corresponds to one terrain point. The four channels pack all the data the shader needs:

| Channel | Field | Description |
|---|---|---|
| R | `surfaceValue` | Terrain density (0 = empty, 1 = fully solid). The surface is drawn where this crosses the `surfaceLevel` threshold (~0.5). |
| G | `oreValue` | How much ore is left at this point (0 = depleted/none). Controls how much of the ore shape bleeds through. |
| B | `oreType` | Which ore type is here, encoded as a discrete float (0.1 steps). Used to index into the ore color and shape atlases. |
| A | `hardness` | How resistant this point is to mining. Higher hardness means slower digging. Also used to tint the rock color slightly. |

This maps exactly to the `TerrainPoint` struct on the CPU side:
```cpp
struct TerrainPoint {
    float surfaceValue, oreValue, oreType, hardness;
};
```

When you load a map PNG, each pixel's RGBA bytes (0-255) get normalized to 0-1 floats and packed into a `TerrainPoint`. The chunk uploads the whole array as a `PIXELFORMAT_UNCOMPRESSED_R32G32B32A32` texture, which the shader reads with point filtering so there's no hardware interpolation blurring the values.

### Square Marching on the GPU
The fragment shader runs once per pixel of each visible chunk. For every pixel it figures out which 2x2 grid cell it falls into and samples the `R` channel (surface density) at all four corners. Those four booleans (is each corner solid?) produce a 4-bit index into a 16-entry lookup table that lists which cell edges the terrain boundary crosses.

The shader then computes where exactly on each edge the surface level threshold is crossed, draws a line between those two edge points, and checks which side of that line the current pixel is on. If it's on the solid side it gets the foreground (rock) color, otherwise the background (cave wall) color.

The surface position on each edge can be placed at the exact interpolated crossing point (`interpolationAmount = 1.0`) or snapped to the midpoint of the edge (`interpolationAmount = 0.0`), giving either smooth organic walls or a blocky pixel look.

### Ore rendering in the shader
Ore is also drawn entirely in the same shader pass. If a texel has `oreValue > 0` and a non-zero `oreType`, the shader samples a shape atlas texture using the ore type to pick a row and a hash of the texel's position to add a bit of random offset. If that shape value is high enough relative to the remaining `oreValue`, the pixel is colored with the ore's color instead of rock. This means ore visually "fills" the terrain and shrinks as you mine it, without any separate geometry.

### Mining
When you mine, the CPU walks all terrain points within the radius, reduces their `surfaceValue` (or `oreValue` if ore is present) based on mining power, falloff, and the point's `hardness`, then calls `UpdateTexture()` to push the modified `TerrainPoint` array back to the GPU. The shader picks up the change on the next frame automatically.

### Raycasting
Raycasts use Amanatides-Woo DDA grid traversal to hop between chunks efficiently. Cost scales with how many chunks the ray crosses, not the total terrain size.

### Movement
`V2SecondOrderDynamics` is a second-order spring-damper that gives the player smooth, physically convincing movement with no physics engine. Tune it with a frequency, damping ratio, and initial response coefficient.

### Render pipeline
`RenderPipeline` holds an ordered list of `RenderPass` objects. Each pass renders into a `RenderTexture2D` and hands the result to the next one. Terrain, entities, post-processing, and UI each live in their own pass.


## Roadmap

| System | Status | Notes |
|---|---|---|
| Player game feel | WIP | Animations, screen shake, mining feedback |
| Ore UI | WIP | Ore meter polish, multiple ore types |
| Ore system | WIP | More ore types, depth-based distribution |
| Store and Upgrades | Planned | Place to sell ores and buy upgrades |
| Enemy AI | Planned | Predator/prey behaviour, chunk-based activation |
| Enemy types | Planned | More variety, scaling danger with depth |
| Points of interest | Planned | Ruins, caches, unique cavern rooms |
| Oxygen + respawn | Planned | Limited oxygen, death leaves a lootable wreck |
| Mining radius shader | Planned | Visual indicator for mining range via post-processing |
| Story / goal | Idea | Some kind of ending or objective |
| Procedural map gen | Idea | Generate the PNG maps instead of hand-authoring them |
| Web build | Idea | Emscripten scaffolding is already in CMakeLists |

## Building

### Requirements

| | |
|---|---|
| CMake >= 3.30 | |
| C++23 compiler | MSVC on Windows, GCC or Clang on Linux/macOS |
| Raylib 5.5 | Auto-fetched via `FetchContent` if not already installed |
| OpenGL 4.3+ | The terrain shader uses `#version 430` |

> macOS: OpenGL is deprecated on Apple platforms and GLSL 4.30 doesn't run on Apple Silicon without a port.

### Clone and build

```bash
git clone https://github.com/Dalv-Dalv/SquareMarchingMiningGame
cd SquareMarchingMiningGame

cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release   # Windows
cmake --build build -j$(nproc)         # Linux / macOS
```

Assets (`Shaders/`, `Textures/`, `Sounds/`, `GeneratedMaps/`) are copied next to the binary automatically.

### Run

```bash
./build/bin/ProiectPOO       # Linux / macOS
build\bin\ProiectPOO.exe     # Windows
```

---

## Configuration

Edit [`config.h`](config.h) before building:

```cpp
#define SCREEN_WIDTH  1920
#define SCREEN_HEIGHT 1080
#define USE_FULLSCREEN true
#define USE_VSYNC true

#define TERRAIN_INTERPOLATION_AMOUNT 0.0  // 0 = hard edges, 1 = fully interpolated
```

---

## Controls

> These will change as the input system gets fleshed out.

| Action | Input |
|---|---|
| Move | WASD / Arrow Keys |
| Mine / Use tool | Left mouse (hold) |
| Switch tool | Scroll wheel or 1-5 |
