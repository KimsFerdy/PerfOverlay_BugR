# Minimalistic Performance Viewer — Full Release Documentation

---

## 1. README (Plugin Folder)

### Minimalistic Performance Viewer

A clean, lightweight performance HUD overlay for Unreal Engine 5. Displays FPS, frame time, and RAM on all platforms — with battery on mobile. Includes scrolling sparkline graphs and color-coded health indicators.

> UE 5.3+ | Windows | Android | Runtime Plugin

---

#### Table of Contents
- [Features](#features)
- [Installation](#installation)
- [Setup](#setup)
- [Widget Bindings Reference](#widget-bindings-reference)
- [Blueprint Events Reference](#blueprint-events-reference)
- [Sparkline Widget Guide](#sparkline-widget-guide)
- [Toggling the Overlay](#toggling-the-overlay)
- [Platform Notes](#platform-notes)
- [Troubleshooting](#troubleshooting)

---

#### Features

- FPS with exponential moving average smoothing
- Frame time in milliseconds
- RAM usage (physical)
- Battery level on Android (auto-hidden on desktop)
- FPS health color coding — Green / Yellow / Red based on your target FPS
- Scrolling sparkline graphs with fill area and threshold lines
- All stats exposed to Blueprint via named widget auto-bindings and events
- Toggle overlay at runtime via Blueprint or C++
- Zero N/A clutter — unsupported stats auto-hide per platform

---

#### Installation

1. After installing your Addon from FAB
2. Enable via **Edit → Plugins → Performance → Minimalistic Performance Viewer**

Your `.uproject` should have this — the editor adds it automatically:
```json
{ "Name": "PerformanceViewer", "Enabled": true }
```
> ⚠️ Do NOT add `TargetAllowList` to this entry — it causes cook failures.

---

#### Setup

**Step 1 — Create your overlay Widget Blueprint**
- Content Browser → Add → User Interface → Widget Blueprint
- Parent class: `PerfOverlayWidget` (search All Classes)
- Name it e.g. `WBP_PerfOverlay`

**Step 2 — Create your HUD Blueprint**
- Content Browser → Add → Blueprint Class
- Parent class: `PerformanceViewerHUD`
- Name it e.g. `BP_PerfHUD`
- Class Defaults → **Overlay Widget Class** → `WBP_PerfOverlay`
- Class Defaults → **Target FPS** → your target (default: 60)

**Step 3 — Assign to your Game Mode**
- Game Mode Blueprint → Class Defaults → **HUD Class** → `BP_PerfHUD`

**Step 4 — Wire up Event Graph in your overlay widget**
```
Event Tick
  → Get Owning Player → Get HUD → Cast To BP_PerfHUD
  → Get Stats Collector
  → Update Stats (Target: self, Collector: from Get Stats Collector)
```
> ⚠️ Do NOT add the widget to viewport manually — the HUD does this in BeginPlay.

---

#### Widget Bindings Reference

Name your UMG widgets with these exact variable names and C++ populates them every tick. All optional.

**Text Blocks**

| Variable Name | Content | Platform | Color-coded |
|---|---|---|---|
| `Text_FPS` | `105 FPS` | All | ✅ |
| `Text_FrameTime` | `9.21 ms` | All | ❌ |
| `Text_RAM` | `RAM  599 MB` | All | ❌ |
| `Text_Battery` | `Battery  85%` | Android only — auto-hidden on desktop | ❌ |

**Progress Bars**

| Variable Name | Fills based on | Platform |
|---|---|---|
| `Bar_RAM` | RAM used / 8192 MB | All |
| `Bar_Battery` | Battery 0–100 | Android only — auto-hidden on desktop |

> RAM bar max is 8192 MB. Change `MaxRAMMB` in `PerfOverlayWidget.cpp` to match your target.

**Sparklines**

| Variable Name | Data | Threshold line | Platform |
|---|---|---|---|
| `Sparkline_FPS` | FPS history | Target FPS | All |
| `Sparkline_FrameTime` | Frame time (ms) | 1000 / TargetFPS | All |
| `Sparkline_RAM` | RAM (MB) | None | All |

**Adding a label prefix**

Option A — Static Text Block with `FPS:` next to it in a Horizontal Box.

Option B — Override `BP_OnFPSUpdated`:
```
Event BP_OnFPSUpdated
  → Format Text "FPS: {fps}"
  → Set Text on Text_FPS
```

---

#### Blueprint Events Reference

| Event | Parameters | Use for |
|---|---|---|
| `BP_OnStatsUpdated` | `FPerfFrameData Frame` | All stats at once |
| `BP_OnFPSUpdated` | `FPS`, `FrameTimeMS`, `Health`, `HealthColor` | Custom FPS display |
| `BP_OnMemoryUpdated` | `VRAMUsedMB`, `VRAMTotalMB`, `RAMUsedMB` | Memory display |
| `BP_OnBatteryUpdated` | `BatteryLevel` | Mobile battery display |

**Helper (Blueprint Pure)**

| Function | Returns |
|---|---|
| `GetHealthColor(Health)` | `FLinearColor` — Green / Yellow / Red |

**Health thresholds**

| State | Condition | Color |
|---|---|---|
| Good | FPS >= TargetFPS | 🟢 Green |
| Warning | FPS >= TargetFPS × 0.5 | 🟡 Yellow |
| Critical | FPS < TargetFPS × 0.5 | 🔴 Red |

---

#### Sparkline Widget Guide

Find **Perf Sparkline** in the UMG widget palette under PerformanceViewer.

Recommended size: 200×50 or 160×40.

| Property | Default | Description |
|---|---|---|
| `ColorGood` | Green | Line color when healthy |
| `ColorWarning` | Yellow | Line color at warning threshold |
| `ColorCritical` | Red | Line color at critical threshold |
| `FillAlpha` | 0.25 | Opacity of fill area |
| `LineThickness` | 1.5 | Stroke width in pixels |
| `ThresholdValue` | 0 | Horizontal reference line (0 = off) |
| `FixedMaxValue` | 0 | Fix Y axis max (0 = auto-scale) |
| `WarningRatio` | 0.75 | Fraction of max triggering Warning |
| `CriticalRatio` | 0.90 | Fraction of max triggering Critical |

---

#### Toggling the Overlay

Blueprint — call on `BP_PerfHUD`: `Toggle Overlay`, `Show Overlay`, `Hide Overlay`, `Is Overlay Visible`

C++:
```cpp
APerformanceViewerHUD* PVHud = Cast<APerformanceViewerHUD>(GetHUD());
if (PVHud) PVHud->ToggleOverlay();
```

---

#### Platform Notes

| Platform | FPS | FrameTime | RAM | Battery |
|---|---|---|---|---|
| Windows | ✅ | ✅ | ✅ | ➖ hidden |
| Android | ✅ | ✅ | ✅ | ✅ |
| Mac* | ✅ | ✅ | ✅ | ➖ hidden |
| Linux* | ✅ | ✅ | ✅ | ➖ hidden |
| iOS* | ✅ | ✅ | ✅ | ✅ |

*Not officially supported yet — untested. Coming in a future update.

---

#### Troubleshooting

**Plugin fails to cook / "module not available on platform"**
Make sure `.uproject` entry has no `TargetAllowList`. Delete `Saved/Cooked/` and repackage.

**Cook takes 1–2 hours**
Same root cause as above. Fix the `.uproject` entry.

**Overlay not showing**
Confirm HUD Class is set in Game Mode. Confirm Overlay Widget Class is set in `BP_PerfHUD`. Do not add widget to viewport manually.

**All stats showing 0**
`Update Stats` is not being called. Check Event Tick graph in your overlay widget.

**RAM bar seems wrong**
Default max is 8192 MB. Edit `MaxRAMMB` in `PerfOverlayWidget.cpp` to match your platform.
