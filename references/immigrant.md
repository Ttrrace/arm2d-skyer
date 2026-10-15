# Arm-2D Deployment With MDK/RTE

### When To Use

Use this skill for tasks such as:

- Adding Arm-2D to a Keil/MDK project through CMSIS-Pack/RTE.
- Configuring `arm_2d_cfg.h` and a display adapter.
- Connecting Arm-2D to LCD/OLED/RGB/MIPI/framebuffer displays.
- Tuning PFB, dirty regions, synchronous flushing, and asynchronous flushing.
- Integrating Arm-2D benchmarks, official demos, or custom scenes.
- Debugging Arm-2D frame rate, flush time, DMA/display port behavior, and rendering bottlenecks.

### Deployment Principles

Prefer Pack/RTE integration instead of manually copying the whole Arm-2D source tree into the project.

- Keep Arm-2D selectable and upgradable from MDK's Manage Run-Time Environment.
- Do not edit files inside the installed Pack directory.
- Keep project-specific code in local project folders such as `RTE/`, `board/`, `bsp/`, `drivers/`, or `applications/`.
- Isolate display-hardware logic in the display port/display adapter. Application code should use Arm-2D scene/demo APIs.

### Pack/RTE Integration Steps

1. Install the Arm-2D Pack.
2. If redeploying Arm-2D or switching Pack/RTE versions, clear the project's `RTE/Acceleration` directory first, then let MDK/RTE regenerate it.
3. Select the required Arm-2D components in MDK's RTE configuration. Unless the user requests a minimal configuration, selecting all Arm-2D components is acceptable; the compiler/linker can remove unused code. Avoid prematurely omitting helpers, headers, or scene/demo support. In projects using an RTOS, also select Arm-2D RTOS Helper.
4. Confirm the project has Arm-2D configuration files.
5. Confirm include paths cover the generated RTE folder, Arm-2D configuration folder, display adapter/port folder, and application folder.
6. Keep Arm-2D configuration project-local. Do not patch the Pack source itself.

Clearing `RTE/Acceleration` prevents stale generated files, old configuration headers, and old component references from being mixed with the new Pack version. If the directory contains hand-written project configuration, migrate or back it up into the project's own board/application configuration area first.

Common project files:

```text
RTE/Acceleration/arm_2d_cfg.h
RTE/Acceleration/arm_2d_disp_adapter_0.h
board/arm2d/
applications/
```

The display adapter header usually holds screen/PFB/flush configuration; `arm_2d_cfg.h` usually holds global/helper/dirty-region/memory settings. Re-check PFB dimensions against the target resolution, pixel format, SRAM, DMA, and display interface instead of copying another project's settings.

The exact paths may differ. The important split is:

- Pack files provide generic Arm-2D source and components.
- RTE/project-local files provide configuration.
- board/driver code provides screen flush, DMA, cache, and synchronization hooks.
- application code provides scenes and demos.

### Initialization Flow

Typical startup sequence:

```c
arm_2d_init();
disp_adapter0_init();
```

Then drive the display adapter from the main loop or an RTOS thread:

```c
while (1) {
    (void)__disp_adapter0_task();
}
```

When measuring benchmark/demo performance, avoid adding unnecessary delay/yield calls unless they are part of the intended scheduling model.

### Display Adapter Configuration

A display adapter should define:

- Screen width and height.
- Pixel format, such as RGB565, RGB888, or ARGB8888.
- PFB block width and height.
- PFB count or heap size.
- Whether dirty-region optimization is enabled.
- Whether flushing is synchronous or asynchronous.
- Whether RGB565 byte swapping is needed.
- Cache clean/invalidate policy.

First make the display correct, then tune performance.

### PFB Configuration

PFB is Arm-2D's partial framebuffer/block rendering buffer. Tune it with these tradeoffs in mind:

- Too small: too many blocks, more scheduling and flush overhead.
- Too large: higher memory usage and sometimes less effective dirty-region clipping.
- PFB size should be measured against screen resolution, cache behavior, DMA alignment, bus efficiency, and scene type.
- If dynamic PFB is enabled, confirm that manual settings are actually taking effect.
- Alignment settings can improve DMA/display efficiency or increase the actual rendered area. Measure both.

Use hardware data as the final judge.

### Dirty Regions

Dirty regions are useful for UI, gauges, small animations, and scenes where only part of the screen changes.

- They can reduce CPU rendering and display flush workload.
- They may help less for fullscreen shaders, video-like content, or full-screen scrolling.
- Do not assume disabling dirty regions is faster. Measure FPS, flush count, and actual flushed area.

When validating dirty regions, check:

- FPS/frame time.
- Flush count per frame.
- Flush pixel count or region sizes.
- Missing updates, ghosting, or a scene that stops after one frame.

### Synchronous And Asynchronous Flushing

Synchronous flushing:

- Is simple and deterministic.
- Works well with one PFB or very fast display flushes.
- Can be faster when flush time is already low.

Asynchronous flushing:

- Tries to let the CPU prepare the next PFB while DMA/display hardware flushes the previous one.
- Usually requires at least two PFB buffers.
- Requires correct flush-complete, buffer-release, cache, and concurrency handling.
- With only one PFB, async flushing usually cannot provide real overlap and may add callback/ISR/state-machine overhead.

Enable async flushing because measurements prove it helps, not because the name sounds faster.

### Display Flush Port

The display flush port is the key bridge between Arm-2D and the physical display.

Common implementations:

- CPU copy into a framebuffer.
- 2D DMA/accelerator copy into a framebuffer.
- SPI/QSPI/MIPI/8080 transfer of the updated region.
- LTDC/RGB display with a framebuffer where flush updates the matching region.

Verify:

- Source PFB address, target framebuffer address, and stride.
- Pixel format compatibility.
- RGB565 byte order.
- Cache clean/invalidate behavior.
- DMA completion callback or synchronous wait logic.
- Error flags or counters.

If a 2D DMA/GPU/accelerator is available, prefer using it for large copies or format conversion while the CPU handles scene logic and required software rendering.

### Demo And Scene Integration

Keep Arm-2D demos in the application layer and decoupled from the display port.

Common organization:

```text
applications/arm2d_demo.c
applications/arm2d_demo.h
applications/arm2d_scene_xxx.c
applications/arm2d_scene_xxx.h
```

Scene files often include:

```c
#include "arm_2d.h"
#include "arm_2d_helper.h"
#include "arm_2d_disp_adapter_0.h"
```

Common Arm-2D scene render callback form:

```c
arm_fsm_rt_t fnScene(void *pTarget,
                     const arm_2d_tile_t *ptTile,
                     bool bIsNewFrame);
```

For custom scenes:

- Use Arm-2D tile/region APIs to get the valid area of the current PFB.
- Do not assume every render pass covers the full screen.
- For fullscreen animation, explicitly enable full-frame refresh or mark the frame dirty every frame.
- Benchmarks, official demos, and custom demos should be replaceable without hardwiring display-driver code into the demo.

### Arm-2D API Notes And Pitfalls

Do not guess Arm-2D APIs from intuition. Check the headers and demos from the active Pack version before writing adapter or scene code.

Core concepts:

- `arm_2d_tile_t` represents a 2D pixel area. It may be a framebuffer, PFB, child tile, image asset, or generic render target.
- `arm_2d_region_t` represents a rectangular area inside a tile, including location and size.
- `arm_2d_location_t` represents a 2D coordinate.
- `arm_fsm_rt_t` is a common Arm-2D async/state-machine return type. Do not treat it as plain `void` or `bool`.
- `arm_2d_scene_t` is the scene base used by the scene player. User scenes commonly embed it with `implement(arm_2d_scene_t);`.

A common scene callback signature is:

```c
arm_fsm_rt_t fnScene(void *pTarget,
                     const arm_2d_tile_t *ptTile,
                     bool bIsNewFrame);
```

The first parameter is usually `void *pTarget`, not `arm_2d_scene_t *`. If custom scene state is needed, follow the current demo/framework pattern to recover it. Do not invent a signature.

When getting the valid PFB area and pixel pointer from a tile, prefer Arm-2D helper/API calls instead of hand-rolled pointer arithmetic:

```c
arm_2d_location_t tOffset;
arm_2d_region_t tValidRegion;
uint16_t *phwBuffer;

arm_2d_get_address_and_region_from_tile(ptTile,
                                        &tValidRegion,
                                        &tOffset,
                                        uint16_t,
                                        phwBuffer);
```

When writing pixels, distinguish:

- The current valid PFB region `tValidRegion`.
- The current PFB offset in full-screen coordinates `tOffset`.
- The destination tile stride.
- The pixel format and pixel type, such as `uint16_t` for RGB565.

Common mistakes:

- Assuming every scene callback receives the full-screen framebuffer.
- Ignoring PFB offset so every block renders as if it starts at logical `(0,0)`.
- Using region width as stride when the actual stride is the destination tile/framebuffer line width.
- Ignoring `bIsNewFrame`, which can break animation timing, state updates, or dirty marking.
- Forgetting full-frame refresh, causing procedural animation or shaders to render one frame and stop.
- Treating Arm-2D async return values as synchronous function results.

For fullscreen animation, shaders, or real-time gauges that change every frame, explicitly enable full-frame refresh or mark dirty regions every frame:

```c
arm_2d_helper_pfb_full_frame_refresh_mode(&DISP_ADAPTER.use_as__arm_2d_helper_pfb_t,
                                          true);
```

Replace `DISP_ADAPTER` with the actual display adapter object used by the project.

Scene/player/demo guidance:

- Start from the Pack-provided demo initialization and scene append flow.
- Do not bypass the scene player and poke internal fields unless the current version's docs or demos show that pattern.
- Re-check function signatures and struct fields after changing Arm-2D Pack versions.
- Keep custom demos decoupled from the display flush port.

### Performance Verification

Establish observability before optimizing.

Recommended metrics:

- FPS.
- Frame time.
- Flush time.
- Flush count.
- Flush pixel count or region size.
- CPU load.
- DMA/display error count.
- Cache, bus, or alignment-related faults.

Bottleneck interpretation:

- Low flush time but high frame time: likely CPU rendering bottleneck.
- High flush time: inspect display bus, DMA, framebuffer, cache, and region size.
- High FPS but no increasing flush count: likely stale/fake data or a scene that is not refreshing.
- Ghosting or partial updates: inspect dirty regions, tile regions, and full-frame refresh logic.

### Build, Download, And Runtime Verification

Generic flow:

1. Build the project.
2. Download it to the target.
3. Confirm the program resets and runs.
4. Read or print Arm-2D runtime statistics.
5. Compare PFB, dirty-region, async, DMA, and cache configurations.
6. After enabling LTO or changing optimization levels, re-check symbol addresses and runtime state.

In MDK command-line workflows, UV4 can automate build/download. With a debugger/probe tool, read RAM statistics to validate runtime behavior.

### Common Pitfalls

- Redeploying Arm-2D Pack/RTE without clearing `RTE/Acceleration`, which can mix stale generated files with new components.
- Arm-2D Pack/RTE components are selected, but include paths miss the project-local config directory.
- PFB settings are changed, but dynamic PFB or other macros override them.
- RGB565 colors are wrong because of RGB/BGR order or byte order.
- DCache is enabled but clean/invalidate is missing, so DMA/display reads stale data.
- Async flush is enabled with only one PFB and no real pipeline overlap happens.
- Dirty-region logic causes a scene to render one frame and then stop.
- Benchmark FPS does not match application FPS because the scene workload is different.
- LTO moves symbols and the debug tool reads old addresses.
