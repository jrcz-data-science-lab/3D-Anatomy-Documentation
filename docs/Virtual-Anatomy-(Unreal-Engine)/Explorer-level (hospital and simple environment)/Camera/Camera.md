# UCPP_CameraControls

## Overview

Computational camera controller for orbit-style movement around a target point. Uses spherical coordinates to derive a world position and rotation. Not a placed Actor; accessed from `UCPP_User`.

- Spherical components:
  - Polar (vertical/up–down)
  - Azimuth (horizontal/left–right)
  - Radius (distance from target)
- Converts spherical to Cartesian for Unreal coordinates and always rotates to face the LookAt target.

## Versions shipped to client

- Legacy (earlier shipped build)
  - Orbit camera using spherical coordinates (Polar, Azimuth, Radius).
  - Zoom adjusts radius with min/max clamping.
  - LookAt (target) is clamped to thresholds on X and Z.
  - No direct-radius override API.

- Current implementation (in code now)
  - Same spherical model, clamped zoom, and LookAt clamping.
  - Adds `SetRadiusDirectly(float)` to set radius precisely (used e.g. by Mesh Isolator) without applying min/max zoom clamps.

## Input mapping

- Current system (as in code now)
  - Mouse drag: pans camera on X/Z (left–right = X, up–down = Z).
    - Code path: `ACPP_User::MoveWithMouse` → `CameraControls->MoveHorizontal/MoveVertical`.
  - WASD keys: rotate camera around target.
    - W/S: vertical rotation (`RotatePolar`)
    - A/D: horizontal rotation (`RotateAzimuth`)
  - Mouse wheel (or bound axis): zoom in/out (`Zoom`), clamped to min/max radius.

- Legacy system 
  - Mouse drag: rotated camera around target (polar/azimuth changes).
  - WASD keys: panned camera on X/Z (horizontal/vertical LookAt movement).
  - Mouse wheel (or bound axis): zoom in/out (clamped).

## Definitions

- `ANATOMY_PI` — custom constant used for math (`#define ANATOMY_PI 3.14159...`).

## Public methods

- `void Init(const FVector& LookAtPos, const FVector& StartPos, float MaxRadius = 700.0f, float MinRadius = 1.0f)`
  - Sets the initial LookAt position and starting camera position.
  - Initializes min/max radius, XY/Z thresholds, and internal spherical from `StartPos`.

- `void RotatePolar(float by)`
  - Adjusts the polar (vertical) angle by radians; capped near ±(PI/2).

- `void RotateAzimuth(float by)`
  - Adjusts the azimuth (horizontal) angle by radians; wrapped into [0, 2PI).

- `void MoveHorizontal(float by)`
  - Pans the LookAt position along world X by the given amount; clamps and recomputes.

- `void MoveVertical(float by)`
  - Pans the LookAt position along world Z by the given amount; clamps and recomputes.

- `void Zoom(float by)`
  - Changes the spherical radius by the given amount; clamped to `[MinimumRadius, MaximumRadius]`.

- `void SetRadiusDirectly(float NewRadius)` (current implementation)
  - Sets spherical radius exactly without min/max zoom clamping; then recomputes.

- `void SetLookAtTarget(const FVector& LookAtTarget)`
  - Replaces the LookAt position; clamps to thresholds; recomputes.

- `FVector GetLookAtPos() const`
  - Returns the current LookAt position.

- `FVector3d& GetPosition()`
  - Returns a reference to the current camera world position.

- `FRotator& GetRotation()`
  - Returns a reference to the current camera rotation.

## Private methods

- `FVector3d RecalculatePosition()`
  - Converts spherical → Cartesian relative to `LookAtPosition`.
  - x = Center.X + R * cos(Polar) * cos(Azimuth)
  - y = Center.Y + R * cos(Polar) * sin(Azimuth)
  - z = Center.Z + R * sin(Polar)
  - Calls `RecalculateRotation()` and returns the new position.

- `void RecalculateRotation()`
  - Sets `Orientation = FindLookAtRotation(Location, LookAtPosition)`.

- `void ClampLookAtPosition()`
  - Clamps X and Z of `LookAtPosition` to threshold bounds.

## Private properties

- `FRotator Orientation`
- `FVector3d Location`
- `float Radius`
- `float MaximumRadius`
- `float MinimumRadius`
- `FVector LookAtPosition`
- `FSphericalPoint CameraLocationInSpherical`
- `const float FullCircle = ANATOMY_PI * 2.F`
- Thresholds (current defaults set in `Init`):
  - `float MinimumThresholdX = -700.0f`
  - `float MaximumThresholdX = 600.0f`
  - `float MinimumThresholdZ = 0.0f`
  - `float MaximumThresholdZ = 1200.0f`

## Notes

- All rotation inputs are radians.
- Movement methods recompute position each call; rotation is always set to face `LookAtPosition`.
- Both legacy and current versions use the same spherical model and threshold clamping; the current version additionally provides `SetRadiusDirectly` for precise radius control.
