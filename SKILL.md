---
name: arm2d-skyer
description: Develop ARM-2D embedded UIs with scenes, PFBs, generic loaders, dirty regions, RGB565 assets, GIF and sprite animation pipelines, Keil MDK/RTE integration, display adapters, MCU performance and memory tuning, procedural effects, and board peripheral services consumed by scenes.
---

# ARM-2D Embedded UI Workflow

Use this skill to work on ARM-2D projects in a portable, maintainable way that respects the target hardware. Focus on practical engineering rather than generic tutorials.

## Typical Context

- Language: C.
- Graphics stack: ARM-2D, PFB, Scene Player, Generic Loader.
- Project environments: Keil MDK, CMSIS-Pack/RTE, RTOS or bare metal.
- Displays: LCD, OLED, RGB, MIPI, framebuffer; SPI, QSPI, 8080, LTDC, and similar interfaces.
- Pixel formats: RGB565, RGB888, ARGB8888.
- Tasks: dynamic UIs, GIF/animation assets, partial refresh, low-RAM rendering, and procedural shader-like effects.

## Core Principles

- Favor code that runs reliably on the target MCU over theoretical elegance.
- For animations, effects, and large images, examine SRAM, Flash, CPU time per frame, interference with Arm-2D/PFB flushing, FPU availability, and the configured floating-point ABI/FPU options.
- Before choosing fixed point, lookup tables, integer approximations, floating point, or acceleration, inspect the actual MCU/SoC for FPU, DSP/SIMD, 2D accelerators, DMA, and display capabilities. Do not generalize one chip's optimization rules to other platforms.
- A successful Keil build does not establish runtime safety: scratch, PFB, and generic-loader allocations can still fail.
- Do not propose uncompressed full-frame arrays as the final solution unless Flash capacity clearly permits them.
- Verify multicore and hardware acceleration capabilities in the project before enabling them.
- Inspect an existing project before changing it; preserve working scenes and loaders.
- For UI requests, search the Arm-2D library, current project, Pack examples, and existing demos/scenes for APIs, helpers, controls, and patterns before composing an implementation. Do not invent APIs or immediately build a separate drawing framework.
- Arm-2D tools include Python generators for fonts, masks, image arrays, and GIF/animation assets. Use existing generators when appropriate after verifying their arguments, output format, and target display resolution.

## Standard Workflow

1. Inspect the project and hardware:
   - MCU/SoC, core, FPU, DSP/SIMD, and 2D acceleration.
   - FPU, float ABI, optimization, and LTO settings in Keil/CMake/Makefile configurations.
   - Display resolution, pixel format, interface, and DMA/display flush path.
   - `main.c`, `project/mdk/template.uvprojx`, `project/mdk/RTE/Acceleration/arm_2d_scene_*.c`, existing `user_generic_loader_*`, and display adapter/platform/ST7789 files, where present.
2. Identify the task: layout/controls/transitions, static assets, GIF/animation, dynamic rendering/generic loader, procedural effects, memory/performance tuning, or peripheral/sensor/key/RTC integration.
3. For UI, scene, animation, or drawing API work:
   - Search `arm_2d_helper.h`, `arm_2d_scene_*.c`, official demos, current scenes, and the display adapter.
   - Learn parameters, lifetimes, asynchronous completion, and return semantics from official demos, Pack examples, and existing calls. Do not infer usage from names alone.
   - Reuse tile, region, opacity, mask, transform, dirty-region, scene-player, and helper APIs.
   - Add a new framework, control system, or rendering layer only after confirming that existing Arm-2D capabilities cannot satisfy the request.
4. For peripherals, sensors, keys, IR, RTC, or buzzers:
   - Check drivers, application wrappers, initialization order, pins/buses, return values, and actual call sites.
   - For the RP2040 Flip / Crystal Mouse example project, read [board peripheral integration](references/rp2040-flip-board-peripherals.md).
   - Scenes consume cached application snapshots. Keep I2C, repeated ADC sampling, blocking waits, and frequent logs out of PFB draw handlers.
   - Distinguish existing repository interfaces from proposed service-layer APIs.
5. Prefer independent modules that preserve working code.
6. Update Keil project entries when adding `.c/.h` files.
7. Build when possible and report changed files, Code/RO-data/RW-data/ZI-data changes, errors/warnings, and runtime risks.

## References

Read only the references relevant to the task:

- [Generic loader](references/generic-loader.md): loader structure, ROI decoding, and scene integration.
- [Asset pipeline](references/asset-pipeline.md): GIF, sprites, RGB565, masks, and zhRGB565.
- [Porting and integration](references/immigrant.md): Pack/RTE, display adapters, PFB, dirty regions, RTOS Helper, and build/download/runtime verification.
- [RP2040 performance](references/rp2040-performance.md): guidance for RP2040/Tufty2040 and similar resource-constrained MCUs without an FPU; do not apply its limits universally.
- [Keil projects](references/keil-project.md): project-file edits and build settings.
- [RP2040 Flip peripherals](references/rp2040-flip-board-peripherals.md): QMI8658, power key, IR, light sensing, proposed environmental sensing, BM8563, PAJ7620, buzzer, service contracts, and scene integration.

The original Chinese entrypoint is [SKILL.zh-CN.md](SKILL.zh-CN.md); original reference documents are in `references/zh-CN/`.

## UI Requests

For pages, controls, animations, transitions, dashboards, menus, progress bars, buttons, lists, dialogs, and icon overlays:

1. Search existing capabilities first.
2. Inspect current `arm_2d_scene_*.c/.h`, `user_scene_*`, `user_*_view`, display adapters, Pack helpers/scene player/demos, and headers such as `arm_2d.h`, `arm_2d_helper.h`, and `arm_2d_types.h`.
3. Follow demo initialization, calls, asynchronous waits, and resource release. Match existing project styles and lifetimes. Resolve uncertainty through demos, source, or call sites.
4. Reuse tiles/regions/child tiles; opacity/alpha/masks; copy/fill/color keys/mirroring/rotation/transforms; dirty regions/PFB/scene player; and helper/demo patterns for controls, progress bars, lists, text, and gauges.
5. Add lightweight wrappers only when needed, aligned with Arm-2D's tile/region/scene model.
6. Explain which existing APIs/examples are reused and which logic is new when presenting a design or implementation.

## Python Asset Tools

Bundled tools are in `python/`. Also inspect the Arm-2D library, Pack, and current project for existing resource generators.

For images, fonts, masks, GIFs, animation frames, or C arrays:

1. Prefer existing scripts over one-off converters unless they cannot meet the requirement.
2. Generate resources and add the resulting `.c/.h` or asset files to the project as appropriate.
3. Verify target resolution, pixel format, and intended display region before generation.
4. Clarify missing material parameters: resource/tile/C-array names; target width/height; aspect ratio, cropping, centering or filling; RGB565/RGB888/ARGB8888; mask/alpha/color key; and GIF frame selection, rate limits, scaling, sprite layout, or loader-supported format.
5. Read script arguments and output conventions before running commands.

Tools:

- `python/gif2png.py`: GIF to sprite sheet.
- `python/img2c.py`: image to Arm-2D tile/RGB565/mask/zhRGB565 assets.
- `python/jinja2c.py`: C generation from templates.
- `python/ttf2c.py`: font to C data.
- `python/__img2c_lmsk.py`: LMSK compression helper.
- `python/__img2c_zhRGB565.py`: zhRGB565 compression helper.
- `python/gif_tile_tool.py`: configurable sprite-sheet preview helper.
- `python/video_rembg_tool.py`: configurable video frame extraction/background-removal helper.

The last two are configuration-driven scripts: edit their configuration block before execution. Read their source rather than invoking `--help`, since they execute processing at the top level.

## Generic Loader Conventions

Consider a generic loader for dynamic content, compressed asset decoding, and frame-by-frame animation output.

Typical module names:

```text
user_generic_loader_xxx.c
user_generic_loader_xxx.h
```

Typical lifecycle operations:

```c
xxx_init();
xxx_depose();
xxx_on_load();
xxx_on_frame_start();
xxx_on_frame_complete();
xxx_show();
```

`fnDecode()` must respect `ptROI`; do not assume a contiguous full-screen framebuffer. See [generic-loader.md](references/generic-loader.md).

## GIF and Animation Assets

Base solutions on existing scripts and actual project loader support. Do not describe an unimplemented container or decoder as usable.

- Uncompressed RGB565 frames are simplest to play but consume substantial Flash.
- Sprite sheets are convenient intermediates and may not be suitable final firmware assets.
- zhRGB565/QOI require corresponding loaders/components; add the required loader or use a supported format if absent.
- Confirm whether masks/alpha exist and how composition works.
- For sprite/zhRGB565 animations streamed from filesystems or XIP Flash, inspect frame layout and actual loader access order. Prefer a frame that spans the full sheet width and occupies a contiguous address range. Common generators achieve this with `columns=1`, a vertical single column, not a horizontal single row.

When Flash savings matter, consider frame-rate limits, scaling, cropping, tile/delta updates, palette animation, and implementable zhRGB565/QOI paths. See [asset-pipeline.md](references/asset-pipeline.md).

## Performance

Watch for per-pixel 64-bit division, complex iteration, full-screen computation every frame, large double buffers that exhaust scratch/PFB memory, and multicore startup/stack/critical-section/debugging complexity.

Prefer low-resolution caches with upscaling, palette animation, lookup tables and precomputed Flash maps, ROI-aware decoding, tile/block updates, and shader-like procedural effects suited to MCUs.

For MCUs without an FPU or with costly floating point, favor fixed point, lookup tables, and integer approximations. For hardware with FPU/DSP/SIMD/2D acceleration, decide using compiler settings and actual profiling. The [RP2040 notes](references/rp2040-performance.md) apply only where their platform constraints match.

## Boundaries

- Do not delete existing project files unless explicitly requested.
- Keep large generated assets out of the skill package.
- Do not claim unverified decoders work.
- Do not invent Arm-2D APIs; check current project code or local Pack examples.
- Avoid directly changing official Arm-2D APIs, Pack source interfaces, or private structure layouts. Use public APIs, configuration, wrappers/adapters, project-local extensions, or upstream patches.
- Do not depend on private structure fields unless official demos/public headers permit access. Explain version coupling and maintenance risks if such access is necessary.
