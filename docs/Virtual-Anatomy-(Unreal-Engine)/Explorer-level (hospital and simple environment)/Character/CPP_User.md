# ACPP_User

## Class description

Handles user input and camera control. Creates camera objects and spring arm components when the application starts. `BP_User` inherits from this class, allowing parameter configuration in the Unreal Editor.

Inherits from `ACharacter`, so the player controls a character in the scene.

Owns and initializes:
- `UCPP_CameraControls` for orbit camera logic
- `UMeshSelector` for mesh highlighting
- `FRayCaster` for precise click detection

## Public fields

### `ACPP_User()`

Constructor sets up camera, spring arm, and owned components.

- Creates spring arm (Camera Boom) and attaches to root component:

```c++
CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("Camera Boom"));
CameraBoom->SetupAttachment(RootComponent);
```

- Creates camera and attaches to the spring arm socket:

```c++
MainCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("Main camera"));
MainCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
```

- Creates components used at runtime:
  - `CameraControls = CreateDefaultSubobject<UCPP_CameraControls>(...)`
  - `MeshSelector   = CreateDefaultSubobject<UMeshSelector>(...)`
  - `UserRayCaster  = MakeUnique<FRayCaster>()`

Default speeds set in constructor: `CameraSpeed = 0.5f`, `ZoomSpeed = 20.0f`.

If you open `BP_User` (full Blueprint editor), you can see the spring arm and camera in the viewport.

![camera and user in view port](https://jrcz-data-science-lab.github.io/VirtualAnatomy-Documentation/images/camera-and-spring-arm-in-view-port.png)

### `AActor* TargetActor`

> `UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Target Actor")`

Invisible actor that serves as the camera pivot (orbit center). The camera always looks at this actor and orbits around it.

In `BeginPlay()`, it is found by searching for actors with the tag `"CenterPointTag"` (the last item in that list is used).

When a mesh is isolated, the Mesh Isolator moves `TargetActor` to the selected mesh center to re-center the pivot. See [Mesh Isolator](../Mesh-Isolator/MeshIsolator.md).

### `USpringArmComponent* CameraBoom;`
> `UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Camera)`

Spring arm controlling camera offset from the character; attached to the root component.

### `UCameraComponent* MainCamera;`
> `UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = Camera)`

Main camera component attached to the spring arm.

### `UCPP_CameraControls* GetCameraControls() const`

Returns the camera controls component used for orbit camera math, zooming, and panning. See [Camera Controls](../Camera/Camera.md).

### `UMeshSelector* GetMeshSelector() const`

Returns the mesh selector used to highlight the selected mesh. See [UMeshSelector](../Mesh-Selection/Mesh-selector/UMeshSelector.md).

### `void SmoothlyMoveToTheSelectedActorOnClick(const AActor* SelectedActor = nullptr)`

Moves the character (and thus the camera) smoothly to the selected actor’s location and rotation.

Implementation uses `UKismetSystemLibrary::MoveComponentTo` on the root component with a short duration (~0.2s), easing and sweeping enabled.

Parameter:
- `SelectedActor` — the actor to move toward; must be non-null.

### `void MoveHorizontal(const FInputActionValue& Value)`

Pans the camera pivot (LookAt position) along world X.

### `void MoveVertical(const FInputActionValue& Value)`

Pans the camera pivot (LookAt position) along world Z.

### `void MoveWithMouse(const FInputActionValue& Value)`

Pans the camera on X/Z using mouse drag input (maps to `CameraControls->MoveHorizontal/MoveVertical`).

### `void RotateHorizontal(const FInputActionValue& Value)`

Rotates the camera horizontally (azimuth) using keyboard input (A/D).

### `void RotateVertical(const FInputActionValue& Value)`

Rotates the camera vertically (polar) using keyboard input (W/S).

### `void Zoom(const FInputActionValue& Value)`

Zooms in/out by changing the spherical radius; clamped to the min/max radius configured in camera controls.

### `APlayerController* GetUserPlayerController()`

Returns the player controller.

### `UQuizUIManager* QuizUIManagerRef`

Pointer to the quiz UI manager used to display quiz UI.

### `void Tick(float DeltaTime)`

Stores frame delta time for frame rate independent movement.

----

## Protected fields

### `void BeginPlay()`

Initialization sequence:
1. Sets input mode to GameAndUI (cursor visible, no mouse lock).
2. Adds Enhanced Input mapping context.
3. Initializes `MeshSelector` with world context.
4. Initializes `UserRayCaster` and adds ignored actors.
5. Finds `TargetActor` by tag `"CenterPointTag"`.
6. Loads camera sensitivity settings from `UCPP_GameInstance`.
7. Initializes `CameraControls` using `TargetActor` location and starting position `FVector(500, 0, 500)`.
8. Sets actor location and rotation to match camera.
9. Ensures IsolateMesh is initialized via `AnatomyUtils::GetIsolateMeshInstance(GetWorld())`.

### `void Click()`

Executed on each mouse click.

Workflow:
1. Get mouse position from player controller.
2. Deproject screen position to world (origin + direction ray).
3. Use `UserRayCaster->TraceLine` for precise hit test (max length ~1000 units).
4. On hit: retrieve `UMeshComponent*` via `UserRayCaster->GetCurrentHitComponent()` and call `MeshSelector->HighlightComponent(...)`.
5. On miss: call `MeshSelector->DeselectAllComponents()`.

See [FRayCaster](../Mesh-Selection/Raycasting/How-is-raycasting-done.md).

### `void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)`

Binds Enhanced Input actions:
- `IA_Rotate`         → `MoveWithMouse` (mouse drag for panning X/Z)
- `IA_Zoom`           → `Zoom` (mouse wheel for zoom)
- `IA_Click`          → `Click` (mouse click selection)
- `IA_MoveHorizontal` → `RotateHorizontal` (A/D for horizontal rotation)
- `IA_MoveVertical`   → `RotateVertical` (W/S for vertical rotation)

## Protected member fields

### `UInputMappingContext* InputMapping;`
> `UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "EnhancedInput", meta = (AllowPrivateAccess = "true"))`

Configured in `BP_User`; defines the input mapping used for this character.

### `UInputAction* IA_Rotate, IA_Zoom, IA_Click, IA_MoveHorizontal, IA_MoveVertical`
> `UPROPERTY(EditAnywhere, BlueprintReadOnly, Category= "EnhancedInput")`

Enhanced Input actions this character binds at runtime.

### `float CameraSpeed`
> `UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Camera)`

Camera pan/rotate speed (loaded from game instance at BeginPlay).

### `float ZoomSpeed`
> `UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = Camera)`

Camera zoom speed (loaded from game instance at BeginPlay).

## Private members

### `float InternalDeltaTime`

Time between ticks; used for frame rate independent movement.

### `TUniquePtr<FRayCaster> UserRayCaster`

Unique pointer for the ray caster used to determine which mesh the user clicked. Ray casting is chosen over pixel picking to avoid an extra scene pass.

### `UPROPERTY() UCPP_CameraControls* CameraControls`

Pointer to camera controls (spherical orbit, zoom, panning). See [Camera Controls](../Camera/Camera.md).

### `UPROPERTY() UMeshSelector* MeshSelector`

Pointer to mesh selector (highlight selected meshes). See [UMeshSelector](../Mesh-Selection/Mesh-selector/UMeshSelector.md).
