# arm2d-skyer

[English](README.md) | [简体中文](README.zh-CN.md)

用于 ARM-2D 嵌入式 UI 开发的 Codex skill，覆盖 scene、PFB、generic loader、dirty region、GIF/图片/字体资源转换、Keil MDK/CMSIS-Pack 集成、显示适配、性能与内存调优，以及外设缓存数据接入。

## 文件结构

- `SKILL.md`：默认英文入口。
- `SKILL.zh-CN.md`：保留的中文入口。
- `references/`：6 份英文参考文档。
- `references/zh-CN/`：原版中文参考文档；原移植文档本身还包含英文部分。
- `python/`：8 个资源处理及压缩工具。

RP2040 Flip / Crystal Mouse 文档描述的是特定示例工程，固件本身不在本仓库中。引脚、地址及接口必须对照目标板和当前工程核实。

## 安装

将仓库克隆到 Codex 的 skills 目录。如果同名目录已经存在，先比较或备份再替换。

```powershell
git clone https://github.com/Ttrrace/arm2d-skyer.git "$env:USERPROFILE\.codex\skills\arm2d-skyer"
```

自定义了 `CODEX_HOME` 时，使用其下的 `skills/arm2d-skyer`。macOS/Linux 命令见英文 README。

在新聊天中使用 `$arm2d-skyer`，或直接提出符合描述的 ARM-2D 任务。默认入口是英文，可以要求用中文回答，中文原文见 `SKILL.zh-CN.md`。

## Python 工具

阅读和使用 skill 文档不需要 Python。运行资源转换工具时，按所选脚本安装依赖：

```sh
python -m pip install Pillow numpy Jinja2 freetype-py
python python/gif2png.py --help
python python/img2c.py --help
```

`gif_tile_tool.py` 和 `video_rembg_tool.py` 直接执行配置区指定的处理流程，运行前先修改配置，不要用 `--help` 试运行。示例路径已改为相对当前工作目录的路径。视频工具还导入 `opencv-python` 和 `rembg`，并需要适合本机的推理后端；本次文档翻译未验证该可选处理链路。

生成资源前，确认目标 Pack、loader、像素格式、尺寸和 SRAM/Flash 预算。

## 来源与许可

这是独立的 skill 包，不是 Arm 官方产品。保留工具中的版权与许可声明，详见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。未带明确许可的文件不额外授予仓库级许可。
