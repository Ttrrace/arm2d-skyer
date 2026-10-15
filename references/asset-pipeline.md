# Asset and Animation Pipeline

## Goals

Balance Flash, SRAM, decoding speed, transparency/masking, partial refresh, and maintainability against the actual MCU/SoC constraints.

## Common Formats

### Uncompressed RGB565 Frames

Playback is simple, needs no decoder, and can use `arm_2d_tile_t` directly. Flash consumption is high:

```text
width * height * 2 * frame_count
240 * 240 * 2 * 80 = 9.216 MB
```

Use as a final format only when Flash capacity permits it.

### Sprite Sheets

Useful for desktop previews and intermediate assets. Firmware still stores raw or compressed pixels; a sprite sheet alone does not save Flash.

### Masks and Alpha

For transparent animation, establish whether to use a 1-bit mask or A2/A4/A8 alpha, whether antialiased edges are required, and whether hard transparency edges are acceptable. A 1-bit mask saves space but can produce jagged edges.

### zhRGB565 / QOI

Suitable for compressed image assets. Confirm the corresponding Arm-2D loader/component is enabled. Bundled zhRGB565 helpers do not establish that a target project can decode their output.

## Frame Layout and Storage Contiguity

When an entire sprite sheet is encoded by scanline and a file/ROM loader reads per-frame ROIs:

- In a multicolumn sheet, each frame scanline is usually contiguous, but other frames separate successive scanlines. Reading a frame spans multiple disjoint ranges.
- In a vertical single column, a frame fills the sheet width and its scanlines are adjacent, supporting sequential reads, block caching, and prefetching.
- Common generators use `columns=1` for a vertical single column. The reliable criterion is `frame_width == sheet_width`; do not confuse this with a horizontal single row.

This reduces seeks, fragmented reads, and cache misses; it does not inherently accelerate compression. Benefits typically decrease in this order:

```text
SD + FatFS > SPI/QSPI XIP Flash > actual on-chip Flash > SRAM
```

1. Inspect whether the loader accesses whole-sheet row offsets and count seeks, reads, and cache refills per frame.
2. For SD/FatFS or XIP streaming, prefer vertical single-column assets (`columns=1`) and align reads to sectors/cache lines.
3. Record compressed size, average/worst frame time, seek/read counts, and cache hits. Do not compare size or perceived FPS alone.
4. If each frame is independently stored in one contiguous block, follow its index and actual I/O behavior instead of applying sprite-sheet conclusions mechanically.

## Choosing an Approach

- Quick visual verification: temporary RGB565 frames or sprite-to-array conversion are acceptable.
- Flash savings: reduce dimensions/rate/frame count, use QOI/zhRGB565 where supported, update tiles/blocks, animate palettes, or store only changing foreground regions over a fixed background.
- Transparent GIF: confirm background stability, antialiased-edge requirements, 1-bit-mask acceptability, target size, and frame count.
- Real-time effects: prefer low-resolution buffers with upscaling, palette animation, and precomputed Flash maps.

## Python Tools

In `python/`: `gif2png.py` converts GIFs to sprite sheets; `img2c.py` generates C/Arm-2D tiles; `ttf2c.py` converts fonts; `jinja2c.py` generates C from templates; `__img2c_lmsk.py` and `__img2c_zhRGB565.py` provide compression helpers. Read arguments or supported help before use.

Do not present animation containers without format documentation, desktop encoders/decoders, or an MCU generic loader as ready-to-use solutions. Automatic Keil resource insertion remains future work.
