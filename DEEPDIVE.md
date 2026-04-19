# wcap Deep Dive: Architecture and Implementation

This document provides an in-depth technical explanation of how wcap works, its encoding architecture, and how it compares to FFmpeg-based screen recorders.

## Table of Contents
- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Screen Capture: Windows.Graphics.Capture API](#screen-capture-windowsgraphicscapture-api)
- [Video Encoding: Media Foundation vs FFmpeg](#video-encoding-media-foundation-vs-ffmpeg)
- [GPU-Accelerated Processing Pipeline](#gpu-accelerated-processing-pipeline)
- [Shaders: What They Are and How They Work](#shaders-what-they-are-and-how-they-work)
- [Audio Capture and Encoding](#audio-capture-and-encoding)
- [SDK Compatibility: IGraphicsCaptureSession5/6 Fallback](#sdk-compatibility-igraphicscapturesession56-fallback)
- [Build Process](#build-process)

---

## Overview

wcap is a lightweight Windows screen recorder that uses modern Windows APIs for maximum performance:

| Component | API Used |
|-----------|----------|
| Screen Capture | Windows.Graphics.Capture (WinRT) |
| Video Encoding | Media Foundation (MF) |
| Audio Capture | WASAPI Loopback |
| GPU Processing | Direct3D 11 Compute Shaders |

**Supported Codecs:**
- Video: H.264, H.265 (HEVC), AV1
- Audio: AAC, FLAC
- Container: MP4

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SCREEN CAPTURE LAYER                            │
│                  (Windows.Graphics.Capture API)                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Window/Monitor Selection → Direct3D11CaptureFramePool         │  │
│  │ Frame callbacks on compositor thread → D3D11 Texture (BGRA)   │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                  ┌───────────────┴────────────────┐
                  ↓                                ↓
      ┌──────────────────────┐       ┌─────────────────────────┐
      │    VIDEO PATH        │       │      AUDIO PATH         │
      │  (GPU Accelerated)   │       │   (WASAPI Loopback)     │
      └──────────────────────┘       └─────────────────────────┘
                  │                              │
      ┌───────────┴───────────┐                  │
      │                       │                  │
  ┌───▼────┐            ┌─────▼─────┐            │
  │ Copy   │   D3D11    │  Resize   │   D3D11   │
  │ + Crop │──Compute──→│  2-Pass   │──Compute──→│
  └────────┘   Shader   │ (H + V)   │   Shader  │
                        └─────┬─────┘            │
                              │                  │
                        ┌─────▼──────┐           │
                        │YUV Convert │   D3D11  │
                        │RGB → NV12  │──Compute──→
                        │   /P010    │   Shader │
                        └─────┬──────┘           │
                              │                  │
                   ┌──────────┴──────────┐       │
                   │                     │       │
                   ↓                     ↓       ↓
        ┌──────────────────┐   ┌──────────────────────────┐
        │ Video Sample     │   │ Audio Sample             │
        │ Ring Buffer (8)  │   │ Ring Buffer (16)         │
        │ NV12/P010 YUV    │   │ 16-bit PCM               │
        └────────┬─────────┘   └───────────┬──────────────┘
                 │                         │
                 └────────────┬────────────┘
                              │
                   ┌──────────▼──────────┐
                   │   IMFSinkWriter     │
                   │  (Async Encoding)   │
                   │ ┌────────────────┐  │
                   │ │ Video MFT      │  │  ← Hardware (GPU) or Software
                   │ │ H264/HEVC/AV1  │  │
                   │ └────────────────┘  │
                   │ ┌────────────────┐  │
                   │ │ Audio MFT      │  │  ← Software only
                   │ │ AAC/FLAC       │  │
                   │ └────────────────┘  │
                   │ ┌────────────────┐  │
                   │ │ MP4 Muxer      │  │
                   │ └────────────────┘  │
                   └──────────┬──────────┘
                              │
                       ┌──────▼──────┐
                       │  MP4 File   │
                       └─────────────┘
```

---

## Screen Capture: Windows.Graphics.Capture API

wcap uses the **Windows.Graphics.Capture** API (introduced in Windows 10 1903) instead of older methods like BitBlt or Desktop Duplication.

### Key Components

| Interface | Purpose |
|-----------|---------|
| `IGraphicsCaptureItem` | Represents the window or monitor being captured |
| `IDirect3D11CaptureFramePool` | Pool of D3D11 textures for captured frames |
| `IGraphicsCaptureSession` | Controls the capture session |

### Session Interface Versions

Different Windows versions expose different session capabilities:

| Interface | Windows Version | Features |
|-----------|-----------------|----------|
| `IGraphicsCaptureSession` | 1903 (19H1) | Basic capture |
| `IGraphicsCaptureSession2` | 2004 (20H1) | Hide mouse cursor |
| `IGraphicsCaptureSession3` | Win 11 | Hide yellow recording border |
| `IGraphicsCaptureSession5` | Win 11 24H2+ | `MinUpdateInterval` for framerate control |
| `IGraphicsCaptureSession6` | Win 11 24H2+ | `IncludeSecondaryWindows` for popups |

### Why Windows.Graphics.Capture?

1. **GPU-native**: Frames are delivered as D3D11 textures, no CPU copy needed
2. **Efficient**: Uses compositor's frame pool, minimal overhead
3. **Modern features**: Border hiding, cursor control, app-local audio
4. **DRM-aware**: Respects display affinity (protected content)

---

## Video Encoding: Media Foundation vs FFmpeg

### wcap's Approach: Media Foundation

wcap uses **Microsoft Media Foundation (MF)**, the native Windows multimedia framework:

```c
// wcap creates a sink writer that handles encoding + muxing
IMFSinkWriter* SinkWriter;
MFCreateSinkWriterFromURL(FileName, NULL, Attributes, &SinkWriter);

// Submit frames asynchronously
IMFSinkWriter_WriteSample(SinkWriter, VideoStreamIndex, Sample);
```

### FFmpeg-based Recorders (OBS, ShareX, etc.)

Most open-source recorders use **FFmpeg's libavcodec**:

```c
// FFmpeg approach
AVCodecContext* codec_ctx = avcodec_alloc_context3(codec);
avcodec_open2(codec_ctx, codec, NULL);
avcodec_send_frame(codec_ctx, frame);
avcodec_receive_packet(codec_ctx, packet);
```

### Comparison Table

| Aspect | Media Foundation (wcap) | FFmpeg (OBS, etc.) |
|--------|-------------------------|-------------------|
| **Platform** | Windows-only | Cross-platform |
| **Hardware Encoding** | Native OS integration | Via hwaccel APIs |
| **Codec Support** | H.264, H.265, AV1 | 100+ codecs |
| **Container Support** | MP4, MKV (limited) | 100+ formats |
| **Dependencies** | None (OS built-in) | Large library (~50MB) |
| **Binary Size** | ~70 KB | Several MB |
| **Latency** | Lower (OS-integrated) | Variable |
| **Customization** | Limited | Highly configurable |

### How Media Foundation Hardware Encoding Works

```
┌──────────────────────────────────────────────────────────────┐
│                    Media Foundation                          │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                 IMFSinkWriter                           │ │
│  │  ┌───────────────────────────────────────────────────┐  │ │
│  │  │              MFT (Media Foundation Transform)     │  │ │
│  │  │  ┌──────────────────────────────────────────────┐ │  │ │
│  │  │  │  Hardware Encoder (if available)             │ │  │ │
│  │  │  │  - NVIDIA NVENC (nvidia_h264, nvidia_hevc)   │ │  │ │
│  │  │  │  - Intel QuickSync (qsv_h264, qsv_hevc)      │ │  │ │
│  │  │  │  - AMD VCE (amd_h264, amd_hevc)              │ │  │ │
│  │  │  └──────────────────────────────────────────────┘ │  │ │
│  │  │                      OR                           │  │ │
│  │  │  ┌──────────────────────────────────────────────┐ │  │ │
│  │  │  │  Software Encoder (fallback)                 │ │  │ │
│  │  │  │  - Microsoft H.264 Software Encoder          │ │  │ │
│  │  │  └──────────────────────────────────────────────┘ │  │ │
│  │  └───────────────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

wcap queries available encoders and prefers hardware:

```c
// From wcap_encoder.h - Hardware encoder detection
MFTEnumEx(MFT_CATEGORY_VIDEO_ENCODER,
          MFT_ENUM_FLAG_HARDWARE | MFT_ENUM_FLAG_SORTANDFILTER,
          &InputType, &OutputType, &Activates, &Count);
```

### Why Media Foundation for wcap?

1. **Zero dependencies**: Ships with Windows, no DLLs needed
2. **Tiny binary**: ~70KB vs several MB for FFmpeg-based apps
3. **Native GPU integration**: Direct D3D11 texture input
4. **Async encoding**: Non-blocking via `IMFAsyncCallback`
5. **OS-optimized**: Microsoft-tuned for Windows performance

### Trade-offs

| Media Foundation Advantage | FFmpeg Advantage |
|---------------------------|------------------|
| Smaller binary | More codecs (VP9, etc.) |
| No DLL hell | Cross-platform |
| Tighter OS integration | More customization |
| Simpler deployment | Better documentation |

---

## GPU-Accelerated Processing Pipeline

All video processing happens on the GPU using D3D11 compute shaders.

### Stage 1: Texture Resize (wcap_tex_resize.h)

Two-pass separable resize for quality:

```
Pass 1 (Horizontal): Input[W×H] → Temp[W'×H]
Pass 2 (Vertical):   Temp[W'×H] → Output[W'×H']
```

**Shaders:**
- `ResizePassH.hlsl` - Horizontal downsampling
- `ResizePassV.hlsl` - Vertical downsampling
- `ResizeLinearPassH/V.hlsl` - Gamma-correct linear space versions

### Stage 2: Color Conversion (wcap_yuv_convert.h)

RGB (BGRA) to YUV conversion:

```
Input:  BGRA (8-bit per channel)
Output: NV12 (8-bit Y + UV) or P010 (10-bit Y + UV)
```

**Color Space Matrices:**
```c
// BT.709 (HD content, width ≥ 1280)
Y  =  0.2126×R + 0.7152×G + 0.0722×B
Cb = -0.1146×R - 0.3854×G + 0.5000×B + 128
Cr =  0.5000×R - 0.4542×G - 0.0458×B + 128

// BT.601 (SD content)
Y  =  0.299×R + 0.587×G + 0.114×B
Cb = -0.169×R - 0.331×G + 0.500×B + 128
Cr =  0.500×R - 0.419×G - 0.081×B + 128
```

### Why GPU Processing?

| CPU Approach | GPU Approach (wcap) |
|--------------|---------------------|
| ReadPixels to CPU memory | Stay on GPU |
| CPU resize (slow) | Compute shader resize (fast) |
| CPU color convert | Compute shader convert |
| Upload to encoder | Zero-copy to encoder |

The frame never leaves the GPU until it's encoded.

---

## Shaders: What They Are and How They Work

### What is a Shader?

A **shader** is a small program that runs on the **GPU** instead of the CPU. In wcap, shaders perform image processing tasks like:

- **Resizing** captured frames (e.g., 4K → 1080p)
- **Color conversion** from RGB to YUV (required for video encoding)

```
┌─────────────────────────────────────────────────────────────────┐
│                         GPU                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Thousands of Shader Cores (parallel processing)        │   │
│  │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐      │   │
│  │  │ S │ │ S │ │ S │ │ S │ │ S │ │ S │ │ S │ │ S │ ...  │   │
│  │  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘      │   │
│  │  Each core processes one pixel simultaneously!          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Why Does wcap Need Shaders?

Without shaders (CPU processing):
```
4K frame (3840×2160) = 8.3 million pixels
Resize + Color convert on CPU: ~50ms per frame (only 20 fps possible!)
```

With shaders (GPU processing):
```
Same 4K frame processed by thousands of GPU cores in parallel
Resize + Color convert on GPU: ~1ms per frame (1000+ fps possible!)
```

### wcap's Shaders (wcap_shaders.hlsl)

| Shader | Purpose |
|--------|---------|
| `ResizePassH` | Horizontal downscaling |
| `ResizePassV` | Vertical downscaling |
| `ResizeLinearPassH/V` | Gamma-correct resize (better quality) |
| `ConvertSinglePass` | RGB → YUV conversion (fast) |
| `ConvertPass1` + `ConvertPass2` | RGB → YUV (higher quality) |

### Shader Source Code Example

The shaders are written in **HLSL** (High Level Shader Language), which is like C for GPUs:

```hlsl
// From wcap_shaders.hlsl - RGB to YUV conversion
[numthreads(16, 16, 1)]  // 16×16 = 256 threads per group
void ConvertSinglePass(uint3 OutputPos: SV_DispatchThreadID)
{
    // Each thread processes one 2×2 pixel block
    uint4 Pos4 = OutputPos.xyxy * 2 + uint4(0, 0, 1, 1);

    // Sample RGB colors
    float3 Color = ConvertIn.SampleLevel(LinearSampler, ColorPos, 0);

    // Convert to YUV using color matrix
    ConvertOutUV[OutputPos.xy] = RgbToUV(Color) * RANGE_UV + OFFSET_UV;
    ConvertOutY[Pos4.xy] = RgbToY(ConvertIn[Pos4.xy]) * RANGE_Y + OFFSET_Y;
    // ... process remaining pixels
}
```

### The Shader File Types

```
wcap_shaders.hlsl          ← Source code (human-readable)
       │
       ▼ (fxc.exe compiler)
       │
       ├── ResizePassH.dxbc    ← Compiled GPU bytecode (binary)
       ├── ResizePassH.asm     ← Disassembly (for debugging)
       ├── ResizePassH.dcs     ← Compressed bytecode
       └── ResizePassH.h       ← C header with byte array
```

| File Extension | What It Is |
|----------------|------------|
| `.hlsl` | **Source code** - High Level Shader Language (like C for GPUs) |
| `.dxbc` | **Compiled bytecode** - DirectX Bytecode for GPU execution |
| `.asm` | **Disassembly** - Human-readable GPU assembly for debugging |
| `.dcs` | **Compressed** - Smaller bytecode for embedding |
| `.h` | **C header** - Byte array to embed in executable |

### Why Are There Assembly (.asm) Files?

The `.asm` files are **NOT** CPU assembly - they're **GPU shader assembly**, a disassembly of the compiled shader bytecode for debugging purposes:

```asm
// From ResizePassH.asm - GPU instructions (NOT x86/ARM!)
cs_5_0                                          // Compute Shader 5.0
dcl_resource_texture2d (float,float,float,float) t0   // Input texture
dcl_uav_typed_texture2d (uint,uint,uint,uint) u0      // Output texture
dcl_thread_group 16, 16, 1                            // 16×16 threads per group

resinfo_indexable(texture2d) r0.xy, l(0), t0.xyzw     // Get texture size
div r0.xy, r1.xyxx, r0.xyxx                           // Calculate scale
mul r2.xy, r0.xyxx, r1.zwzz                           // Multiply coordinates
...
```

These files are generated by `fxc.exe /Fc` flag - they help developers see exactly what GPU instructions were generated from their HLSL code.

### How Shaders Are Compiled

The shader compilation process in `build.cmd`:

```batch
:fxc
fxc.exe /nologo %FXC% /WX /Ges /T cs_5_0 /E %1 /Fo shaders\%1.dxbc /Fc shaders\%1.asm wcap_shaders.hlsl
fxc.exe /nologo /compress /Vn %1ShaderBytes /Fo shaders\%1.dcs /Fh shaders\%1.h shaders\%1.dxbc
```

**Step 1**: Compile HLSL → DXBC + ASM
```
fxc.exe /T cs_5_0 /E ResizePassH /Fo ResizePassH.dxbc /Fc ResizePassH.asm wcap_shaders.hlsl
         ▲          ▲              ▲                    ▲
         │          │              │                    │
    Target:     Entry point    Output bytecode    Output assembly
    Compute      function
    Shader 5.0
```

**Step 2**: Compress and generate C header
```
fxc.exe /compress /Vn ResizePassHShaderBytes /Fh ResizePassH.h ResizePassH.dxbc
                   ▲                          ▲
                   │                          │
              Variable name            C header output
              in generated code
```

### How Shaders Are Embedded in the Executable

The `.h` files contain the compiled shader as a C byte array:

```c
// shaders/ResizePassH.h (auto-generated)
const BYTE ResizePassHShaderBytes[] =
{
     66,  83,  67,  68,   1,   0,    // Compressed DXBC bytecode
      1,   0,   0,   0,   0,   0,
     ...
};
```

At runtime, wcap creates the GPU shader from this embedded data:

```c
// wcap_tex_resize.h
#include "shaders/ResizePassH.h"

ID3D11ComputeShader* Shader;
ID3D11Device_CreateComputeShader(Device,
    ResizePassHShaderBytes,           // Embedded byte array
    sizeof(ResizePassHShaderBytes),
    NULL, &Shader);
```

### Shader Summary

| Question | Answer |
|----------|--------|
| **What are shaders?** | Small GPU programs for parallel image processing |
| **Why needed?** | GPU processes millions of pixels 50-100× faster than CPU |
| **Why .asm files?** | Debug output showing compiled GPU instructions (not CPU assembly) |
| **How compiled?** | `fxc.exe` (DirectX Shader Compiler) → bytecode → C header |
| **How included?** | Byte arrays in `.h` files, `#include`d into C code |

---

## Audio Capture and Encoding

### WASAPI Loopback

wcap captures system audio using WASAPI loopback:

```c
// Get default audio endpoint
IMMDeviceEnumerator_GetDefaultAudioEndpoint(enumerator, eRender, eConsole, &device);

// Initialize loopback capture
IAudioClient_Initialize(client, AUDCLNT_SHAREMODE_SHARED,
                        AUDCLNT_STREAMFLAGS_LOOPBACK, ...);
```

### Application-Local Audio (Windows 10 2004+)

For window capture, wcap can capture only that app's audio:

```c
// Activate audio capture for specific process
ActivateAudioInterfaceAsync(processId, IAudioClient, ...);
```

### Audio Pipeline

```
WASAPI (32-bit float, 48kHz)
    → Resample (if needed)
    → Convert to 16-bit PCM
    → AAC/FLAC encoder
    → MP4 muxer
```

---

## SDK Compatibility: IGraphicsCaptureSession5/6 Fallback

### The Problem

The Windows SDK versions installed on some development machines don't include the newest WinRT interface definitions. When compiling wcap, the compiler fails:

```
error C2065: '__x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5': undeclared identifier
error C2065: '__x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession6': undeclared identifier
```

These interfaces are defined in Windows SDK 10.0.22621.0+ but older SDKs don't have them.

### The Solution: Fallback Definitions

We added manual COM interface definitions in `wcap_screen_capture.h`:

```c
#ifndef ____x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5_FWD_DEFINED__
#define ____x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5_FWD_DEFINED__

typedef struct __x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5Vtbl
{
    // IUnknown
    HRESULT (STDMETHODCALLTYPE* QueryInterface)(...);
    ULONG (STDMETHODCALLTYPE* AddRef)(...);
    ULONG (STDMETHODCALLTYPE* Release)(...);
    // IInspectable
    HRESULT (STDMETHODCALLTYPE* GetIids)(...);
    HRESULT (STDMETHODCALLTYPE* GetRuntimeClassName)(...);
    HRESULT (STDMETHODCALLTYPE* GetTrustLevel)(...);
    // IGraphicsCaptureSession5
    HRESULT (STDMETHODCALLTYPE* get_MinUpdateInterval)(...);
    HRESULT (STDMETHODCALLTYPE* put_MinUpdateInterval)(...);
} __x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5Vtbl;

#endif
```

### Does the Fallback Lack Capability?

**No.** The fallback definitions are **functionally identical** to the SDK versions:

| Aspect | SDK Definition | Fallback Definition |
|--------|---------------|---------------------|
| GUID | Same | Same |
| Vtable layout | Same | Same |
| Method signatures | Same | Same |
| Runtime behavior | Same | Same |

The only difference is where the typedef comes from:
- **With new SDK**: From `<windows.graphics.capture.h>`
- **With old SDK**: From our inline definition

### Why Not Just Update the Windows SDK?

Several reasons:

1. **Build machine constraints**: CI/CD systems may use fixed SDK versions
2. **Corporate environments**: IT policies may restrict SDK updates
3. **Backward compatibility**: Developers on older Windows may have matching SDKs
4. **Simplicity**: Self-contained build without external dependencies
5. **wcap's philosophy**: Minimal dependencies, works out-of-the-box

### Runtime Behavior

The code handles missing OS features gracefully:

```c
__x_ABI_CWindows_CGraphics_CCapture_CIGraphicsCaptureSession5* Session5;
if (SUCCEEDED(QueryInterface(Session, &IID_IGraphicsCaptureSession5, &Session5)))
{
    // Only called if Windows supports this interface
    Session5->put_MinUpdateInterval(Duration);
    Session5->Release();
}
// If QueryInterface fails, we simply don't use the feature
```

**Result**: The app runs correctly on:
- Old Windows + old SDK (features disabled)
- Old Windows + new SDK (features disabled)
- New Windows + old SDK (features enabled via fallback)
- New Windows + new SDK (features enabled via SDK)

---

## Build Process

### Complete Build Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                        build.cmd                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Compile Shaders (fxc.exe)                              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  wcap_shaders.hlsl                                         │ │
│  │       │                                                    │ │
│  │       ▼                                                    │ │
│  │  fxc.exe (DirectX Shader Compiler)                         │ │
│  │       │                                                    │ │
│  │       ├──→ shaders/ResizePassH.dxbc  (GPU bytecode)        │ │
│  │       ├──→ shaders/ResizePassH.asm   (disassembly)         │ │
│  │       ├──→ shaders/ResizePassH.h     (C byte array)        │ │
│  │       │                                                    │ │
│  │       └──→ (repeat for all 7 shaders)                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Step 2: Compile Resources (rc.exe)                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  wcap.rc (icons, manifest) → wcap.res                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Step 3: Compile C Code (cl.exe)                                │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  wcap.c                                                    │ │
│  │    ├── #include "wcap_config.h"                            │ │
│  │    ├── #include "wcap_encoder.h"                           │ │
│  │    │      └── #include "wcap_tex_resize.h"                 │ │
│  │    │             └── #include "shaders/ResizePassH.h"  ◄── │ │
│  │    │             └── #include "shaders/ResizePassV.h"      │ │
│  │    │      └── #include "wcap_yuv_convert.h"                │ │
│  │    │             └── #include "shaders/ConvertPass1.h"     │ │
│  │    │             └── #include "shaders/ConvertPass2.h"     │ │
│  │    └── ...                                                 │ │
│  │                                                            │ │
│  │  cl.exe wcap.c wcap.res → wcap-x64.exe                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Build Commands

```batch
# Build for current architecture (auto-detect x64 or arm64)
build.cmd

# Build specifically for x64
build.cmd x64

# Build for ARM64
build.cmd arm64

# Debug build with sanitizers (x64 only)
build.cmd debug
```

### Requirements

- **Visual Studio** with C++ Desktop workload
- **Windows SDK** (10.0.19041.0 or newer recommended)
- **fxc.exe** - DirectX Shader Compiler (included with Windows SDK)

### What Gets Built

| File | Size | Description |
|------|------|-------------|
| `wcap-x64.exe` | ~70 KB | Main executable (x64) |
| `wcap-arm64.exe` | ~70 KB | Main executable (ARM64) |
| `shaders/*.h` | ~25 KB total | Embedded shader bytecode |
| `shaders/*.asm` | ~18 KB total | Debug disassembly (not in exe) |

---

## Summary

wcap is designed around these principles:

1. **Native Windows APIs**: Media Foundation, Windows.Graphics.Capture, WASAPI
2. **GPU-first**: All processing stays on GPU via D3D11 compute shaders
3. **Zero dependencies**: Single executable, no DLLs
4. **Async encoding**: Non-blocking frame submission
5. **SDK resilience**: Fallback definitions for newer APIs

This makes wcap extremely lightweight (~70KB) while still supporting modern features like hardware-accelerated H.265/AV1 encoding, HDR capture, and application-local audio.
