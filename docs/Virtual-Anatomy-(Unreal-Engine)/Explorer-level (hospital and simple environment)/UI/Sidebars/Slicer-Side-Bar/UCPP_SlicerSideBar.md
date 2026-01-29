# `UCPP_SlicerSideBar`

A user interface widget for controlling the slicer (enable/disable, distance, rotation) and visualizing the plane.

Responsibilities:
- Show/hide slicer plane when the sidebar is shown/hidden
- Toggle slicer enabled state via checkbox (writes `isSlicerOff`)
- Adjust distance and rotation via reusable slider widgets
- Provide quick preset buttons for distance/rotation
- Restore saved position/rotation values at construct
- Manage error states when slicer actor or widgets are missing
- Optional: toggle between snapped vs free slider movement

## Lifecycle

### `NativeConstruct()`
- Finds `BP_Slicer_C_0` in the world and caches `ACPP_Slicer` reference.
- Binds checkbox and slider callbacks.
- Binds distance/rotation preset buttons.
- If present, binds `FreeAdjustToggle` (snap vs free movement) and applies initial state.
- Hides the slicer plane mesh initially if present.
- Loads saved values from `UCPP_GameInstance` (Position, Direction, DisablementState).
- Forces checkbox unchecked and applies `ChangeSlicerCheckbox(false)` so the slicer starts disabled (matching ACPP_Slicer BeginPlay).
- Restores saved `Position` and `Direction` to sliders and actor.

## Controls

### Checkbox → enabled/disabled
`ChangeSlicerCheckbox(bool bIsChecked)` sets:
- `NewValue = bIsChecked ? 0.0f : 1.0f`
- `SlicerActor->ToggleSlicer(NewValue)`
- Updates internal `SlicerParameters` mirror

This means: checked = ON (0.0), unchecked = OFF (1.0).

### Distance slider
`ChangeSlicerDistance(float NewValue)`:
- Updates `SlicerParameters.Distance`
- Calls `SlicerActor->SetSpringArmLength(NewValue)`
- Updates `DistanceSlider->MainSlider` value

### Rotation slider
`ChangeSlicerRotation(float NewValue)`:
- Updates `SlicerParameters.Rotation`
- Sets yaw on `ACPP_Slicer` using `SetRotation`
- Updates `RotationSlider->MainSlider` value

### Preset buttons
- Distance: 0, 50, 100, 150, 200 (cm)
- Rotation: Left(0°), Right(180°), Rear(90°), Front(270°)

### Free adjust toggle (unlock slider)
If the optional `FreeAdjustToggle` is bound:
- Checked = free movement (no quantization)
- Unchecked = snap to step size
- Calls `UCPP_SlicerSlideBar::SetQuantizeEnabled` for both sliders.

## Plane visualization rules
- When sidebar becomes visible, call `ShowSlicerPlane()` → attempts `SlicerActor->GetSlicerPlane()->SetVisibility(true)` if the component exists.
- When hidden, call `HideSlicerPlane()` similarly sets plane invisible.

## Error handling
- If slicer blueprint or required widgets are missing, sets `bIsSlicerGivingError = true` and early-outs.
- Logs warnings for missing plane component; keeps sidebar usable where possible.
