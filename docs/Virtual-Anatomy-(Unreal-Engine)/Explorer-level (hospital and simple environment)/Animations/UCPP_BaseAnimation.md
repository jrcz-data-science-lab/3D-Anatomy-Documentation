# UCPP_BaseAnimation

This class serves as the base class for all potential animations in the game. It is designed to be inherited by other C++ classes, not directly by Blueprints. Its primary purpose is to start or stop animations in response to events fired by the `SimulationManager` class. This class may also be extended to hold various parameters in the future.

## Protected Properties

### `TObjectPtr<UCPP_SimulationManager> SimulationManager`

**Description**: 

An instance of the `SimulationManager`, allowing the animation to listen for events triggered by it.

### `TObjectPtr<UAnimSequence> AnimationOfHumanBody`

**Description**: 

The animation sequence that will play when the simulation starts.

## Protected Methods

### `void OnStartSimulation()`

**Description**: 

Executed when the simulation start event is triggered by the `SimulationManager`. The base implementation plays `AnimationOfHumanBody` in a loop on the owning skeletal mesh.

### `void OnStopSimulation()`

**Description**: 

Executed when the simulation end event is triggered by the `SimulationManager`. The base implementation stops the current animation on the owning skeletal mesh.

### `virtual void OnSimulationUpdate(const FSimulationSlideBarsParameters& UpdatedParameters)`

**Description**:

Virtual hook called whenever the simulation parameters (sliders) are updated. The base implementation does nothing; derived animation classes can override this to adjust animation behavior in real time (e.g., change playback speed based on BPM).

### `virtual void OnDiagnosisChange(UCPP_Diagnosis& selectedDiagnosis)`

**Description**:

Virtual hook called whenever the active diagnosis changes. The base implementation does nothing; derived animation classes can override this to react to diagnosis-specific parameters from `selectedDiagnosis.GetSimulationParameters()`.

## Public Methods

### `void NativeBeginPlay()`

**Description**: 

An overridden method that subscribes the animation instance to events from the `SimulationManager`.

### `void BeginDestroy()`

**Description**: 

An overridden method for any cleanup needed when the animation instance is destroyed.