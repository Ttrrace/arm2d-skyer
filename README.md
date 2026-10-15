# arm2d-skyer

[English](README.md) | [简体中文](README.zh-CN.md)

A Codex skill for practical ARM-2D embedded UI development: scenes, partial framebuffers (PFBs), generic loaders, dirty regions, asset conversion, Keil MDK/CMSIS-Pack integration, display adapters, performance tuning, and peripheral data services.

## Contents

- [SKILL.md](SKILL.md): English skill entrypoint.
- [SKILL.zh-CN.md](SKILL.zh-CN.md): original Chinese entrypoint.
- `references/`: English guides for loaders, assets, porting, Keil, RP2040 performance, and peripheral integration.
- `references/zh-CN/`: original Chinese reference files (the original porting reference also includes English).
- `python/`: eight asset-processing and compression tools.

Board-specific references describe an example RP2040 Flip / Crystal Mouse project; that firmware is not included. Verify the target hardware and current APIs before applying its pin assignments or interfaces.

## Install

Clone this repository into your Codex skills directory as `arm2d-skyer`. If you already have that directory, compare or back up its contents before replacing it.

```powershell
# Windows PowerShell; assumes the default Codex home
git clone https://github.com/Ttrrace/arm2d-skyer.git "$env:USERPROFILE\.codex\skills\arm2d-skyer"
```

```sh
# macOS / Linux; honors a custom CODEX_HOME
git clone https://github.com/Ttrrace/arm2d-skyer.git "${CODEX_HOME:-$HOME/.codex}/skills/arm2d-skyer"
```

If `CODEX_HOME` is customized on Windows, use its `skills` subdirectory instead. In a new Codex chat, invoke `$arm2d-skyer` or ask for an ARM-2D task matching its description. English is the default entrypoint; request Chinese guidance or read `SKILL.zh-CN.md` for the original wording.

Example requests:

- Integrate an existing scene into a Keil/RTE ARM-2D project.
- Convert a GIF for RGB565 playback while keeping Flash usage low.
- Diagnose PFB/scratch allocation failures or slow rendering.
- Feed cached IMU/RTC data into a scene without blocking drawing callbacks.

## Asset Tools

The skill documentation requires no Python installation. To run asset tools, install the dependencies needed by the chosen script:

```sh
python -m pip install Pillow numpy Jinja2 freetype-py
python python/gif2png.py --help
python python/img2c.py --help
```

`gif_tile_tool.py` and `video_rembg_tool.py` execute directly from their configuration block; edit it before running. Their example paths are relative to the working directory. The video helper additionally imports `opencv-python` and `rembg`; choose an appropriate rembg inference backend for your environment. Its optional processing path is not validated by the documentation translation.

The tools do not establish target-side decoder support. Confirm the Arm-2D Pack version, loader, pixel format, output dimensions, and memory budget before adding generated assets to firmware.

## Attribution

This is an independent skill package, not an official Arm product. Preserve the copyright and license headers in bundled tools. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). No repository-wide license is assigned to files lacking an explicit license notice.
