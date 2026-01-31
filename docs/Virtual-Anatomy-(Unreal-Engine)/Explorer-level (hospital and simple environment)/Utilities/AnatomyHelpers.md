# AnatomyUtils 

A namespace containing utility functions for operations related to anatomy exploration in the context of a game or simulation. These functions are specifically designed to work reliably at the explorer level. While most functions are robust, some may return `nullptr` under specific conditions, which requires caution during use.

---

## Public Methods 

### `AActor* GetActorByName(const FString& ActorName, const UWorld* World)`

**Description**:  
Retrieves an actor from the world by its name.  

**Parameters**:  
- `ActorName`: The name of the actor to search for (constant reference).  
- `World`: Pointer to the world where the search should occur (non-owning; may be `nullptr`).  

**Returns**:  
A pointer to the found actor. If the actor is not found or `World` is invalid, the function returns `nullptr`. Exercise caution when handling `nullptr` results.  

---

### `AActor* GetRandomActor(const UWorld* World)`

**Description**:  
Selects and retrieves a random actor from the scene.  

**Parameters**:  
- `World`: Pointer to the world from which the actor will be retrieved.  

**Returns**:  
A pointer to a random actor. If the scene is empty or `World` is invalid, the function returns `nullptr`.  

---

### `TObjectPtr<UCPP_SimulationManager> GetSimulationManager(const UWorld* World)`

**Description**:  
Retrieves an instance of the simulation manager from the `GameMode` associated with the provided world.  

**Parameters**:  
- `World`: The world in which the `GameMode` contains the simulation manager instance.  

**Returns**:  
A `TObjectPtr<UCPP_SimulationManager>` pointing to the simulation manager. If the simulation manager is not present in the `GameMode` or `World` is invalid, the function returns `nullptr`.  

---

### `TObjectPtr<UCPP_IsolateMesh> GetIsolateMeshInstance(const UWorld* World)`

**Description**:  
Obtains an instance of the mesh isolator class from the `GameMode`.  

**Parameters**:  
- `World`: The world in which the `GameMode` contains the mesh isolator instance.  

**Returns**:  
A `TObjectPtr<UCPP_IsolateMesh>` pointing to the mesh isolator class. If the instance is not present in the `GameMode` or `World` is invalid, the function returns `nullptr`.  

---

### `TObjectPtr<UMeshSelector> GetMeshSelector(const UWorld* World)`

**Description**:  
Retrieves the mesh selector instance from the user character in the specified world.  

**Parameters**:  
- `World`: The world containing the user actor (`ACPP_User`) with a mesh selector component/instance.  

**Returns**:  
A `TObjectPtr<UMeshSelector>` pointing to the mesh selector instance. If the world does not contain an instance of `ACPP_User` or the user lacks a mesh selector, the function returns `nullptr`.  

---

### `float ConvertBeatsPerMinuteToBeatsPerSecond(const float& BeatsPerMinute, float MaxPossibleBPS = 5)`

**Description**:  
Converts beats per minute (BPM) to beats per second (BPS) and scales it according to an upper bound. Used primarily for blood simulation heartbeat timing.  

**Parameters**:  
- `BeatsPerMinute`: Heart rate in BPM.  
- `MaxPossibleBPS`: Maximum beats per second used as a scaling reference. Default value is 5 (based on 300 BPM / 60s = 5 BPS).  

**Returns**:  
The BPS value used to configure how often `ACPP_BloodParticleSystemBase` (and derived blood systems) should trigger their heartbeat logic.  

---

### `float ConvertDegreesToRadians(float Angle)`

**Description**:  
Converts an angle from degrees to radians.  

**Parameters**:  
- `Angle`: The angle in degrees.  

**Returns**:  
The angle converted to radians.  

---

### `ACPP_User* GetUser(const UWorld* World)`

**Description**:  
Retrieves the user actor from the world (typically the `ACPP_User` pawn controlling the camera and interaction).  

**Parameters**:  
- `World`: The world containing the `ACPP_User` instance.  

**Returns**:  
A pointer to the `ACPP_User` actor. If the world does not contain an `ACPP_User`, the function returns `nullptr`.  

---

## Notes

- Always check pointers returned by these functions for `nullptr` before use to avoid null dereferences.  
- These utilities are intended for operations within the explorer level and assume that the level is set up with the expected `GameMode`, user actor, and managers.  
- `ConvertBeatsPerMinuteToBeatsPerSecond` is used extensively by blood simulation components to convert diagnosis/slider BPM values into heartbeat timer intervals.
