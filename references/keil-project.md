# Keil MDK Project Notes

## Common Files

```text
project/mdk/template.uvprojx
project/mdk/template.uvoptx
project/mdk/RTE/Acceleration/arm_2d_scene_0.c
project/mdk/RTE/Acceleration/arm_2d_scene_0.h
```

## Adding Sources

After adding `.c/.h`, update `.uvprojx` for compilation, optionally synchronize `.uvoptx` for the IDE tree, and add new directories to IncludePath.

For `project/mdk/shader/user_generic_loader_shader_art.c/.h`, add a group and entries such as:

```xml
<File>
  <FileName>user_generic_loader_shader_art.c</FileName>
  <FileType>1</FileType>
  <FilePath>.\shader\user_generic_loader_shader_art.c</FilePath>
</File>
<File>
  <FileName>user_generic_loader_shader_art.h</FileName>
  <FileType>5</FileType>
  <FilePath>.\shader\user_generic_loader_shader_art.h</FilePath>
</File>
```

Example include path (retain actual project directory spellings):

```text
.;..\..\deivers;..\..\application;.\3d;.\fire;.\shader
```

## Optimization and LTO

Unless the user specifies otherwise, this skill's preferred ARM-2D/Keil configuration is `-Ofast` / `Ofast` with Link Time Optimization enabled. Scene rendering, PFB flushing, generic loaders, RGB565 processing, and procedural effects are sensitive to pixel-loop performance. LTO can inline across files and remove unused code.

- Prefer Ofast and LTO when creating or adjusting project configurations.
- Honor explicit O0/O1/O2/O3/Os/Oz choices.
- Disable LTO if requested or required by link errors, weak-symbol overrides, startup/vector issues, or section retention; explain why.
- Temporarily reduce optimization or disable LTO for exception debugging, stepping, memory-corruption investigation, or variable inspection; identify this as a debugging change.
- Report the actual optimization level and LTO state after changing either.
- If Ofast exposes floating-point, strict-aliasing, undefined-behavior, or timing issues, explain the issue and consider a local reduction for affected files before lowering optimization project-wide.

## Building

Locate the installed `UV4.exe` before invoking it. Example only:

```powershell
& 'D:\keil538\UV4\UV4.exe' -b '.\template.uvprojx' -j0 -o '.\Objects\build.log'
```

Report:

```text
Program Size: Code=... RO-data=... RW-data=... ZI-data=...
0 Error(s), 0 Warning(s)
```

Explain material RAM/Flash changes. Updating only `.uvoptx` may not change the build. Check include paths and Flash capacity before adding large generated C assets. If assets already display correctly, focus on the compression/runtime issue rather than repeating basic display bring-up.
