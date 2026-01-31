# Slicer Side Bar

The slicer and slicer sidebar in the Virtual Anatomy project work together to provide users with an intuitive way to interact with and manipulate a slicing plane within the 3D anatomy environment. The slicer itself is an in-world actor responsible for rendering and managing the slicing of anatomical models along a customizable plane. This plane can be moved and rotated in real time, enabling users to explore cross-sections of the body from different angles and depths.

The slicer sidebar is a user interface widget built with UMG that gives users control over the slicer’s behavior. When the sidebar becomes visible, it triggers the slicer to display its slicing plane in the 3D space. Conversely, when the sidebar is hidden, the slicer plane is also hidden to keep the interface clean and focused.

The sidebar provides a checkbox to enable or disable the slicer entirely, sliders to fine-tune the position and rotation of the slicer plane, and several buttons that allow for quick application of predefined distance and rotation values. These controls send commands to the slicer actor in the scene, updating its transform parameters accordingly.

During initialization, the sidebar checks for the presence of the slicer actor and associated widgets. If any required component is missing, it sets an internal error state, which can be queried to diagnose configuration issues. Additionally, the sidebar loads any previously saved slicer parameters from the game instance so that the interface reflects the last-used values, offering a consistent user experience across sessions.

Overall, the slicer and its sidebar work in tandem to give users precise and flexible control over anatomical visualization, blending interactive 3D manipulation with a clean and accessible interface.

## Session Start / Saved State
- On construct, previously saved distance (arm length) and rotation (yaw) are restored.
- The enabled state (checkbox) is not auto-restored: slicer always starts disabled (OFF) for safety and consistency; checkbox is forced unchecked.
- Checkbox mapping: checked = slicer ON (value 0.0 passed to ToggleSlicer), unchecked = slicer OFF (value 1.0).

## Slider Quantization Toggle
- Distance and rotation sliders can operate in snapped (quantized) or free mode.
- When the optional Free Adjust toggle is present and checked, sliders move freely (no snapping).
- When unchecked, slider values snap to configured StepSize.

## Plane Visibility Rules
- When the sidebar widget is shown (opened), the sidebar calls `ShowSlicerPlane()`, which does `SlicerActor->GetSlicerPlane()->SetVisibility(true)` if the plane component exists.
- When the sidebar widget is hidden (closed), it calls `HideSlicerPlane()`, which sets the plane mesh visibility to `false`.
- This only shows/hides the plane mesh in the viewport. It does not enable or disable the slicing effect; that is controlled by the checkbox (which writes `isSlicerOff` via `ToggleSlicer`). The slicer can be ON while the plane mesh is hidden to keep the view uncluttered.
- Both functions are null-guarded; if the plane component is missing, a warning is logged and the call is skipped.

## Error Handling Summary
- Missing slicer actor or critical widget → sidebar sets internal error flag and disables further interactions.
- Missing plane component logs a warning but does not hard-disable the UI.
