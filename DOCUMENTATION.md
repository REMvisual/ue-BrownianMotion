# Brownian Motion — Documentation

**Ornstein–Uhlenbeck mean-reverting noise component for Unreal Engine 5.6 / 5.7 / 5.8 (Win64).**

This guide covers installation, setup, every parameter, the Blueprint API, Sequencer integration, and troubleshooting. No third-party software is required — the plugin is self-contained (two modules: `BrownianMotion` runtime, `BrownianMotionEditor` editor-only).

---

## 1. Installation

1. Purchase/get **Brownian Motion** on [Fab](https://www.fab.com/sellers/REMrepo) and install it to your engine via the **Epic Games Launcher** (Library → Fab Library → Install to Engine). Supported engine versions: **UE 5.6, 5.7, 5.8**.
2. Open your project and enable the plugin: **Edit → Plugins → search "Brownian Motion" → Enable**. Restart the editor when prompted.
3. Add a **Brownian Motion** component to any actor (Add Component → search "Brownian"). Motion starts with sensible defaults the moment you press **Play**.

> Ships with full C++ source. The runtime module works in packaged/shipping builds; only the custom Sequencer track editor is editor-only.

---

## 2. Two ways to use it

**Parent mode (default, recommended).** Add the component to an actor and parent your mesh components *underneath* it in the component tree. The component moves itself; children inherit the motion. Nothing else to configure.

**Affect Parent mode.** Drop the component onto any existing actor and enable **Affect Parent**. The component moves the actor's root — no hierarchy changes needed. If the parent is Static, the component automatically promotes its mobility to Movable (and reverts when Affect Parent is disabled).

**Editor preview.** Enable **Preview in Editor** to see the motion in the viewport without entering PIE.

---

## 3. Parameter reference

All parameters are `BlueprintReadWrite`. Every animatable parameter carries the **Interp** flag, so it can be keyframed in Sequencer (19 keyframable properties in total).

### General

| Property | Type | Default | Description |
|---|---|---|---|
| Active | bool | `true` | Master on/off. Smoothly blends motion in and out, returning to the original position when off. |
| Blend Duration | float | `0.5` s | How long the blend in/out takes when toggling Active (min 0.01). |
| Affect Parent | bool | `false` | Move the parent component/actor root instead of self. |
| Preview in Editor | bool | `false` | Preview motion in the viewport without Play. |

### Motion

| Property | Type | Default | Range | Description |
|---|---|---|---|---|
| Speed | float | `1.0` | 0 – 10 | Time-scale multiplier. `0` = frozen. |
| Amplitude | float | `1.0` | 0 – 2 | Output multiplier. `0` = no motion, `1` = full range. Ideal for Sequencer fade-in/out. |
| Center Pull | float | `2.0` | 0 – 20 | Mean-reversion strength — higher pulls back to center faster. |
| Smoothing | float | `0.5` | 0 – 1 | Critically-damped spring smoothing. `1.0` = very smooth, `0.0` = raw unfiltered noise. |
| Detail | float | `0.0` | 0 – 1 | Fractal detail amount (Voss–McCartney 1/f pink noise). `0` = off. |
| Detail Layers | int | `3` | 1 – 5 | Number of pink-noise layers (visible when Detail > 0). More = richer spectrum. |

### Range

| Property | Type | Default | Description |
|---|---|---|---|
| Per-Direction Range | bool | `false` | When true, each axis (X/Y/Z) gets its own min/max range. |
| Range Min / Range Max | float | `−100` / `100` cm | Displacement envelope, all directions (when Per-Direction Range is off). |
| Range Min/Max X, Y, Z | float | `−100` / `100` cm | Per-axis envelopes (when Per-Direction Range is on). |
| Affect X / Y / Z | bool | `true` | Enable motion per axis. |

Motion is sealed inside the range envelope by the mean-reversion math itself — no clamping, no snapping.

### Rotation

| Property | Type | Default | Description |
|---|---|---|---|
| Enable Rotation | bool | `false` | Adds an independent OU noise process on rotation. |
| Rotation Speed | float | `1.0` | Rotation time-scale, independent from position Speed. |
| Rotation Amplitude | Rotator | `(5, 5, 0)` ° | Per-axis rotation amplitude in degrees (pitch, yaw, roll). Set an axis to `0` to disable it. |

### Collision avoidance

| Property | Type | Default | Description |
|---|---|---|---|
| Enable Collision | bool | `false` | Sphere-sweep repulsion: motion bends away from geometry before contact. |
| Auto Collision Size | bool | `true` | Derive both radii from the owning actor's mesh bounds. |
| Collision Radius | float | `10` cm | Hard shell — the object never gets closer than this to a surface (manual mode only). |
| Detection Radius | float | `50` cm | How far out surfaces are sensed and steering begins (manual mode only). |
| Draw Debug | bool | `false` | Draw collision/detection spheres in the editor viewport. |

The system sweeps in six directions and bends the trajectory (repulsion, not a binary block). An overlap-escape system recovers objects that start inside geometry. Penetrations are counted and reported — see Diagnostics.

### Advanced

| Property | Type | Default | Description |
|---|---|---|---|
| Anchor | Vector | `(0,0,0)` | Bias per axis in normalized space: `0` = center of range, `−1` = toward min, `1` = toward max. |
| Seed | int | `0` | `0` = different motion every play. Any non-zero integer = fully deterministic, identical every run. |
| Independent Axes | bool | `true` | Each axis gets independent noise; when false all axes share one sample. |

### Debug

| Property | Type | Default | Description |
|---|---|---|---|
| Debug Log | bool | `false` | Detailed per-tick logging to the Output Log. |

---

## 4. Blueprint API

All functions are `BlueprintCallable` in category **Brownian Motion**.

| Function | Description |
|---|---|
| `Reset()` | Re-randomize the OU state and zero diagnostics (respects Seed). |
| `SetOrigin()` | Capture the current location as the motion origin. |
| `ActivateMotion()` | Smoothly blend motion in. |
| `DeactivateMotion()` | Smoothly blend out and return to origin. |
| `GetCurrentOffset()` | Current world-space offset from origin. |
| `GetBlendWeight()` | `0` = fully off, `1` = fully on. |

### Diagnostics (category Brownian Motion|Debug)

| Function / member | Description |
|---|---|
| `GetOUState()` | Raw OU process state in normalized `[−1,1]` space. |
| `GetSmoothedState()` | Spring-smoothed state in normalized space. |
| `GetInitialLocation()` | Base location before any Brownian offset. |
| `GetTargetWorldLocation()` | Current world position of the motion target. |
| `GetRawOffset()` | Mapped offset before collision clamp. |
| `GetResolvedCollisionRadius()` / `GetResolvedDetectionRadius()` | Resolved radii (auto or manual). |
| `IsCurrentlyPenetrating()` | True if the actor is inside collision geometry right now. |
| `PenetrationFailureCount` | Penetrations detected since last `Reset()` (BlueprintReadOnly). |
| `OnPenetrationDetected` | Assignable event — fires with world location and normal when penetration is detected despite avoidance. |

---

## 5. Sequencer integration

**Keyframing component parameters.** With the actor selected, every animatable property (Speed, Amplitude, CenterPull, Smoothing, Detail, ranges, per-axis ranges, Anchor, axis enables, rotation parameters) appears as a keyframable track in Sequencer. Classic use: keyframe **Amplitude** to fade drift in and out during cuts.

**Standalone Brownian Motion track.** Add a *Brownian Motion* track directly to any actor from the Sequencer timeline (**+ Track → Brownian Motion**) — no component required. It exposes **Speed, Amplitude, Range Min, Range Max** channels and evaluates deterministic Perlin noise, so scrubbing the playhead is frame-exact and reproducible.

---

## 6. Recipes

**Slow, languid drift (floating debris, buoys, lanterns)**
Smoothing → `0.9–1.0`, Speed → `0.3–0.5`, Center Pull → `3–5`.

**Handheld camera**
Attach to a camera actor, Affect Parent on. Speed `0.2–0.5`, Smoothing `0.7+`, Range `10–30` cm. Add Enable Rotation with a small Rotation Amplitude (`0.5–1.5°`) for micro-rotation. Keyframe Amplitude to fade between cuts.

**Rich organic complexity**
Detail → `0.5–1.0` with `2–5` Detail Layers. Large slow sweeps and small fast details happen simultaneously.

**Nervous, jittery motion**
Smoothing → `0`, Speed → `2.5+`, Center Pull → `4+`.

**Identical motion every run**
Seed → any non-zero integer.

---

## 7. Performance notes

- The OU simulation is pure math with **no per-tick allocations**; a single component is negligible.
- Collision avoidance uses sphere traces and is the most expensive feature when enabled — mitigated by bounds caching (radii recomputed only when the actor moves) and is benchmarked in the included automation tests.
- **Ray tracing:** a dead-zone filter suppresses sub-threshold position updates (0.01 cm) so many Brownian actors don't trigger TLAS rebuild storms with hardware RT enabled.
- The plugin ships with **43 headless automation tests** (axis isolation, range coverage, blend timing, determinism, collision edge cases, smoothness, performance benchmarks). Run them via the Session Frontend → Automation tab, filter "BrownianMotion".

---

## 8. Troubleshooting

**Nothing moves.** Check Active is on and Speed/Amplitude are non-zero. In Affect Parent mode on a Static actor, mobility is promoted automatically — if you toggled it manually, ensure the root is Movable.

**Motion looks identical on two actors.** They share the same non-zero Seed — set Seed to `0` or use different seeds per actor.

**Object starts inside geometry.** The overlap-escape system steers it out automatically; watch `PenetrationFailureCount` / `OnPenetrationDetected` if you need to react.

**Too jittery / too smooth.** Raise Smoothing toward `1.0` (and/or lower Speed). Lower Smoothing toward `0` for raw noise.

**Range looks wrong after moving the actor in-editor.** The origin is captured on BeginPlay (or `SetOrigin()`). Call `SetOrigin()` after repositioning at runtime.

---

## 9. Support

- Bug reports & feature requests: [GitHub Issues](https://github.com/REMvisual/ue-BrownianMotion/issues)
- Epic Developer Community forum thread: linked from the Fab listing's Comments section.
- Free TouchDesigner version of the same math: [td-BrownianMotion](https://github.com/REMvisual/td-BrownianMotion)
