# ACPP_Slicer

This class is responsible for controlling and visualizing a slicing plane in the application. It manages the position, rotation, and visibility of the slicer, synchronizing these properties with a material parameter collection used for mesh slicing.

- Material Parameter Collection asset path: `/Game/Slicer/SliceParameters.SliceParameters`.
- Vector parameters used: `Position` (plane world location) and `Direction` (plane up vector).
- Scalar parameter used: `isSlicerOff` (1 = off/disabled, 0 = on/enabled).

## Public methods

### `ACPP_Slicer()`

Constructor for the `ACPP_Slicer` class; enables ticking.

### `virtual void Tick(float DeltaTime)`

Called every frame (currently no per-frame logic).

Parameters:
- `DeltaTime`: Time elapsed since the last frame.

### `virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)`

Binds input actions to the pawn (not used; slicer is controlled via UI widgets).

Parameters:
- `PlayerInputComponent`: The input component used for binding controls.

### `void SetRotation(const FRotator& Rotation)`

Sets the rotation of the slicer by rotating the attached `USpringArmComponent` and updates material parameters on the next tick.

Parameters:
- `Rotation`: New world-space rotation to apply to the spring arm.

### `FRotator GetRotation()`

Returns the last stored rotation captured at BeginPlay from the spring arm.

Returns:
- `FRotator`: Initial rotation value (not live-updated after SetRotation).

### `void SetSpringArmLength(float Length)`

Changes the spring arm length (distance of the plane from its pivot) and schedules an update of material parameters on the next tick.

Parameters:
- `Length`: New arm length (cm).

### `UStaticMeshComponent* GetSlicerPlane()`

Returns the static mesh component used as the slicer plane.

Returns:
- `UStaticMeshComponent*`: The slicing plane component (can be null if not found in BP).

### `void ToggleSlicer(float NewValue) const`

Toggles slicer enable state via the material parameter collection.

Parameters:
- `NewValue`: `0.0` = on/enabled, `1.0` = off/disabled.

### `void UpdateSlicerProperties() const`

Writes current plane transform to the MPC:
- `Position` is set from `PlaneComponent->GetComponentLocation()`.
- `Direction` is set from `PlaneComponent->GetUpVector()`.

## Protected methods

### `virtual void BeginPlay()`

Initialization flow:
1. Finds `PlaneComponent` and `SpringArmComponent` attached to BP_Slicer.
2. Captures initial `Rotation` from the spring arm.
3. Loads the MPC asset at `/Game/Slicer/SliceParameters.SliceParameters` and gets its world instance.
4. Reads saved values from `UCPP_GameInstance::GetSlicerValues(Position, Direction, DisablementState)`.
5. Applies saved `Position` to `SpringArmComponent->TargetArmLength` and yaw `Direction` to the spring arm.
6. Forces slicer off for this session start: `ToggleSlicer(1.0f)` (isSlicerOff = true).
7. Calls `UpdateSlicerProperties()` to push plane location and direction.

If required components or MPC fail to load, the slicer self-destroys to avoid runtime errors.

### `virtual void EndPlay(const EEndPlayReason::Type EndPlayReason)`

On actor removal, saves current slicer values back to the game instance:
- `Position` from `SpringArmComponent->TargetArmLength`
- `Direction` from `SpringArmComponent->GetComponentRotation().Yaw`
- `DisablementState` from MPC scalar `isSlicerOff`

Parameters:
- `EndPlayReason`: Reason why the actor is ending play.

## Protected properties

### `UStaticMeshComponent* PlaneComponent`

Visible slicing plane mesh.

### `USpringArmComponent* SpringArmComponent`

Controls slicer plane distance and orientation relative to the pivot.

### `UMaterialParameterCollectionInstance* MPC_Instance`

Runtime MPC instance used to update slicing parameters.

### `FRotator Rotation`

Initial rotation captured at BeginPlay; returned by `GetRotation()`.

### `UCPP_GameInstance* GameInstance`

Game instance reference used for saving and restoring slicer state (position, rotation yaw, and disablement flag).

## Notes

- If `PlaneComponent`, `SpringArmComponent`, MPC asset, or MPC instance are missing, the slicer destroys itself.
- Uses `GetWorld()->GetTimerManager().SetTimerForNextTick(...)` to defer property updates after spring arm length changes.
- Default session start state is OFF (disabled) regardless of previously saved disablement; position and rotation yaw are restored.
