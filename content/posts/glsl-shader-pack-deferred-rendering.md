---
title: "Building a GLSL Shader Pack with a Full Deferred Rendering Pipeline"
date: 2026-06-29T12:30:00+10:00
draft: true
description: "How I built UltraRealism — a custom GLSL shader pack for Minecraft with deferred rendering, PBR, screen-space path tracing, and TAA — and the driver-level debugging that was most of the work."
tags: ["glsl", "shaders", "graphics", "minecraft", "nvidia", "debugging"]
---

> **Draft note — needs your review before publishing.** The notes for this post have several `[CHECK]` markers where specifics need to be confirmed. I've flagged them inline below. Fill these in and remove the draft flag before publishing.

UltraRealism is a custom GLSL shader pack I built for Minecraft Java Edition, running through Iris Shaders on Forge. The goal was a full from-scratch realistic rendering pipeline — not a tweak of an existing pack. The result was a deferred renderer with PBR lighting, screen-space path tracing for global illumination, and TAA.

Most of what I'd actually write up about this project is the debugging. The rendering architecture is interesting but followable. The driver-level GLSL quirks on an RTX 4090 under Iris are not well documented, and the fixes are the kind of hard-won detail that's actually useful to someone hitting the same errors.

## The pipeline

**G-buffer pass:** geometry rendered to a set of textures — albedo, normals, roughness, metalness, depth. Nothing is lit yet at this stage.

**Lighting pass:** reads from the G-buffer and applies PBR lighting. Physically based — albedo/normal/roughness/metalness all feed into the BRDF properly rather than approximations.

**Screen-space path tracing:** used for global illumination. [CHECK — confirm how far this actually got in the build: full GI, ambient occlusion only, or something in between?]

**TAA (temporal anti-aliasing):** accumulates samples across frames to reduce noise from the path tracing pass without destroying performance.

**Post-processing stack:** [CHECK — list the actual effects: tonemapping operator used, bloom, anything else in the stack?]

Environment info: [CHECK — Forge version, Iris version, Minecraft 1.x.x?] on an RTX 4090.

## The debugging story

There were two recurring classes of problem throughout development. The renderer architecture was iterative and mostly solved by working through it. These two categories were persistent.

### Iris Shaders compatibility

[CHECK — what were the specific symptoms? Shader compilation failures at load time, rendering artifacts, specific passes not running? Fill this in with real error messages or behaviour if you remember them.]

### NVIDIA GLSL driver errors on the RTX 4090

This was the harder class of problem, and the less-documented one.

NVIDIA's Windows GLSL driver on high-end hardware can reject valid GLSL code that other drivers accept, or silently miscompile it. Three fixes that actually worked:

**GLSL version downgrade.** [CHECK — what version did you downgrade from/to? e.g. `#version 460 core` → `#version 450`?] The higher version was hitting a compatibility break in the driver. Downgrading resolved it.

**Inlining `#include` files.** The Iris shader include mechanism was causing failures in some passes. Fix was to inline the included files directly into the shader source instead of relying on the include path at compile time.

**Float-only hashing.** This was the key unlock. A hashing function in the path tracing pass used `uint` arithmetic. The NVIDIA driver was tripping on the uint path — either miscompiling it or rejecting it outright. Replacing the hash with a float-only implementation fixed it. [CHECK — do you remember the specific shader file or pass this was in? And roughly what the uint hash looked like vs the float replacement?]

## Honest note on how this was built

This stretched well past my comfort zone in shader code. The value of using Claude through the compile/debug loops was that I could move fast — describe the error, get a candidate fix, test, iterate — while I steered the architecture and made the decisions about what the pipeline should be doing. I understood enough to know when a fix was correct and when it wasn't. That's the useful part.

The driver-level GLSL work is genuinely poorly documented. If you're hitting RTX + Iris shader compile failures and the uint→float hash swap or the version downgrade rings a bell, those are real fixes.

---

*[Remove this draft flag and the note at the top once the `[CHECK]` items above are filled in.]*
