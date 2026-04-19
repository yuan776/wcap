# wcap Deep Dive: Architecture and Implementation

This document provides an in-depth technical explanation of how wcap works, its encoding architecture, and how it compares to FFmpeg-based screen recorders.

## Table of Contents
- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Screen Capture: Windows.Graphics.Capture API](#screen-capture-windowsgraphicscapture-api)
- [Video Encoding: Media Foundation vs FFmpeg](#video-encoding-media-foundation-vs-ffmpeg)
- [GPU-Accelerated Processing Pipeline](#gpu-accelerated-processing-pipeline)
- [Audio Capture and Encoding](#audio-capture-and-encoding)
- [SDK Compatibility: IGraphicsCaptureSession5/6 Fallback](#sdk-compatibility-igraphicscapturesession56-fallback)

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

## Summary

wcap is designed around these principles:

1. **Native Windows APIs**: Media Foundation, Windows.Graphics.Capture, WASAPI
2. **GPU-first**: All processing stays on GPU via D3D11 compute shaders
3. **Zero dependencies**: Single executable, no DLLs
4. **Async encoding**: Non-blocking frame submission
5. **SDK resilience**: Fallback definitions for newer APIs

This makes wcap extremely lightweight (~70KB) while still supporting modern features like hardware-accelerated H.265/AV1 encoding, HDR capture, and application-local audio.
