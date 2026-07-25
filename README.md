# PocketPal

A 3D virtual pet running on a $5 microcontroller. Software-rendered. No GPU.

Every vertex transform, triangle rasterization, and pixel write is software. No 3D accelerator. No floating-point SIMD. The renderer is a dual-core producer/consumer pipeline written specifically for the ESP32-S3.

[Full demo video](assets/demo.mp4)

## Hardware

| Component | Spec |
|-----------|------|
| MCU | ESP32-S3 N16R8 |
| CPU | Xtensa LX7 dual-core @ 240 MHz |
| SRAM | 512 KB (320 KB usable internal) |
| PSRAM | 8 MB (Octal, 120 MB/s theoretical) |
| Flash | 16 MB |
| Display | ST7789 SPI LCD, 240x135, RGB565 framebuffer |

## Architecture

Core 1 runs the main loop and game logic. Core 0 runs the rasterizer. They talk through a draw list and a semaphore. That's it.

- **Core 1**: simulation, animation, visibility tests, draw list generation.
- **Core 0**: rasterization, DMA display output.

The render task is pinned to Core 0; the main loop runs on Core 1. In theory they work in parallel. In practice Core 1 submits the draw list and then waits. The real win was reducing Core 1's per-frame work, not load-balancing the two cores. I spent a week trying to make them "balanced" before realizing that was the wrong goal.

## Rendering Pipeline

Each model goes through these stages:

- Transform (fused float)
- Near plane reject
- Near plane clip (homogeneous, Sutherland-Hodgman)
- Perspective projection
- Screen guard band clip
- Backface culling (64-bit shoelace)
- Triangle fan
- Rasterization (integer scanline)

Triangles behind the near plane get rejected before the clipper. Only triangles that actually intersect the near plane run the full Sutherland-Hodgman clip. This sounds obvious but I didn't do it initially, and the profiler made me feel stupid for that.

## Mathematics

Rendering uses a mix of fixed point and floating point, chosen per stage. Not because I planned it this way—because I tried pure fixed-point, it was worse, and I backtracked.

- Transform matrices — Q16.16 fixed point, stored as a `Mat4`.
- Vertex transform + projection — a single fused fixed-to-float pass per vertex. The MVP converts to float once, then each vertex is multiplied and perspective-divided. I tried keeping this in fixed-point. The ESP32-S3 FPU beat it by a wide margin. I don't fully understand why; the FPU is supposed to be slow. It isn't.
- Clip-space transforms / homogeneous clipping — float. For correctness and because I got tired of debugging fixed-point overflow.
- Rasterization — integer scan conversion, perspective interpolation via reciprocal lookup tables. Real divide fallback for spans > 240. The lookup table was a 3-hour detour that saved maybe 2ms per frame. Worth it? Probably not. But it exists now.

## Optimizations

- Unified pipeline — one `drawModel()` path replaced three duplicated renderers (flat / textured / shadow), selected by flags. I had three renderers because I kept adding features without refactoring. It was a mess.
- Integer rasterizer — the entire scan-conversion pipeline uses integer arithmetic. This part actually worked on the first try, which surprised me.
- Reciprocal lookup tables — perspective interpolation avoids hardware division. With a real-divide fallback for spans > 240. The fallback almost never triggers. I should probably remove it.
- Near-plane early reject — triangles fully behind the near plane bypass clipping. Clip calls dropped from tens of thousands to a few hundred. The profiler showed this clearly. I ignored the profiler for two days because I was convinced the rasterizer was slow. It wasn't.
- Improved backface culling — 64-bit shoelace area avoids overflow from large clipped-screen coordinates. The old version overflowed silently and produced garbage triangles. I thought it was a clipping bug. It wasn't.
- Vertex batch transforms — vertices transformed once per model, not once per triangle. Obvious in hindsight. Not obvious when I wrote the first version.
- Guard-band clipping — screen clipping happens after projection. I added this because triangles near the screen edge produced wild coordinates. The guard band is 2x screen size. I don't know if 2x is optimal. It works.
- Sub-mesh frustum culling — whole mesh sections rejected before entering the renderer. The room has a lot of submeshes. This helped.
- Dual-core command pipeline — rendering overlaps simulation. In theory. In practice the overlap is minimal because Core 1 finishes its work quickly and waits. Still, the separation is clean.
- Scratch arena allocator — a 512 KB internal-SRAM arena recycles per-frame vertex transform storage. Replaced earlier independent clip-slot/batch heap allocations that were fragmenting. I didn't notice the fragmentation until I added the 4th pet and the allocator failed. Then I noticed.
- Blended 1D upscaler — half-width mode (120x135) uses a 1D horizontal blended upscaler instead of nearest-neighbor. Each pixel is blended with its right neighbor to reduce aliasing.

## Shadows

Shadows are projected geometry (the pet silhouette projected onto the floor plane), sharing the same pipeline as opaque rendering. No Z-testing, no Z-writing — painted directly into the framebuffer so they overlay the floor and props but get overwritten by the pet itself. Baked per-triangle spotlight shading means static lighting costs nothing per frame. I added this because the scene looked flat without it. It was a 2-hour hack that stayed.

## Memory Management

| Buffer | Location |
|--------|----------|
| Framebuffer | Internal SRAM (DMA-capable, 32-byte aligned) |
| Z-buffer | 16-bit depth, internal SRAM with PSRAM fallback |
| Transformed vertices | Per-frame scratch arena, internal SRAM |
| Texture atlas | Internal SRAM copy when possible; falls back to original PSRAM pointer |
| Model vertices, UVs, triangles | LittleFS into PSRAM; copies attempted in internal SRAM when possible |

Models load from LittleFS into PSRAM. Animated models allocate a per-frame posed-vertex buffer in internal SRAM with PSRAM fallback. The 512 KB scratch arena recycles transform storage each frame. Earlier attempts to place framebuffers in PSRAM caused stuttering from SPI contention. I thought PSRAM was "free" memory because 8 MB >> 512 KB. It isn't free. It cost me a day of debugging.

## Performance

Reported figures from the current full-3D scene:

| Metric | Value |
|--------|-------|
| Source triangles | ~5,000 (varies by pet and room complexity) |
| Drawn per frame | ~2,700 |
| Culled (backface + bounds) | ~2,300 |
| Clipped (near plane) | 0–30 |
| FPS | 32–34 (full resolution) / 40 (half-width mode) |
| Display resolution | 240x135 |

The framebuffer copy and DMA push (Core 1) overlap with background-fill, Z-clear and rasterization (Core 0). The dominant per-frame cost is vertex transform. Room geometry is static in world space but moves in screen space as the camera follows the pet. Pre-transforming room vertices to world space at load would cut room transform cost by ~80%. I know exactly how to do this. I haven't, because 34 FPS is playable and I got tired of optimizing.

A half-width framebuffer mode (120x135) upscales to the display via a 1D horizontal blended upscaler. This trades sharpness for higher frame rate. I added it because I wanted to see 40 FPS on the profiler. I use the full-resolution mode.

## Performance by Pet

| Pet | Triangles | FPS | Notes |
|-----|-----------|-----|-------|
| Turtle | 3,218 | 54–56 | Simple room, static pose, shadows |
| Penguin | 3,633 | 56 | Simple room, static pose, shadows |
| Cat | ~3,600 | 40–45 | Simple room, animated, shadows |
| Dog | — | — | In progress |
| Bunny | 6,770 | 30 | Heavier room geometry, shadows |

The renderer scales with scene complexity. Lighter pets hit 50+ FPS. Full room with animated pets holds 30+ FPS. All pets include dynamic shadows.

## Profiling

The renderer contains a built-in hierarchical profiler using the Xtensa CCOUNT cycle counter. Each core has its own instance (no locking). Records total execution time, call count, and percentage of frame.

Typical regions:

- `core0_frame` / `core1_frame` — whole frame per core
- `core1_commands` — draw-list consumption on Core 0
- `draw_pet3d` — Core 1 building and submitting the 3D draw list
- `tama_update` / `tama_draw` — simulation/animation and 2D overlay on Core 1
- `display_wait` — Core 1 waiting for DMA push to finish
- `drawModel` — per-model draw (transform + clip + raster)
- `xform` — transformProjectMesh vertex pass
- `clipNearPlane` — near-plane homogeneous clip
- `clipScreenGBInt` — guard-band screen clip

Additional diagnostics report triangle counts, clipping stats, culling stats, raster time, and frame sync. This profiler was instrumental in finding the real bottleneck. I spent two days convinced the rasterizer was slow. The profiler said vertex transform was 237 ms. I didn't believe it. I ran the profiler three more times. Then I fixed it.

## Model Format (EMDL)

Custom binary format, little-endian:

- Indexed geometry
- Texture coordinates
- Embedded texture atlases (RGB565)
- Sub-meshes (per-mesh frustum culling)
- Bounding spheres / AABBs
- Precomputed metadata (face normals, baked shade)
- Keyframe vertex animation (v3)
- Skeletal bone animation — joints and weights (v4)

Vertex positions are int16_t scaled by 1/1024 into world units. Models load from LittleFS into PSRAM; animated models allocate a per-frame posed-vertex buffer. The format grew organically as I added features. It's not elegant. It works.

## Current State

The renderer supports:

- Fully textured models
- Animated characters (keyframe + skeletal)
- Dynamic shadows (projected geometry)
- Baked per-triangle spotlight lighting
- Z-buffering
- Perspective-correct texturing
- Dual-core rendering
- Frustum culling
- Near-plane clipping
- Backface culling
- Guard-band clipping
- Command buffering
- Native desktop testing
- ESP32 deployment

All software. All on an ESP32-S3.

## Pets

| Pet | Status | Animation |
|-----|--------|-----------|
| Cat | Done | Skeletal (idle, walk, eat, sleep, clean) |
| Dog | In progress | — |
| Bunny | Done | Static |
| Penguin | Done | Static |
| Turtle | Done | Static |

All pet portraits are visible in the selection menu. The Cat and Dog share the same rigged model with skeletal animation; other pets display in static poses.

## Future Work

- Arena-based scratch allocator (done; further tuning maybe)
- Static geometry caching (I know how to do this; I haven't)
- Additional draw-call batching
- Better spatial partitioning
- Occlusion culling (probably overkill for this scene)
- Further rasterizer optimization
- Material system
- Multiple dynamic lights
- Visibility structures (BSP / portals) for indoor scenes
- More pets
- More rooms
- More interactions

Some of these will happen. Most won't. The project is shipped.

## Lessons

- Measure first. I spent two days convinced the rasterizer was slow. It wasn't. The profiler said vertex transform was 237 ms. I didn't believe it. I ran the profiler three more times. Then I fixed it.
- SIMD is not always SIMD. The ESP32-S3 has vector instructions. Great! Except they're 8/16-bit integer only. I spent a morning reading the ISA manual before accepting I couldn't use them. That morning saved me a week of wasted work.
- Memory hierarchy matters more than clock speed. PSRAM is 3-4x slower than SRAM for random access. I put the framebuffer in PSRAM because 8 MB >> 512 KB. The stuttering was immediate and confusing. Moving it back to SRAM fixed it. I still don't fully understand the bus arbitration.
- Shipping is a feature. 60 FPS is achievable. 34 FPS is shipped. I keep telling myself I'll go back and optimize the room geometry cache. I probably won't.
- Dual-core is easy to describe, hard to balance. Core 1 spends most of the frame waiting. The win was reducing Core 0's work, not balancing the cores. I spent a week trying to make them "even" before realizing that was the wrong goal.
- Use the FPU. Float projection on the ESP32-S3 beat fixed-point software division by a wide margin. I assumed fixed-point would be faster because it's "embedded." It wasn't. The FPU is actually good.
- Animation pipelines are fragile. I exported 5 animation clips from Blender when I only wanted 1. The runtime loaded the wrong clip. The cat looked like it exploded. I spent an afternoon debugging skinning matrices before realizing the loader was defaulting to clip index 0 instead of clip index 1. The fix was one line.

## Media

| Media | File | Description |
|-------|------|-------------|
| Demo video | assets/demo.mp4 | Live device running the virtual pet |
| Hardware photo | assets/hardware.jpg | ESP32-S3 board with the 240x135 ST7789 display |wn |
| Pet portraits | assets/portraits/ | All pet selection portraits |

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | C++ |
| Build System | PlatformIO |
| Framework | Arduino-ESP32 |
| Display Driver | Custom ST7789 (80 MHz Octal SPI, DMA-backed) |
| Math | Custom (Q16.16 fixed-point matrices + selective float transform) |
| Memory | Custom scratch arena allocator |
| Debug | Custom cycle-count profiler (Xtensa CCOUNT) |

## License

(c) 2026. All rights reserved.

This repository contains project documentation only. Source code is not publicly available.
