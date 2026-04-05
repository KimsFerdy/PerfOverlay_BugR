# PerformanceViewer — Plugin Documentation
**Version 1.1** · Unreal Engine 5.3+ · © kims ferdy 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Architecture](#architecture)
4. [Configuration](#configuration)
5. [UPerfStatsCollector](#uperfstatscollector)
   - [FPerfFrameData Reference](#fperfframedata-reference)
   - [Enums](#enums)
   - [History Arrays](#history-arrays)
6. [APerformanceViewerHUD](#aperformanceviewerhud)
7. [UPerfOverlayWidget](#uperfoverlaywidget)
   - [Auto-Bound Widget Names](#auto-bound-widget-names)
   - [Blueprint Events](#blueprint-events)
   - [Static Helper Functions](#static-helper-functions)
8. [UPerfSparklineWidget](#uperfsparklinewidget)
9. [Platform Notes](#platform-notes)
10. [Shipping Checklist](#shipping-checklist)
11. [Changelog](#changelog)

---

## Overview

PerformanceViewer is a runtime performance overlay plugin for Unreal Engine 5. It collects FPS, thread times, memory, GPU info, and platform stats every tick and surfaces them to a fully customisable UMG overlay — no Blueprint logic required for data collection.

**Key design principles:**
- Zero crashes on any platform. Every stat that cannot be read returns a safe sentinel value (`0.f` or `-1`) and the overlay shows `N/A` instead of a misleading zero.
- All bindings are optional. Missing UMG widgets are silently skipped.
- Works in Editor, Development, and Shipping builds. Stats requiring `bAllowStatsInShipping` degrade gracefully instead of crashing.
- No direct platform headers (`Android/AndroidMisc.h`, `IOS/IOSPlatformMisc.h`) — uses only the HAL layer so NDK cross-compilation in packaged plugin builds works out of the box.

---

## Quick Start

### 1 — Add the HUD class

In your **GameMode** Blueprint or C++, set the HUD class to `APerformanceViewerHUD` (or a Blueprint child of it).

### 2 — Create the overlay widget Blueprint

1. Create a new **Widget Blueprint**.
2. Reparent it to `UPerfOverlayWidget` *(File → Reparent Blueprint)*.
3. Add any of the [named widgets](#auto-bound-widget-names) to your layout. Naming is the only requirement — C++ fills them automatically every tick.

### 3 — Assign the widget class

In your HUD Blueprint defaults (or `APerformanceViewerHUD` subclass), set **Overlay Widget Class** to the Widget Blueprint you just created.

### 4 — Set Target FPS

Set **Target FPS** on the HUD (default `60`). This drives FPS health colour coding and sparkline threshold lines.

That's it. Hit Play.

---

## Architecture

```
APerformanceViewerHUD  (Tick every frame)
        │
        ├─ UPerfStatsCollector::Tick()
        │       Collects all stats → writes FPerfFrameData
        │
        └─ UPerfOverlayWidget::UpdateStats()
                │
                ├─ AutoUpdateTextBlocks()    ← populates all Text_ bindings
                ├─ AutoUpdateProgressBars()  ← populates all Bar_ bindings
                ├─ AutoUpdateSparklines()    ← populates all Sparkline_ bindings
                │
                └─ Fires BP_On* events      ← your Blueprint logic hooks in here
```

The HUD owns both objects. Stats flow one way: Collector → Overlay. You never need to wire anything in Blueprint for the auto-bindings to work.

---

## Configuration

### HUD Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `TargetFPS` | `float` | `60` | Used for FPS health thresholds and sparkline target lines. Clamped 1–360. |
| `OverlayWidgetClass` | `TSubclassOf<UPerfOverlayWidget>` | — | Assign your UMG Blueprint here. Must inherit `UPerfOverlayWidget`. |

### Collector Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `TargetFPS` | `float` | `60` | Synced from HUD each tick. |
| `HistorySize` | `int32` | `80` | Number of frames kept in each history buffer. Clamped 10–500. |

### DefaultEngine.ini — Recommended additions

```ini
[Core.System]
; Required for thread times (Game / Render / GPU) in packaged builds
bAllowStatsInShipping=true

[/Script/Engine.Engine]
; Activates stat unit system on startup so GetStatUnitData() returns data immediately
bEnableStatUnitOnStartup=true
```

Without `bAllowStatsInShipping`, thread times fall back to FApp timing (Game thread only) and show `N/A` for Render Thread and GPU.

---

## UPerfStatsCollector

Collects all performance data each tick. Created and owned by `APerformanceViewerHUD`. You can also create it manually in Blueprint with `NewObject` and call `Tick` yourself if you don't want to use the HUD.

```cpp
UFUNCTION(BlueprintCallable)
void Tick(float DeltaTime);

UFUNCTION(BlueprintPure)
const FPerfFrameData& GetCurrentFrame() const;
```

### FPerfFrameData Reference

All fields are `BlueprintReadOnly`. `0.f` / `-1` / `"Unknown"` mean the stat is not available on this platform or build config — never a real zero.

#### Frame

| Field | Type | Notes |
|---|---|---|
| `FPS` | `float` | Exponentially smoothed (α = 0.1). Never raw. |
| `FrameTimeMS` | `float` | Raw delta time in milliseconds. |

#### Thread Times

Requires `bAllowStatsInShipping=true` in Shipping builds. Falls back to FApp timing otherwise — `RenderThreadTimeMS` and `GPUTimeMS` will be `0.f` (shown as `N/A`).

| Field | Type | Notes |
|---|---|---|
| `GameThreadTimeMS` | `float` | Game thread frame time. Always available (FApp fallback). |
| `RenderThreadTimeMS` | `float` | `0.f` without stats enabled in Shipping. |
| `GPUTimeMS` | `float` | `0.f` without stats enabled in Shipping. |

#### RAM

| Field | Type | Notes |
|---|---|---|
| `RAMUsedMB` | `float` | Physical memory in use. |
| `RAMTotalMB` | `float` | Total physical memory on device. |
| `RAMUsedVirtualMB` | `float` | Virtual memory in use. |

#### VRAM / Texture Pool

| Field | Type | Notes |
|---|---|---|
| `VRAMUsedMB` | `float` | **Desktop:** RHI allocated texture bytes. **Mobile:** texture pool consumed. `0` = N/A. |
| `VRAMTotalMB` | `float` | **Desktop:** dedicated video memory. **Mobile:** `r.TextureStreamingPoolSize` budget. `0` = N/A. |
| `VRAMAvailableMB` | `float` | Most reliable cross-platform value. `0` = N/A. |

> **Mobile note:** Raw GPU VRAM is not exposed on Android or iOS. `VRAMTotalMB` reflects Unreal's managed texture streaming pool — this is the value that actually drives texture quality, streaming decisions, and OOM kills, making it more actionable than raw VRAM would be.

#### GPU Info

Cached on the first tick. Never changes at runtime.

| Field | Type | Notes |
|---|---|---|
| `GPUName` | `FString` | e.g. `"NVIDIA GeForce RTX 4080"`, `"Adreno (TM) 740"` |
| `GPUVendor` | `FString` | One of: `NVIDIA`, `AMD`, `Intel`, `Qualcomm`, `ARM`, `Apple`, `Imagination`, `Unknown` |

#### Rendering

| Field | Type | Notes |
|---|---|---|
| `DrawCalls` | `int32` | Always `-1` in v1.1 (not reliably available in Shipping without engine modification). |
| `GraphicsAPI` | `FString` | One of: `DirectX 11`, `DirectX 12`, `Vulkan`, `OpenGL`, `Metal`, `Unknown`. Cached after first tick. |

#### Platform

| Field | Type | Notes |
|---|---|---|
| `BatteryLevel` | `int32` | `0–100` on mobile. `-1` on desktop (N/A). |
| `TemperatureLevel` | `int32` | Android only via `FGenericPlatformMisc`. `-1` on iOS and desktop. |

#### Derived

| Field | Type | Notes |
|---|---|---|
| `Bottleneck` | `EPerfBottleneck` | `CPU`, `GPU`, or `Balanced`. Uses 10% hysteresis to avoid per-frame flipping. |
| `FPSHealth` | `EFPSHealth` | `Good` (≥ target), `Warning` (≥ 50% of target), `Critical` (< 50%). |
| `MemoryPressure` | `EMemoryPressure` | `Safe` (< 65%), `Warning` (65–85%), `Critical` (> 85%) of pool consumed. |
| `MemoryPressureRatio` | `float` | `0.0–1.0`. Direct input for progress bars. |

### Enums

```
EPerfBottleneck  →  CPU | GPU | Balanced
EFPSHealth       →  Good | Warning | Critical
EMemoryPressure  →  Safe | Warning | Critical
```

All three are `BlueprintType` and available everywhere in Blueprint.

### History Arrays

Each array holds the last `HistorySize` frames (default 80). Used by sparklines but also accessible directly in Blueprint.

| Function | Returns |
|---|---|
| `GetFPSHistory()` | `TArray<float>` — smoothed FPS values |
| `GetFrameTimeHistory()` | `TArray<float>` — frame times in ms |
| `GetGPUTimeHistory()` | `TArray<float>` — GPU times in ms |
| `GetRAMHistory()` | `TArray<float>` — RAM used in MB |
| `GetVRAMHistory()` | `TArray<float>` — VRAM used in MB (or pool used on mobile) |

---

## APerformanceViewerHUD

Extends `AHUD`. Owns the collector and overlay widget. Ticks both every frame.

```cpp
UFUNCTION(BlueprintCallable) void ShowOverlay();
UFUNCTION(BlueprintCallable) void HideOverlay();
UFUNCTION(BlueprintCallable) void ToggleOverlay();
UFUNCTION(BlueprintPure)     bool IsOverlayVisible() const;
UFUNCTION(BlueprintPure)     UPerfStatsCollector* GetStatsCollector() const;
```

`GetStatsCollector()` gives Blueprint direct access to `FPerfFrameData` and history arrays if you want to build custom widgets outside the overlay system.

---

## UPerfOverlayWidget

Base class for your UMG overlay Blueprint. Reparent your Widget Blueprint to this class. All data population is automatic — you only need to name your widgets correctly.

### Auto-Bound Widget Names

Name your UMG widgets **exactly** as shown. C++ finds them by name at construction time (`BindWidget`). All are optional — missing widgets are skipped silently and never crash.

> **Why exact names?** This is how `meta=(BindWidget)` works in UMG. The engine matches C++ variable names to widget names in the hierarchy at construction. There is no alternative lookup mechanism.

#### Text Blocks — FPS

| Widget Name | Content | Notes |
|---|---|---|
| `Text_FPS` | `"60 FPS"` | Coloured by `EFPSHealth` |
| `Text_FPS_Value` | `"60"` | Number only, also coloured |
| `Text_FrameTime` | `"16.67 ms"` | |
| `Text_FrameTime_MS` | `"16.67"` | Number only |

#### Text Blocks — Thread Times

| Widget Name | Content | Notes |
|---|---|---|
| `Text_GameThread` | `"GT  12.34 ms"` | Shows `N/A` if unavailable |
| `Text_GameThread_MS` | `"12.34"` | Number only |
| `Text_RenderThread` | `"RT  12.34 ms"` | |
| `Text_RenderThread_MS` | `"12.34"` | |
| `Text_GPU` | `"GPU 12.34 ms"` | |
| `Text_GPU_MS` | `"12.34"` | |
| `Text_Bottleneck` | `"BOTTLENECK: GPU"` | Coloured by bottleneck type |

#### Text Blocks — VRAM

| Widget Name | Content | Notes |
|---|---|---|
| `Text_VRAM` | `"VRAM  512 / 4096 MB"` | Falls back to `"512 MB free"` or `"N/A"` |
| `Text_VRAM_Used` | `"512 MB"` | `"N/A"` if unavailable |
| `Text_VRAM_Total` | `"4096 MB"` | `"N/A"` if unavailable |
| `Text_VRAM_Free` | `"3584 MB"` | Available headroom |

#### Text Blocks — RAM

| Widget Name | Content | Notes |
|---|---|---|
| `Text_RAM` | `"RAM  1024 / 4096 MB"` | |
| `Text_RAM_Used` | `"1024 MB"` | |
| `Text_RAM_Total` | `"4096 MB"` | |
| `Text_RAM_Free` | `"3072 MB"` | |

#### Text Blocks — Memory Pressure

| Widget Name | Content | Notes |
|---|---|---|
| `Text_MemoryPressure` | `"MEM PRESSURE: WARNING (73%)"` | Coloured by pressure level |
| `Text_MemoryPressure_Pct` | `"73%"` | Number only, also coloured |

#### Text Blocks — GPU & Rendering

| Widget Name | Content |
|---|---|
| `Text_GPUName` | `"NVIDIA GeForce RTX 4080"` |
| `Text_GPUVendor` | `"NVIDIA"` |
| `Text_GraphicsAPI` | `"DirectX 12"` |
| `Text_DrawCalls` | `"Draws  N/A"` (always N/A in v1.1) |

#### Text Blocks — Platform

| Widget Name | Content | Notes |
|---|---|---|
| `Text_Battery` | `"Battery  87%"` | `"Battery  N/A"` on desktop |
| `Text_Temperature` | `"Temp  Level 2"` | `"Temp  N/A"` on iOS/desktop |

#### Progress Bars

| Widget Name | Range | Colour logic |
|---|---|---|
| `Bar_VRAM` | `0–1` (used / total) | Green → Yellow → Red at 70% / 90% |
| `Bar_RAM` | `0–1` (used / total) | Green → Yellow → Red at 70% / 90% |
| `Bar_MemoryPressure` | `0–1` (pressure ratio) | Green → Yellow → Red at 65% / 85% |
| `Bar_Battery` | `0–1` | Red → Yellow → Green (inverted, low = bad) |

#### Sparklines

Add a **Perf Sparkline** widget to your layout and name it as below. The sparkline widget is found in the UMG palette under *PerformanceViewer*.

| Widget Name | Data source | Threshold line |
|---|---|---|
| `Sparkline_FPS` | FPS history | Target FPS |
| `Sparkline_FrameTime` | Frame time history | 1000 / TargetFPS ms |
| `Sparkline_GPU` | GPU time history | None |
| `Sparkline_RAM` | RAM used history | None |
| `Sparkline_VRAM` | VRAM used history | None |

### Blueprint Events

Override these in your UMG Blueprint for custom logic. All are called every tick automatically — you do not need to call them yourself.

```
BP_OnStatsUpdated(Frame: FPerfFrameData)
    Full frame data. Use this if you want everything in one place.

BP_OnFPSUpdated(FPS, FrameTimeMS, Health: EFPSHealth, HealthColor: LinearColor)

BP_OnThreadTimesUpdated(GameThread, RenderThread, GPU: float, Bottleneck: EPerfBottleneck)

BP_OnMemoryUpdated(VRAMUsedMB, VRAMTotalMB, VRAMAvailableMB, RAMUsedMB: float,
                   MemPressure: EMemoryPressure, PressureRatio: float)

BP_OnGraphicsInfo(GraphicsAPI: string, DrawCalls: int,
                  GPUName: string, GPUVendor: string)

BP_OnPlatformInfo(BatteryLevel: int, TemperatureLevel: int)
```

### Static Helper Functions

Available in Blueprint anywhere (no instance needed).

| Function | Input | Returns |
|---|---|---|
| `GetHealthColor` | `EFPSHealth` | `FLinearColor` — green / yellow / red |
| `GetBottleneckColor` | `EPerfBottleneck` | `FLinearColor` — green / orange / blue |
| `GetBottleneckLabel` | `EPerfBottleneck` | `FString` — `"CPU"` / `"GPU"` / `"BALANCED"` |
| `GetMemoryPressureColor` | `EMemoryPressure` | `FLinearColor` — green / yellow / red |
| `GetMemoryPressureLabel` | `EMemoryPressure` | `FString` — `"SAFE"` / `"WARNING"` / `"CRITICAL"` |

---

## UPerfSparklineWidget

A custom Slate-backed UMG widget that renders a filled area sparkline with an optional threshold line. Found in the UMG palette under **PerformanceViewer**.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `ColorGood` | `FLinearColor` | Green | Line / fill colour when value is below warning ratio. |
| `ColorWarning` | `FLinearColor` | Yellow | Line / fill colour when value is between warning and critical ratio. |
| `ColorCritical` | `FLinearColor` | Red | Line / fill colour when value exceeds critical ratio. |
| `ThresholdColor` | `FLinearColor` | White 35% α | Colour of the horizontal threshold line. |
| `FillAlpha` | `float` | `0.25` | Opacity of the filled area under the line. |
| `LineThickness` | `float` | `1.5` | Sparkline stroke width in pixels. |
| `ThresholdValue` | `float` | `0` | Y value at which the threshold line is drawn. `0` = hidden. Set automatically by the overlay for FPS and FrameTime sparklines. |
| `FixedMaxValue` | `float` | `0` | Pin the Y axis maximum. `0` = auto-scale to data. |
| `FixedMinValue` | `float` | `0` | Pin the Y axis minimum. |
| `WarningRatio` | `float` | `0.75` | Fraction of the Y range at which colour switches to Warning. |
| `CriticalRatio` | `float` | `0.90` | Fraction of the Y range at which colour switches to Critical. |

### Blueprint function

```
SetData(InData: TArray<float>)
    Pushes a new data array into the sparkline and triggers a repaint.
    Called automatically by the overlay — only needed if driving the widget manually.
```

---

## Platform Notes

| Feature | Windows | Mac | Linux | Android | iOS |
|---|---|---|---|---|---|
| FPS & Frame Time | ✅ | ✅ | ✅ | ✅ | ✅ |
| Game Thread Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| Render Thread Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| GPU Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| RAM Used / Total | ✅ | ✅ | ✅ | ✅ | ✅ |
| VRAM Used / Total | ✅ | ✅ | ✅ | Pool only | Pool only |
| VRAM Available | ✅ | ✅ | ✅ | ✅ | ✅ |
| GPU Name | ✅ | ✅ | ✅ | ✅ | ✅ |
| Graphics API | ✅ | ✅ | ✅ | ✅ | ✅ |
| Battery | ❌ N/A | ❌ N/A | ❌ N/A | ✅ | ✅ |
| Temperature | ❌ N/A | ❌ N/A | ❌ N/A | ✅ | ❌ N/A |
| Draw Calls | ❌ N/A | ❌ N/A | ❌ N/A | ❌ N/A | ❌ N/A |

\* Requires `bAllowStatsInShipping=true` in Shipping builds. Degrades to FApp fallback otherwise.

---

## Shipping Checklist

Before packaging a Shipping build with thread time stats enabled:

```ini
; DefaultEngine.ini
[Core.System]
bAllowStatsInShipping=true

[/Script/Engine.Engine]
bEnableStatUnitOnStartup=true
```

Without these, the overlay still works — it just shows `N/A` for Render Thread and GPU time instead of real values. FPS, RAM, VRAM, GPU name, battery, and temperature all work in Shipping with no ini changes.

---

## Changelog

### v1.1
- **New:** `VRAMAvailableMB` field — cross-platform texture pool headroom
- **New:** `GPUName` and `GPUVendor` — cached on first tick, displayed in overlay
- **New:** `EMemoryPressure` enum + `MemoryPressureRatio` — crash-predictive mobile metric
- **New:** `GetVRAMHistory()` + `Sparkline_VRAM` binding
- **New:** Split text bindings — `Text_VRAM_Used/Total/Free`, `Text_RAM_Used/Total/Free`, `Text_FPS_Value`, `Text_FrameTime_MS`, `Text_GameThread_MS`, `Text_RenderThread_MS`, `Text_GPU_MS`, `Text_MemoryPressure_Pct`
- **New:** `Bar_MemoryPressure` progress bar binding
- **New:** `BP_OnMemoryUpdated` extended with `VRAMAvailableMB`, `EMemoryPressure`, `PressureRatio`
- **New:** `BP_OnGraphicsInfo` extended with `GPUName` and `GPUVendor`
- **Fix:** VRAM now uses `RHIGetTextureMemoryStats()` — replaces previously non-compiling `FRHIMemoryStats` / `GetMemoryStats()` path
- **Fix:** Removed `SetStatEnabled` call (protected member) — replaced with ini/console guidance
- **Fix:** Removed direct `Android/AndroidMisc.h` and `IOS/IOSPlatformMisc.h` includes — fixes NDK cross-compilation fatal error in packaged plugin builds
- **Fix:** Battery and GPU name now routed through `FPlatformMisc` HAL — no platform header required
- **Fix:** `GPUFrameTime[0]` (correct UE5 field) replaces `GPUTime` (UE4 field, doesn't exist in UE5)
- **Fix:** Bottleneck detection uses 10% hysteresis band to prevent per-frame state flipping
- **Fix:** RAM progress bar uses real `RAMTotalMB` instead of hardcoded 8192 MB

### v1.0
- Initial release: FPS, frame time, thread times, RAM, basic VRAM, draw calls, graphics API, battery, temperature, sparklines
