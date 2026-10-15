# RP2040 / Tufty2040 Performance and Memory

These notes apply to RP2040/Tufty2040 constraints, not every MCU. RP2040 can produce attractive small-screen Arm-2D UIs but has no desktop GPU or hardware FPU.

## Memory Accounting

Keil reports:

```text
Program Size: Code=... RO-data=... RW-data=... ZI-data=...
```

RW/ZI describe static memory only. Reserve runtime space for stack, heap, Arm-2D scratch, PFB blocks, generic-loader resources, peripherals, DMA, and SDK structures. Near-limit ZI can yield a successful build followed by a black screen or hang.

## Expensive Patterns

Avoid excessive per-pixel 64-bit division/multiplication, complex iteration, per-pixel sin/exp/log (including surrounding LUT calculations), full-screen 240x240 computation every frame, and large double buffers.

## Preferred Approaches

- Low-resolution caches with upscaling, such as `80x80 -> 240x240` or `60x60 -> 240x240`: much less computation, simple ROI sampling, and possible double buffering. Suits abstract effects; blurring makes it less suitable for detailed UI.
- Precomputed Flash maps for black-hole/vortex/polar effects: radius, angle, distortion, or palette-index maps. Runtime work becomes frame offsets, palette lookups, and simple brightness adjustments.
- Palette animation: a demo-scene technique with low SRAM/CPU use and strong visual motion.
- Tile/block updates: update only changing tiles for compressed animation where supported.

## Dual Core

RP2040 has two cores, but Keil/CMSIS-Pack integration requires care. `multicore_launch_core1()` depends on SDK runtime; core1 stack space may be insufficient; FIFO use can conflict with SDK logic; debugging/startup can change timing; heavy core1 computation can expose IRQ/PIO/DMA issues.

Establish a stable single-core version first and use a compile-time option:

```c
#ifndef FEATURE_USE_CORE1
#   define FEATURE_USE_CORE1 0
#endif
```

If dual-core operation produces a black screen, disable it to verify the primary rendering path.

## Diagnosing Bottlenecks

For working display with low FPS: reduce cache resolution, reduce iterations, remove division, replace computation with lookup tables, shrink updated regions, and use precomputed assets.

For a black screen or failure to start: reduce ZI, disable core1, replace complex effects with color bars/gradients to validate the loader, check scratch/PFB allocation failures, and confirm the scene is appended correctly.
