# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

Build the executable (requires Visual Studio with C++ Desktop workload):
```
build.cmd           # Build for host architecture (x64 or arm64)
build.cmd x64       # Build x64 release
build.cmd arm64     # Build ARM64 release
build.cmd debug     # Build debug with sanitizers (x64 only)
```

The build script compiles HLSL shaders, then compiles the C code with MSVC. Output is `wcap-x64.exe` or `wcap-arm64.exe`.

## Architecture Overview

wcap is a single-file Windows screen recorder using modern Windows APIs. The codebase uses a header-only implementation pattern where `.h` files contain both interface and implementation.

### Core Components

- **wcap.c** - Main entry point, window procedure, hotkey handling, tray icon, orchestrates capture/encode flow
- **wcap_screen_capture.h** - Windows.Graphics.Capture API wrapper for capturing windows/monitors
- **wcap_encoder.h** - Media Foundation video/audio encoding to MP4 (H264/HEVC/AV1 + AAC/FLAC)
- **wcap_audio_capture.h** - WASAPI loopback recording, supports application-local audio capture
- **wcap_config.h** - Settings dialog (Win32 UI), INI file persistence
- **wcap_tex_resize.h** - GPU texture downscaling with optional gamma-correct resize
- **wcap_yuv_convert.h** - GPU RGB to YUV conversion for video encoding
- **wcap_shaders.hlsl** - Compute shaders for resize and color conversion

### Key Design Patterns

- Uses C11 with `_Atomic` for thread-safe buffer management between capture and encoder
- All COM interfaces are used directly in C (no C++ wrappers)
- Hardware acceleration via D3D11 for capture, resize, color conversion, and encoding
- Asynchronous encoding with IMFAsyncCallback for non-blocking video/audio sample processing
- Configuration stored in INI file next to executable

### Recording Flow

1. Hotkey triggers capture (window/monitor/region)
2. `ScreenCapture_CreateFor*` initializes Windows.Graphics.Capture session
3. `Encoder_Start` sets up Media Foundation sink writer with video/audio streams
4. `OnCaptureFrame` callback receives D3D11 textures, passes to encoder
5. Audio captured on separate thread via WASAPI, submitted periodically via timer
6. `Encoder_Stop` finalizes MP4 file

### Windows Version Requirements

- Minimum: Windows 10 1903 (build 19H1) for Windows.Graphics.Capture
- Windows 10 2004 (20H1) for hiding cursor, application-local audio
- Windows 11 for hiding recording border, rounded corners, secondary window capture
