# ARM-2D Generic Loader Reference

## When to Use

Use a generic loader when:

- Rendering dynamic or procedural content, such as particle simulations.
- Generating or decoding assets block by block for an ROI.
- Assets are too large to decode full frames into SRAM.
- Combining PFB, ROI, and dirty regions for partial refresh.

## Basic Structure

Create a module pair, for example:

```text
project/mdk/<feature>/user_generic_loader_<feature>.c
project/mdk/<feature>/user_generic_loader_<feature>.h
```

The header commonly contains:

```c
typedef struct xxx_cfg_t {
    arm_2d_size_t tSize;
    uint16_t bUseHeapForVRES : 1;

    struct {
        const arm_loader_io_t *ptIO;
        uintptr_t pTarget;
    } ImageIO;

    arm_2d_scene_t *ptScene;
} xxx_cfg_t;

typedef struct xxx_t xxx_t;

struct xxx_t {
    union {
        arm_2d_tile_t tTile;
        inherit(arm_generic_loader_t);
    };

ARM_PRIVATE(
    xxx_cfg_t tCFG;
)
};
```

The C implementation commonly starts with:

```c
#define __GENERIC_LOADER_INHERIT__
#define __xxx_IMPLEMENT__

#include "user_generic_loader_xxx.h"
```

## Initialization Pattern

```c
arm_generic_loader_cfg_t tCFG = {
    .bUseHeapForVRES = this.tCFG.bUseHeapForVRES,
    .tColourInfo.chScheme = ARM_2D_COLOUR,
    .bBlendWithBG = false,
    .ImageIO = {
        .ptIO = this.tCFG.ImageIO.ptIO,
        .pTarget = this.tCFG.ImageIO.pTarget,
    },
    .UserDecoder = {
        .fnDecoderInit = &__xxx_decoder_init,
        .fnDecode = &__xxx_draw,
    },
    .ptScene = this.tCFG.ptScene,
};

arm_generic_loader_init(&this.use_as__arm_generic_loader_t, &tCFG);
```

Set the tile size after initialization:

```c
this.tTile.tRegion.tSize = this.tCFG.tSize;
```

## Show Pattern

```c
void xxx_show(xxx_t *ptThis,
              const arm_2d_tile_t *ptTile,
              const arm_2d_region_t *ptRegion,
              bool bIsNewFrame)
{
    ARM_2D_UNUSED(bIsNewFrame);

    if (-1 == (intptr_t)ptTile) {
        ptTile = arm_2d_get_default_frame_buffer();
    }

    arm_2d_tile_copy_only(&this.tTile, ptTile, ptRegion);
}
```

## fnDecode Contract

Function signature:

```c
static arm_2d_err_t __xxx_draw(
    arm_generic_loader_t *ptObj,
    arm_2d_region_t *ptROI,
    uint8_t *pchBuffer,
    uint32_t iTargetStrideInByte,
    uint_fast8_t chBitsPerPixel)
```

Rules:

- Write only the region corresponding to `ptROI`.
- Do not assume the target is a contiguous full-screen framebuffer.
- Advance each row using `iTargetStrideInByte`.
- A typical RGB565 loop:

```c
for (int_fast16_t y = yStart; y < yLimit; y++) {
    uint16_t *phwPixel = (uint16_t *)pchBuffer;

    for (int_fast16_t x = xStart; x < xLimit; x++) {
        *phwPixel++ = colour;
    }

    pchBuffer += iTargetStrideInByte;
}
```

## Scene Integration

Include the loader header in `arm_2d_scene_0.h`.

In `arm_2d_scene_0.c`:

```c
xxx_t Xxx;
```

Load:

```c
xxx_on_load(&Xxx);
```

Frame start:

```c
xxx_on_frame_start(&Xxx);
```

Depose:

```c
xxx_depose(&Xxx);
```

PFB draw handler:

```c
arm_2d_align_centre(__top_canvas, 240, 240) {
    xxx_show(&Xxx, ptTile, &__centre_region, bIsNewFrame);
}
```

Initialize:

```c
xxx_cfg_t tCFG = {
    .tSize = {
        .iWidth = 240,
        .iHeight = 240,
    },
    .ptScene = &this.use_as__arm_2d_scene_t,
};
xxx_init(&Xxx, &tCFG);
```

## Notes

- For expensive decoders, do not recompute the entire frame for every ROI.
- Consider a low-resolution cache sampled and upscaled for each ROI.
- For compressed assets, prefer ROI-aware or tile/block decoding where possible.
