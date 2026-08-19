# PerformanceViewer — Plugin Documentation
**Version 1.3** · Unreal Engine 5.3+ · © kims ferdy 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Architecture](#architecture)
4. [Configuration](#configuration)
5. [UPerformanceViewerSubsystem](#uperformanceviewersubsystem)
6. [UPerfStatsCollector](#uperfstatscollector)
   - [FPerfFrameData Reference](#fperfframedata-reference)
   - [Enums](#enums)
   - [History Arrays](#history-arrays)
7. [APerformanceViewerHUD](#aperformanceviewerhud)
8. [UPerfOverlayWidget](#uperfoverlaywidget)
   - [Auto-Bound Widget Names](#auto-bound-widget-names)
   - [Blueprint Events](#blueprint-events)
   - [Static Helper Functions](#static-helper-functions)
9. [UPerformanceViewerBlueprintLibrary](#uperformanceviewerblueprintlibrary)
10. [UPerfSparklineWidget01 / 02](#uperfsparklinewidget01--02)
11. [Platform Notes](#platform-notes)
12. [Shipping Checklist](#shipping-checklist)
13. [Changelog](#changelog)

---

## Overview

PerformanceViewer is a runtime performance overlay plugin for Unreal Engine 5. It collects FPS, thread times, memory, GPU info, CPU info, and platform stats every tick and surfaces them to a fully customisable UMG overlay — no Blueprint logic required for data collection.

**Key design principles:**
- Zero crashes on any platform. Every stat that cannot be read returns a safe sentinel value (`0.f` or `-1`) and the overlay shows `N/A` instead of a misleading zero.
- All bindings are optional. Missing UMG widgets are silently skipped.
- Works in Editor, Development, and Shipping builds. Stats requiring `bAllowStatsInShipping` degrade gracefully instead of crashing.
- No direct platform headers (`Android/AndroidMisc.h`, `IOS/IOSPlatformMisc.h`) — uses only the HAL layer so NDK cross-compilation in packaged plugin builds works out of the box.
- Data collection is decoupled from the built-in overlay. `UPerformanceViewerSubsystem` owns the collector at the World level, so any custom UMG panel can read live stats without going through `APerformanceViewerHUD` at all.

---

## Quick Start

There are **two ways to capture data** — through the HUD, or directly from the widget itself. Neither is "more correct"; pick whichever fits how your project is already structured.

### Way 1 — through the HUD

1. In your **GameMode**, set the HUD class to `APerformanceViewerHUD` (or a Blueprint child of it).
2. Create a new **Widget Blueprint**, reparent it to `UPerfOverlayWidget` *(File → Reparent Blueprint)*.
3. Add any of the [named widgets](#auto-bound-widget-names) to your layout. Naming is the only requirement — C++ fills them automatically every tick.
4. In your HUD Blueprint defaults, set **Overlay Widget Class** to the Widget Blueprint you just created.
5. Set **Target FPS** on the HUD (default `60`). This drives FPS health colour coding and sparkline threshold lines.

Hit Play. That's it. The HUD owns the flow: it fetches the collector from `UPerformanceViewerSubsystem` and pushes a full `UpdateStats()` into the overlay widget every single frame — this is what the built-in overlay (`UPerfOverlayWidget`) uses, and it's the right choice when `APerformanceViewerHUD` is (or can be) your project's actual HUD class.

### Way 2 — directly from the widget itself

You don't need `APerformanceViewerHUD` at all — this is the right choice for a custom panel living somewhere `APerformanceViewerHUD` can't reach, or when your project already has its own HUD class. `UPerformanceViewerSubsystem` is what makes this work: it's a World Subsystem, so it exists and ticks the collector automatically the moment the world is running, with or without any HUD involved. From here you have two options, both entirely widget-side:

**Pull (bind once, UMG re-polls it every frame for you):**

1. Drop a `TextBlock` in your Widget Blueprint.
2. Click the **Bind** icon next to its `Text` property.
3. Pick a function from `UPerformanceViewerBlueprintLibrary` — e.g. `GetTextFPS`. Done. No manual formatting, no HUD reference, no casting.

**Push (fires on its own schedule, not every frame):**

1. In your widget's Construct graph, get `UPerformanceViewerSubsystem` and bind an event to its `OnStatsUpdated` delegate.
2. Wire your `SetText` (or any other) nodes off that event instead of a Bind or Event Tick.
3. It fires `DataCaptureRate` times/second (default 10) — see [OnStatsUpdated](#onstatsupdated--the-push-based-alternative-to-event-tick) for why that's cheaper than per-frame polling without losing accuracy.

---

## Architecture

```
UPerformanceViewerSubsystem  (UTickableWorldSubsystem — auto-created per World, self-ticking)
        │
        └─ UPerfStatsCollector::Tick()
                Collects all stats → writes FPerfFrameData

APerformanceViewerHUD  (Tick every frame)
        │
        ├─ fetches the collector from UPerformanceViewerSubsystem (does not own it)
        │
        └─ UPerfOverlayWidget::UpdateStats()
                │
                ├─ AutoUpdateTextBlocks()    ← populates all Text_ bindings
                │       (delegates to UPerformanceViewerBlueprintLibrary for every string)
                ├─ AutoUpdateProgressBars()  ← populates all Bar_ bindings
                ├─ AutoUpdateSparklines()    ← populates all Sparkline_ bindings
                │
                └─ Fires BP_On* events      ← your Blueprint logic hooks in here

Any other UMG widget
        │
        └─ GetWorld()->GetSubsystem<UPerformanceViewerSubsystem>()->GetStatsCollector()
                or, simpler: bind straight to UPerformanceViewerBlueprintLibrary functions
```

Stats flow one way: Subsystem → Collector → (Overlay | any custom widget). You never need to wire anything in Blueprint for the auto-bindings or the library functions to work.

---

## Configuration

### Subsystem Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `TargetFPS` | `float` | `60` | Used for FPS health thresholds and sparkline target lines. Clamped 1–360. |

### HUD Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `TargetFPS` | `float` | `60` | Forwarded into the subsystem's `TargetFPS` each tick — kept on the HUD too so existing placed HUD instances configure the same way as before. |
| `OverlayWidgetClass` | `TSubclassOf<UPerfOverlayWidget>` | — | Assign your UMG Blueprint here. Must inherit `UPerfOverlayWidget`. |

### Collector Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `TargetFPS` | `float` | `60` | Synced from the subsystem each tick. |
| `HistorySize` | `int32` | `80` | Number of frames kept in each history buffer. Clamped 10–500. |
| `HitchMultiplier` | `float` | `2.0` | A frame counts as a hitch once `FrameTimeMS` exceeds `(1000/TargetFPS) * HitchMultiplier`. Clamped 1.1–10.0. |
| `HitchWindowSeconds` | `float` | `5.0` | Rolling window `RecentHitchCount` counts within. Clamped 1–60. |

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

## UPerformanceViewerSubsystem

A `UTickableWorldSubsystem` — auto-created per World, ticks itself, owns the one `UPerfStatsCollector` instance for that World. This is the actual home of the data; `APerformanceViewerHUD` is just one consumer of it.

```cpp
UFUNCTION(BlueprintPure)
UPerfStatsCollector* GetStatsCollector() const;

UPROPERTY(EditAnywhere, BlueprintReadWrite)
float TargetFPS = 60.f;

UPROPERTY(EditAnywhere, BlueprintReadWrite)
float DataCaptureRate = 10.f;

UPROPERTY(BlueprintAssignable)
FOnPerfStatsUpdated OnStatsUpdated;
```

Fetch it from anywhere:

```cpp
UPerformanceViewerSubsystem* PV = GetWorld()->GetSubsystem<UPerformanceViewerSubsystem>();
const FPerfFrameData& Frame = PV->GetStatsCollector()->GetCurrentFrame();
```

In Blueprint: `Get Game Instance` isn't needed — use the `Get Performance Viewer Subsystem` node (World Context auto-supplied), or skip this entirely and bind straight to [`UPerformanceViewerBlueprintLibrary`](#uperformanceviewerblueprintlibrary) instead.

### OnStatsUpdated — the push-based alternative to Event Tick

`OnStatsUpdated` fires `DataCaptureRate` times/second (default 10, clamped 1–120) carrying the latest `FPerfFrameData`. Bind your `SetText` nodes (or any other custom update logic) to it once, in your widget's Construct graph, instead of polling in Event Tick or relying on UMG property Binds' own per-frame polling.

**Important: `DataCaptureRate` only throttles the broadcast, not the sampling.** `UPerfStatsCollector::Tick()` still runs every single engine frame underneath regardless of this value — FPS smoothing, hitch detection, and thread timing all need real per-frame deltas to stay accurate, so lowering `DataCaptureRate` doesn't make those readings less precise, only less frequently *pushed out* to listeners.

The built-in overlay (`UPerfOverlayWidget`) does **not** use this event — `APerformanceViewerHUD::Tick()` still calls `UpdateStats()` unconditionally every frame, same as before `OnStatsUpdated` existed. `OnStatsUpdated` is specifically for custom panels that want a cheaper, self-chosen update cadence; the stock overlay stays maximally live by design.

---

## UPerfStatsCollector

Collects all performance data each tick. Created and owned by `UPerformanceViewerSubsystem`.

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
| `VRAMUsedMB` | `float` | **This process's own** GPU memory usage. Windows: `IDXGIAdapter3::QueryVideoMemoryInfo` (`CurrentUsage`), inherently per-process. Linux: `VK_EXT_memory_budget` (`heapUsage`) via Vulkan, same per-process semantics. Mobile: texture pool consumed. `0` = N/A. |
| `VRAMTotalMB` | `float` | Adapter's dedicated video memory (desktop) or the managed texture streaming pool budget (mobile). `0` = N/A. |
| `VRAMAvailableMB` | `float` | Budget remaining for this process. Most reliable cross-platform value. `0` = N/A. |
| `VRAMSystemUsedMB` | `float` | **All processes combined** on this adapter — system-wide, not just this game. Windows only (PDH `GPU Adapter Memory` counter). `0` when unavailable. |
| `VRAMSystemFreeMB` | `float` | `VRAMTotalMB - VRAMSystemUsedMB`. Windows only. |
| `bVRAMSystemWideAvailable` | `bool` | True only where a system-wide counter actually exists (currently Windows). Gate any UI using the System/OtherApps fields on this. |
| `VRAMOtherAppsUsedMB` | `float` | `VRAMSystemUsedMB - VRAMUsedMB` — VRAM everything *except* this game is using. Derived, only meaningful when `bVRAMSystemWideAvailable` is true. |

> **Mobile note:** Raw GPU VRAM is not exposed on Android or iOS. `VRAMTotalMB` reflects Unreal's managed texture streaming pool — this is the value that actually drives texture quality, streaming decisions, and OOM kills, making it more actionable than raw VRAM would be.
>
> **Linux note:** Per-process VRAM (`VRAMUsedMB`/`VRAMTotalMB`/`VRAMAvailableMB`) is read directly through the Vulkan API (`vkGetPhysicalDeviceMemoryProperties2` + `VK_EXT_memory_budget`, resolved dynamically via `vkGetInstanceProcAddr` against `IVulkanDynamicRHI`) — no DXGI equivalent exists on Linux, so this is the real per-platform mechanism, not a stub. There is currently no system-wide ("other apps") equivalent on Linux — Vulkan, like DXGI, only exposes a process's own usage; a true system total would need vendor-specific APIs (NVML, AMD sysfs) which aren't wired up.

#### GPU Info

Cached on the first tick. Never changes at runtime.

| Field | Type | Notes |
|---|---|---|
| `GPUName` | `FString` | e.g. `"NVIDIA GeForce RTX 4080"`, `"Adreno (TM) 740"` |
| `GPUVendor` | `FString` | One of: `NVIDIA`, `AMD`, `Intel`, `Qualcomm`, `ARM`, `Apple`, `Imagination`, `Unknown` |

#### CPU Info

`CPUCoreCount`/`CPUThreadCount` are cached on the first tick (they never change at runtime). `CPUUsagePercent`/`CPUCoresUsed` are live every tick, sourced from `FPlatformTime::GetCPUTime()` (auto-updated by the engine's own Core ticker — no manual polling needed).

| Field | Type | Notes |
|---|---|---|
| `CPUCoreCount` | `int32` | Physical core count. `-1` if unavailable. |
| `CPUThreadCount` | `int32` | Logical core count (including hyperthreads). `-1` if unavailable. |
| `CPUUsagePercent` | `float` | This process's CPU usage as a percentage of **total machine capacity** (0–100). |
| `CPUCoresUsed` | `float` | This process's CPU usage expressed as core/thread-equivalents currently in use — e.g. `2.3` means the process is using the equivalent of 2.3 logical cores right now. |

#### Rendering

| Field | Type | Notes |
|---|---|---|
| `DrawCalls` | `int32` | `-1` when not reliably available in this build config. |
| `PrimitivesDrawn` | `int32` | `-1` when not reliably available. |
| `ActorCount` | `int32` | Actor count of the collector's owning World. `-1` if no valid World. |
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
| `RecentHitchCount` | `int32` | Number of hitch frames (see `HitchMultiplier`/`HitchWindowSeconds`) within the rolling window. |

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

Extends `AHUD`. Fetches the collector from `UPerformanceViewerSubsystem` (does not own or tick it) and owns the overlay widget.

```cpp
UFUNCTION(BlueprintCallable) void ShowOverlay();
UFUNCTION(BlueprintCallable) void HideOverlay();
UFUNCTION(BlueprintCallable) void ToggleOverlay();
UFUNCTION(BlueprintPure)     bool IsOverlayVisible() const;
UFUNCTION(BlueprintPure)     UPerfStatsCollector* GetStatsCollector() const;
```

`GetStatsCollector()` still works exactly as before — kept for backward compatibility with anything already using it. New code building custom panels should prefer going through `UPerformanceViewerSubsystem` or `UPerformanceViewerBlueprintLibrary` directly, since neither requires `APerformanceViewerHUD` to be your project's actual HUD class.

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
| `Text_VRAM` | `"VRAM POOL  512 / 4096 MB"` | Falls back to `"... MB free"` or `"N/A"` |
| `Text_VRAM_Used` | `"512 MB"` | This process's own usage. `"N/A"` if unavailable |
| `Text_VRAM_Total` | `"4096 MB"` | `"N/A"` if unavailable |
| `Text_VRAM_Free` | `"3584 MB"` | Available headroom |
| `Text_VRAM_System` | `"VRAM SYSTEM  1200 MB used / 2896 MB free"` | System-wide, all processes. `"N/A"` where unavailable (currently Windows only) |
| `Text_VRAM_System_Used` | `"1200 MB"` | |
| `Text_VRAM_System_Free` | `"2896 MB"` | |

> No auto-bound slot exists yet for `VRAMOtherAppsUsedMB` — use `UPerformanceViewerBlueprintLibrary::GetTextVRAMOtherApps` directly if you want it in a custom panel.

#### Text Blocks — RAM

| Widget Name | Content | Notes |
|---|---|---|
| `Text_RAM` | `"RAM  1024 / 4096 MB"` | |
| `Text_RAM_Used` | `"1024 MB"` | |
| `Text_RAM_Total` | `"4096 MB"` | |
| `Text_RAM_Free` | `"3072 MB"` | |

> `RAMUsedVirtualMB` also has no auto-bound slot — `UPerformanceViewerBlueprintLibrary::GetTextRAMVirtual` covers it for custom panels.

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
| `Text_DrawCalls` | `"Draws  N/A"` (build-config dependent) |
| `Text_Primitives` | `"Prims  N/A"` (build-config dependent) |
| `Text_Hitches` | `"HITCHES  0"` |
| `Text_ActorCount` | `"Actors  142"` |

> No auto-bound slots exist yet for CPU stats (`CPUCoreCount`/`CPUThreadCount`/`CPUUsagePercent`/`CPUCoresUsed`) — `UPerformanceViewerBlueprintLibrary` covers all four for custom panels.

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

Add a **Perf Sparkline 01** widget (see [UPerfSparklineWidget01 / 02](#uperfsparklinewidget01--02)) to your layout and name it as below. Found in the UMG palette under *PerformanceViewer*.

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
                   MemPressure: EMemoryPressure, PressureRatio: float,
                   VRAMSystemUsedMB, VRAMSystemFreeMB: float, bVRAMSystemWideAvailable: bool)

BP_OnGraphicsInfo(GraphicsAPI: string, DrawCalls: int,
                  GPUName: string, GPUVendor: string)

BP_OnPlatformInfo(BatteryLevel: int, TemperatureLevel: int)
```

### Static Helper Functions

Available in Blueprint anywhere (no instance needed) — these stay on `UPerfOverlayWidget` rather than the newer Blueprint Library so existing graphs referencing them don't break.

| Function | Input | Returns |
|---|---|---|
| `GetHealthColor` | `EFPSHealth` | `FLinearColor` — green / yellow / red |
| `GetHealthLabel` | `EFPSHealth` | `FString` — `"GOOD"` / `"WARNING"` / `"CRITICAL"` |
| `GetBottleneckColor` | `EPerfBottleneck` | `FLinearColor` — green / orange / blue |
| `GetBottleneckLabel` | `EPerfBottleneck` | `FString` — `"CPU"` / `"GPU"` / `"BALANCED"` |
| `GetMemoryPressureColor` | `EMemoryPressure` | `FLinearColor` — green / yellow / red |
| `GetMemoryPressureLabel` | `EMemoryPressure` | `FString` — `"SAFE"` / `"WARNING"` / `"CRITICAL"` |

---

## UPerformanceViewerBlueprintLibrary

The direct-bind catalogue — **72 static functions**: 40 pre-formatted `FText` getters plus 32 raw native-type getters, one raw getter per `FPerfFrameData` field (not per Text function — `GetTextFPS`/`GetTextFPSValue` both read `FPS`, so there's one `GetFPS`, not two). Text getters are string-formatted identically to what the built-in overlay shows (`AutoUpdateTextBlocks` calls these same functions internally, so the stock overlay and any hand-built panel never drift apart); Raw getters return the native `float`/`int32`/`bool`/enum/`FString` value for binding non-Text properties (a `ProgressBar`'s Percent, a `Slider`'s Value) or doing Blueprint math, without going through `GetStatsCollector → GetCurrentFrame → Break`.

Every function takes a `WorldContextObject` pin, auto-hidden and auto-wired to `self` by the Designer (the same mechanic as `GetGameInstance`/`GetPlayerController`) — so binding one is a single step: click **Bind** on a property, pick the function, done. Text getters return `FText::GetEmpty()` and Raw getters return the same sentinel `FPerfFrameData`'s own default constructor would (`0.f` / `-1` / `"Unknown"` / etc.) if the subsystem/collector isn't available yet (e.g. Designer preview, no PIE running) — safe to bind unconditionally, never crashes.

Categories pair up — `PerformanceViewer|Text|Memory` sits next to `PerformanceViewer|Raw|Memory` in the node picker, etc.

#### FPS — Text (5) / Raw (3)

`GetTextFPS` · `GetTextFPSValue` · `GetTextFrameTime` · `GetTextFrameTimeValue` · `GetTextFPSHealth`

`GetFPS` · `GetFrameTimeMS` · `GetFPSHealth`

#### Threads / Bottleneck — Text (7) / Raw (4)

`GetTextGameThread` · `GetTextGameThreadValue` · `GetTextRenderThread` · `GetTextRenderThreadValue` · `GetTextGPU` · `GetTextGPUValue` · `GetTextBottleneck`

`GetGameThreadTimeMS` · `GetRenderThreadTimeMS` · `GetGPUTimeMS` · `GetBottleneck`

#### Memory — Text (15) / Raw (13)

`GetTextVRAM` · `GetTextVRAMUsed` · `GetTextVRAMTotal` · `GetTextVRAMFree` · `GetTextVRAMSystem` · `GetTextVRAMSystemUsed` · `GetTextVRAMSystemFree` · `GetTextVRAMOtherApps` · `GetTextRAM` · `GetTextRAMUsed` · `GetTextRAMTotal` · `GetTextRAMFree` · `GetTextRAMVirtual` · `GetTextMemoryPressure` · `GetTextMemoryPressurePct`

`GetVRAMUsedMB` · `GetVRAMTotalMB` · `GetVRAMAvailableMB` · `GetVRAMSystemUsedMB` · `GetVRAMSystemFreeMB` · `IsVRAMSystemWideAvailable` · `GetVRAMOtherAppsUsedMB` · `GetRAMUsedMB` · `GetRAMTotalMB` · `GetRAMUsedVirtualMB` · `GetMemoryPressure` · `GetMemoryPressureRatio`

> Note: `GetTextVRAM`/`GetTextRAM` are combined display strings built from multiple fields, so they have no single 1:1 Raw counterpart — read the individual `VRAMUsedMB`/`VRAMTotalMB`/etc. Raw getters directly instead.

#### CPU — Text (4) / Raw (4)

`GetTextCPUCores` · `GetTextCPUThreads` · `GetTextCPUUsage` · `GetTextCPUCoresUsed`

`GetCPUCoreCount` · `GetCPUThreadCount` · `GetCPUUsagePercent` · `GetCPUCoresUsed`

#### Scene / Rendering — Text (4) / Raw (4)

`GetTextDrawCalls` · `GetTextPrimitives` · `GetTextHitches` · `GetTextActorCount`

`GetDrawCalls` · `GetPrimitivesDrawn` · `GetRecentHitchCount` · `GetActorCount`

#### Platform — Text (5) / Raw (5)

`GetTextGraphicsAPI` · `GetTextGPUName` · `GetTextGPUVendor` · `GetTextBattery` · `GetTextTemperature`

`GetGraphicsAPI` · `GetGPUName` · `GetGPUVendor` · `GetBatteryLevel` · `GetTemperatureLevel`

Example (C++, though the whole point is you don't need to write this — bind directly in the Designer instead):

```cpp
const FText FPSText = UPerformanceViewerBlueprintLibrary::GetTextFPS(this);
const float RawFPS = UPerformanceViewerBlueprintLibrary::GetFPS(this);
```

---

## UPerfSparklineWidget01 / 02

Two interchangeable sparkline flavors, both found in the UMG palette under **PerformanceViewer**. Feed either with `SetData(TArray<float>)` — called automatically by the overlay for the `Sparkline_*` bindings; only needed manually if driving a sparkline outside the overlay system.

### UPerfSparklineWidget01 — health-indicator style

The original sparkline: a filled-area line that auto-colors by how close the latest value is to a danger zone, plus an optional threshold reference line. Built for "is this metric in trouble right now," not general charting.

| Property | Type | Default | Description |
|---|---|---|---|
| `ColorGood` / `ColorWarning` / `ColorCritical` | `FLinearColor` | Green / Yellow / Red | Line + fill colour, picked by where the latest value sits between `WarningRatio` and `CriticalRatio`. |
| `ThresholdColor` | `FLinearColor` | White 35% α | Colour of the horizontal threshold line. |
| `FillAlpha` | `float` | `0.25` | Opacity of the filled area under the line. |
| `LineThickness` | `float` | `1.5` | Sparkline stroke width in pixels. |
| `ThresholdValue` | `float` | `0` | Y value the threshold line is drawn at. `0` = hidden. Set automatically by the overlay for FPS/FrameTime sparklines. |
| `FixedMaxValue` / `FixedMinValue` | `float` | `0` | Pin the Y axis. `0` (max) = auto-scale to data. |
| `WarningRatio` / `CriticalRatio` | `float` | `0.75` / `0.90` | Fraction of the Y range at which colour switches. |

### UPerfSparklineWidget02 — general-purpose chart style

Ported from UMGToolkit's `UUMGGrouperSparkline` (Line and Bar styles only — Candlestick is trading-chart-specific and irrelevant to performance metrics, so it wasn't carried over). A fixed line/bar colour rather than a danger-zone indicator, an optional reference grid, and Material-capable brush styling for bars — for panels that want a plain trend chart instead of a health readout.

| Property | Type | Default | Description |
|---|---|---|---|
| `Style` | `EPerfSparkline02Style` | `Line` | `Line` or `Bar`. |
| `bAutoScale` / `MinValue` / `MaxValue` | | `true` / `0` / `100` | Same auto-scale-or-fixed convention as Widget01. |
| `Padding` | `float` | `4` | Inset between the widget's edges and the drawn chart. |
| `LineColor` / `LineThickness` | | White / `2.0` | Line style only. |
| `BarBrush` / `BarWidthRatio` | | — / `0.7` | Bar style only. `BarBrush` can be a Material. |
| `bDrawGrid` | `bool` | `false` | Static background reference grid. |
| `GridStyle` | `EPerfSparkline02GridStyle` | `Lines` | `Lines` (plain reference lines) or `TiledImage` (a brush tiled `GridColumns` × `GridRows` times — can be a Material). |
| `GridColumns` / `GridRows` | `int32` | `4` / `4` | Grid density. |
| `VerticalGridColor` / `HorizontalGridColor` / `GridLineThickness` | | White 15% α / White 15% α / `1.0` | `Lines` grid style only. |
| `GridCellBrush` | `FSlateBrush` | — | `TiledImage` grid style only. |

---

## Platform Notes

| Feature | Windows | Mac | Linux | Android | iOS |
|---|---|---|---|---|---|
| FPS & Frame Time | ✅ | ✅ | ✅ | ✅ | ✅ |
| Game Thread Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| Render Thread Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| GPU Time | ✅ | ✅ | ✅ | ✅ * | ✅ * |
| RAM Used / Total | ✅ | ✅ | ✅ | ✅ | ✅ |
| VRAM Used / Total (this process) | ✅ DXGI | ✅ | ✅ Vulkan | Pool only | Pool only |
| VRAM system-wide / other apps | ✅ | ❌ N/A | ❌ N/A | ❌ N/A | ❌ N/A |
| VRAM Available | ✅ | ✅ | ✅ | ✅ | ✅ |
| CPU Core / Thread Count | ✅ | ✅ | ✅ | ✅ | ✅ |
| CPU Usage % | ✅ | ✅ | ✅ | ✅ | ✅ |
| GPU Name | ✅ | ✅ | ✅ | ✅ | ✅ |
| Graphics API | ✅ | ✅ | ✅ | ✅ | ✅ |
| Battery | ❌ N/A | ❌ N/A | ❌ N/A | ✅ | ✅ |
| Temperature | ❌ N/A | ❌ N/A | ❌ N/A | ✅ | ❌ N/A |
| Draw Calls / Primitives | ❌ N/A | ❌ N/A | ❌ N/A | ❌ N/A | ❌ N/A |

\* Requires `bAllowStatsInShipping=true` in Shipping builds. Degrades to FApp fallback otherwise.

Per-process VRAM on Linux goes through Vulkan directly (`VK_EXT_memory_budget`), not a fallback path — verified against a standalone reference program run under WSL2 (Ubuntu 24.04, llvmpipe software Vulkan): heap enumeration and the budget-extension support check both behave correctly; `Budget`/`Used` read `0` there specifically because llvmpipe is a software rasterizer with no real VRAM to report against, not because of a bug in the query itself. Real (non-zero) numbers need an actual GPU-backed Vulkan driver — WSL's own GPU passthrough (`dzn`, Mesa's D3D12 Vulkan driver) isn't shipped in Ubuntu's official `mesa-vulkan-drivers` package on either the rolling or 24.04 LTS channel as of this writing, so a genuine Linux box or dual-boot is still needed to see real figures before shipping.

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

Without these, the overlay still works — it just shows `N/A` for Render Thread and GPU time instead of real values. FPS, RAM, VRAM, GPU name, CPU stats, battery, and temperature all work in Shipping with no ini changes.

---

## Changelog

### v1.3
- **New:** `UPerformanceViewerSubsystem` (`UTickableWorldSubsystem`) — the collector now lives here, self-ticking, reachable from any UMG widget via `GetWorld()->GetSubsystem<...>()` regardless of which HUD class (if any) a project uses. `APerformanceViewerHUD` fetches from it instead of owning/ticking its own instance; its existing Blueprint-facing API (`GetStatsCollector()`, `TargetFPS`, `Show/Hide/ToggleOverlay`) is unchanged.
- **New:** `UPerformanceViewerBlueprintLibrary` — 72 static getters (40 pre-formatted `FText`, 32 raw native-type), directly bindable to any UMG property with zero manual formatting or `GetCurrentFrame → Break` chain. `UPerfOverlayWidget::AutoUpdateTextBlocks` now calls into the Text ones instead of duplicating the formatting logic, so the stock overlay and any custom panel read identically.
- **New:** `OnStatsUpdated` on `UPerformanceViewerSubsystem` — a `BlueprintAssignable` delegate firing `DataCaptureRate` times/second (default 10), the push-based alternative to a Blueprint Event Tick or a UMG property Bind for custom panels. Throttles only the broadcast, not the sampling — `UPerfStatsCollector::Tick()` still runs every engine frame underneath, so FPS/hitch/thread-time accuracy is unaffected. The stock overlay does not use this; it still updates every frame via the HUD, by design.
- **New:** Per-process VRAM on Linux via Vulkan (`VK_EXT_memory_budget`, resolved dynamically against `IVulkanDynamicRHI`) — the Vulkan-side counterpart to the existing Windows DXGI path.
- **New:** `VRAMSystemUsedMB` / `VRAMSystemFreeMB` / `bVRAMSystemWideAvailable` — system-wide (all processes) GPU memory usage, Windows only (PDH `GPU Adapter Memory` counter).
- **New:** `VRAMOtherAppsUsedMB` — derived (`VRAMSystemUsedMB - VRAMUsedMB`), "how much VRAM everything except this game is using."
- **New:** `CPUCoreCount`, `CPUThreadCount`, `CPUUsagePercent`, `CPUCoresUsed` — cross-platform via `FPlatformMisc`/`FPlatformTime`, no native/platform-specific code needed.
- **New:** `UPerfSparklineWidget02` — a second sparkline flavor ported from UMGToolkit's `UUMGGrouperSparkline` (Line/Bar styles + grid backdrop + Material-capable brush styling; Candlestick left out as not relevant to perf metrics). The original sparkline is renamed `UPerfSparklineWidget01` to sit alongside it.
- **New:** `GetHealthLabel` on `UPerfOverlayWidget` — `EFPSHealth`'s text counterpart, matching the existing `GetBottleneckLabel`/`GetMemoryPressureLabel` pattern (previously only had a colour helper).
- **Breaking:** `UPerfSparklineWidget` renamed to `UPerfSparklineWidget01`. Any already-placed sparkline widget instance in an existing WBP (including this plugin's own demo overlays) needs to be manually re-placed in the Designer once.

### v1.2
- **New:** `PrimitivesDrawn`, `ActorCount`, `RecentHitchCount` fields → `Text_Primitives`, `Text_ActorCount`, `Text_Hitches` bindings.
- **New:** `HitchMultiplier` / `HitchWindowSeconds` collector properties driving hitch detection.

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
