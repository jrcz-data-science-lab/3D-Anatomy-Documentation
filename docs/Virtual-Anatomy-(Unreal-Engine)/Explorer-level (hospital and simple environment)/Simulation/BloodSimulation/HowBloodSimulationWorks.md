# Blood Simulation

**NOTE**: This section assumes you are at least somewhat familiar with the inner workings of the application and the Niagara system.

## General Idea

This section discusses how the blood simulation is created and designed to account for various parameters and user selections.

The idea of blood in this project is straightforward. Blood is defined as a particle system using Niagara. There are countless resources online that demonstrate how to work with it. Essentially, you can define various parameters of the particle system (e.g., speed, velocity, and gravity). By clicking the **+** button in the visual Niagara editor, you can set these parameters, and Unreal Engine (UE) handles the rest. This results in a working particle simulation that can collide with the environment.

The core functionality relies on a circulatory system (defined in `BP_CirculatorySystem`) that includes:
- Picker mesh for veins 
- Picker mesh for arteries
- A set of splines representing arteries and veins
- Rendering mesh for arteries
- Rendering mesh for veins 

For our purposes, the set of splines is of primary interest. To make the particles follow the spline, we retrieve the spline and pass it to the Niagara system so the particles know their path. This is achieved by assigning the Niagara particle system as a child of the spline, as shown below:

```
RootComponent
 -> SplineComponent
    -> NiagaraSystemComponent
```

To make the Niagara system recognize this structure, we set up a **User-Defined Parameter** of type `Spline`.

<figure markdown="span">
  ![User Parameters](https://jrcz-data-science-lab.github.io/VirtualAnatomy-Documentation/images/user-params-niagara.png)
  <figcaption>User-Defined Parameters in the Niagara System Editor</figcaption>
</figure>

## User Parameters 

With the help of these parameters, we can not only pass splines that the blood should follow but also change the behavior of the Niagara system at runtime. For example, this allows us to modify the speed of the blood particles dynamically.

## Making Particles Follow the Spline 

This process is simple. We define that the simulation should be altered using a custom blueprint. This blueprint takes the age of the particle and moves it along the spline based on its age. Age was chosen as it increases consistently, which makes it suitable for driving the particle along the curve.

This is illustrated in the figure below:

<figure markdown="span">
  ![Blood Particle System](https://jrcz-data-science-lab.github.io/VirtualAnatomy-Documentation/images/blood-particle-system.png)
  <figcaption>User-Defined Parameters in the Niagara System Editor</figcaption>
</figure>

# Spawning Particles 

## Retrieving Splines 

Retrieving splines is not as straightforward as one might expect. To spawn particles, we need to define a spline that can be retrieved from `BP_CirculatorySystem`. Due to the constraints of Unreal Engine, it is impossible to share components between different actors. For instance, if one actor manages the particle system and another contains the splines, passing the spline directly is problematic.

When we pass the spline component as an argument, it becomes `null` for reasons currently unknown. A new spline is then created from scratch, resulting in a straight line.

To address this, we duplicate the spline using C++ and copy the point data from the original spline to the new one. This ensures the splines defined in `BP_CirculatorySystem` can be used with the Niagara system.

The implementation of this solution is shown below:

```c++
void ACPP_BloodPathSystem::Init(UCPP_SimulationManager* SimManager, USplineComponent* Spline,
	UNiagaraSystem* Blood)
{
	// Call Init of the base class
	ACPP_BloodParticleSystemBase::Init(SimManager, Blood);

	if (Spline)
	{
		// Since components cannot be shared between actors, recreate the component manually
		TArray<FVector> SplinePoints;
		for (int i = 0; i < Spline->GetNumberOfSplinePoints(); i++)
		{
			SplinePoints.Push(Spline->GetSplinePointAt(i, ESplineCoordinateSpace::World).Position);
		}

		SplineToFollow->SetSplinePoints(SplinePoints, ESplineCoordinateSpace::World);
		SetActorScale3D(FVector(2.0f, 2.0f, 2.0f));

		BloodNiagaraSystem = Blood;

    		// Destroy the existing component before creating a new one
		if (BloodParticleComponent)
		{
			BloodParticleComponent->DestroyComponent();
			BloodParticleComponent = nullptr;
		}

		// Create new attached Niagara system instance
		BloodParticleComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(
			BloodNiagaraSystem,
			SplineToFollow,
			NAME_None,
			FVector(0.f),
			FRotator(0.f),
			EAttachLocation::KeepRelativeOffset,
			false
		);
	}
}
```

## Spawning Actors 

The particle system is represented by an actor structured as follows:

```c++
=====================================================================
 *          ACPP_BloodPathSystem
 *
 * Root # default
 * |_Spline (spline to follow, supplied by BP_CirculatorySystem)
 *   |_ NiagaraSystem (blood visualization)
 *
=====================================================================
```

This actor is not manually placed in the world. Instead, `CPP_ArteriesBloodFlowSimulation` iterates over the splines it contains and creates an actor for each spline. All logic for `ACPP_BloodPathSystem` is encapsulated within the actor itself.

```c++
// BeginPlay of ACPP_ArteriesBloodFlowSimulation class

// Get all splines from skeletal mesh children
TArray<USceneComponent*> TempSplines;
ArteriesSkeletalMesh->GetChildrenComponents(false, TempSplines);
for (auto& spline : TempSplines)
{
    // Cast the spline from USceneComponent to USplineComponent
    if (auto s = Cast<USplineComponent>(spline))
    {
        // Create an actor representing blood and store it for future reference
        SpawnedBloodParticles.Add(SpawnBloodParticle(s));
        UE_LOG(LogTemp, Display, TEXT("Spline in the arteries component was found"));
    }
}
```

## Runtime updates and reinitialization

- **Slider updates**  
  `ACPP_BloodParticleSystemBase::HandleSimulationUpdate(const FSimulationSlideBarsParameters&)`:
  - Guards against invalid actor, shutdown state, invalid `SimulationManager`, or invalid/unregistered `BloodParticleComponent`.
  - If the Niagara system instance is not ready yet, defers the update with `SetTimerForNextTick` and retries.
  - Recalculates the heartbeat interval from the current BPM.
  - Sets Niagara variables `Speed` and `BloodThickness` from `UpdatedParameters`.
  - Calls `BloodParticleComponent->ReinitializeSystem()` so changes apply immediately.

- **Diagnosis change**  
  `ACPP_BloodParticleSystemBase::HandleDiagnosisChange(UCPP_Diagnosis&)`:
  - Guards against invalid actor, shutdown, or invalid `SimulationManager` / `BloodParticleComponent`.
  - Recalculates heartbeat interval from the current BPM using diagnosis parameters.
  - Reads `FSimulationSlideBarsParameters` from `selectedDiagnosis.GetSimulationParameters()`.
  - Updates Niagara `Speed` and `BloodThickness` (does **not** reinitialize in the base class).

- **Path system specifics**  
  `ACPP_BloodPathSystem` extends the base behavior:
  - `HandleSimulationUpdate`:
    - Calls `Super::HandleSimulationUpdate(UpdatedParameters)` to apply speed/thickness and reinitialize.
    - If the actor and `BloodParticleComponent` are valid, computes 
      `LifeTime = SplineLength / (Speed * 220)` and writes it to Niagara.
    - If the system is active, calls `ReinitializeSystem()` again.
  - `HandleDiagnosisChange`:
    - Logs the diagnosis change.
    - If the particle component is valid, calls `HandleSimulationUpdate(*selectedDiagnosis.GetSimulationParameters())`.
    - If the system is active, calls `ReinitializeSystem()` again.

- **Ruptured artery specifics**  
  `ACPP_RupturedArtery`:
  - In `BeginPlay`, calls `Init(AnatomyUtils::GetSimulationManager(GetWorld()))` and deactivates its Niagara component.
  - `HandleDiagnosisChange` checks `SimulationManager->CompareDiseaseTypes(EDiagnosisType::HypovolemicShock)`:
    - Activates the component for hypovolemic shock.
    - Deactivates it for any other diagnosis.
  - `HandleSimulationUpdate` intentionally does nothing (avoids reinitialization-based squirting when sliders move).
  - `HandleSimulationEnd` deactivates the Niagara component.

## Heartbeat and ticking

- **Heartbeat frequency**  
  Derived from BPM via `AnatomyUtils::ConvertBeatsPerMinuteToBeatsPerSecond(...)` and applied with
  `SetHearthBeatInterval(...)`. Heartbeat is recalculated:
  - During initialization in `Init`.
  - On simulation start via `HandleSimulationStart`.
  - On slider updates via `HandleSimulationUpdate`.
  - On diagnosis changes via `HandleDiagnosisChange`.

- **Per-beat behavior**  
  - Base `ACPP_BloodParticleSystemBase::HeartBeat` only guards against invalid state and stops the timer if needed.
  - `ACPP_BloodPathSystem::HeartBeat` activates the Niagara component when `SimulationManager->GetIsSimulationRunign()` is true and the component exists.

- **Per-frame Tick**  
  - Base class:
    - `PrimaryActorTick.bCanEverTick = false`.
    - `Tick` does nothing beyond `Super::Tick`.
  - `ACPP_BloodPathSystem`:
    - `PrimaryActorTick.bCanEverTick = true`.
    - `Tick` calls `Super::Tick` only; all visual behavior is event/heartbeat-driven.
  - `ACPP_RupturedArtery`:
    - `PrimaryActorTick.bCanEverTick = false`.

## Null-guards and deferred updates

- All handlers in `ACPP_BloodParticleSystemBase` and derived classes use guard checks to avoid crashes when actors/components are destroyed or the world is shutting down:
  - `IsValid(this)` and `bIsShuttingDown`.
  - `SimulationManager` validity.
  - `BloodParticleComponent` validity and registration.
  - A valid Niagara system instance (`GetSystemInstanceController()`).
- If the Niagara system instance is not ready yet in `HandleSimulationUpdate`, the update is deferred by one tick using
  `GetWorld()->GetTimerManager().SetTimerForNextTick(...)` and retried.

## Variables used by Niagara

- **Base class**: `Speed`, `BloodThickness`.
- **Path system**: additionally `LifeTime` (computed from spline length and current speed).

## Spawning overview (runtime)

- `ACPP_ArteriesBloodFlowSimulation::BeginPlay()` gathers child components of the arteries skeletal mesh, filters `USplineComponent` instances, and for each calls `SpawnBloodParticle(spline)`.
- Each spawned `ACPP_BloodPathSystem` executes `Init(SimulationManager, Spline, BloodNiagaraSystem)` and manages its own Niagara instance according to the rules described above.
