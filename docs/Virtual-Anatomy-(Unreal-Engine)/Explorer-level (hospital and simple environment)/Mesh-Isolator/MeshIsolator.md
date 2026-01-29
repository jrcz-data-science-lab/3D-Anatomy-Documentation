# UCPP_IsolateMesh (Mesh Isolator)

## Class Description

`UCPP_IsolateMesh` provides "focus isolation" for a selected anatomical mesh by:
1. Moving the camera's pivot / target actor (`TargetActor`) to the center of the currently selected mesh.
2. Adjusting the camera radius (zoom) using either a fixed distance or a dynamic distance derived from the mesh bounds.


---
## Lifecycle & Dependencies

The object is created as a subobject in `ACPP_GameMode` (`CreateDefaultSubobject<UCPP_IsolateMesh>("IsolateMesh")`) and later initialized in `BeginPlay()` via `Init(UWorld* World)`.

During `Init` it acquires and caches:
- `m_userRef` (player character cast to `ACPP_User`) – used for `TargetActor` and camera control access.
- `m_selector` – gives currently selected mesh component.
- `m_camera` – camera controller used to set look target, radius, and derive position/rotation.
- Stores the initial `TargetActor` location in `targetActorLocation` for potential future restoration logic.

If any dependency is missing, warnings are logged and isolation is skipped.

---
## Public Methods

### `UCPP_IsolateMesh()`
Constructor. Sets default property values (no heavy initialization).

### `void Init(UWorld* World)`
Initializes internal references (User, MeshSelector, CameraControls) and captures the original pivot location.
- `World`: world context provided by GameMode.
- Logs initialization status for each dependency (`OK` / `NULL`).

### `void MoveCenterPoint()`
Primary action method executed (e.g. via button click). Workflow:
1. Validates `m_userRef`, `m_selector`, `m_camera`.
2. Retrieves the currently selected mesh (`UMeshComponent`).
3. Computes mesh center from `Bounds.Origin` and sets the `TargetActor` location there.
4. Updates camera look target via `m_camera->SetLookAtTarget()`.
5. Chooses zoom distance:
   - If `bUseFixedZoomDistance == true` uses `FixedZoomDistance`.
   - Otherwise computes `DesiredDistance = max(1, SphereRadius) * ZoomPaddingFactor`.
6. Applies zoom with `m_camera->SetRadiusDirectly(DesiredDistance)`.
7. Updates user actor transform to match the camera (location + rotation).
8. Caches the mesh in `ZoomedTarget` (currently stored only; not yet used to skip redundant calls).

Fails early with logs if any required pointer or mesh is missing.

---
## Public Properties

### `UPROPERTY(Transient) TObjectPtr<ACPP_User> m_userRef`
Access level: public (per header). Transient reference to the user/player character. Used to access `TargetActor`, camera controls, and to sync player actor transform after zoom.

---
## Private Properties

### `UWorld* m_world`
Raw pointer (engine-owned) to the current world; not a UPROPERTY (intentional: world is engine-managed).

### `UPROPERTY(Transient) TObjectPtr<UCPP_CameraControls> m_camera`
Pointer to camera controller for setting look target and radius.

### `UPROPERTY(Transient) TObjectPtr<UMeshSelector> m_selector`
Pointer to mesh selection system to obtain the currently selected mesh component.

### `FVector targetActorLocation`
Stored original location of the camera pivot (`TargetActor`) captured during initialization. (Currently only stored; restoration logic may be added later.)

### `UPROPERTY(EditAnywhere, Category="MeshIsolator|Zoom") float ZoomPaddingFactor`
Multiplier applied to mesh radius when computing dynamic zoom distance (only when `bUseFixedZoomDistance == false`). Values < 1 zoom tighter; > 1 add more framing space. Default: `0.70f`.

### `UPROPERTY(EditAnywhere, Category="MeshIsolator|Zoom") bool bUseFixedZoomDistance`
If true, always uses `FixedZoomDistance` instead of dynamic mesh-based sizing. Default: `true`.

### `UPROPERTY(EditAnywhere, Category="MeshIsolator|Zoom", meta=(ClampMin="50.0", ClampMax="5000.0")) float FixedZoomDistance`
Absolute camera distance (in cm) when using fixed zoom mode. Clamped 50–5000. Default: `120.0f`.

### `TWeakObjectPtr<UMeshComponent> ZoomedTarget`
Weak pointer tracking the last mesh isolated. NOTE: Current implementation assigns this but does not yet use it to short-circuit repeated calls—future optimization opportunity.

---
## Removed / Deprecated (Older Docs Reference)

The following previously documented members no longer exist in the current header and should not be referenced:
- `HasCenterPointMoved`
- `meshOriginalLocation` (mesh is no longer moved)

---
## Usage Example

```cpp
// Acquire instance (simplified helper)
auto* IsolateMesh = AnatomyUtils::GetIsolateMeshInstance(GetWorld());
if (IsolateMesh)
{
    IsolateMesh->MoveCenterPoint();
}
```

---
## Extension / Future Ideas

- Add method to restore original pivot (`RestoreOriginalPivot()`).
- Add smoothing or animated interpolation for pivot and zoom changes.
- Use `ZoomedTarget` to skip re-isolation if same mesh is selected.
- Support multiple selection isolation cycling.
- Provide a Blueprint callable wrapper for UI designers.

---
## Error Handling & Logging

- Missing references: logs `Warning` and aborts.
- Null `TargetActor` or selected mesh: logs `Warning` and aborts.
- Successful initialization: logs dependency status and original pivot location.

---
## Performance Notes

Current operations are lightweight (position & radius changes only). Dynamic zoom calculation is O(1). No mesh duplication or material changes occur.

---
## Key Takeaways

- Isolation is achieved by shifting camera focus, not relocating meshes.
- Zoom behavior is configurable: fixed versus dynamic radius-based framing.
- Safe pointer checks prevent runtime crashes when dependencies are missing.
- Weak pointer avoids strong ownership of target mesh.

---
