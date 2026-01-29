# Digestive System Simulation

This section describes the digestive system particle flow simulation, which visualizes the movement of food/nutrients through the digestive tract.

## Overview

The digestive system simulation was created by adapting the blood flow simulation architecture. It reuses the concept of particles following splines but is **decoupled from the blood system** to allow independent evolution and customization. The digestive flow is slower than blood flow and uses different timing parameters.

## Architecture

The digestive system consists of **6 main files**:

### C++ Classes (Source Files)

| File | Location | Purpose |
|------|----------|---------|
| `CPP_DigestiveParticleSystemBase.h/.cpp` | `Source/VirtualAnatomy/.../DigestiveSystem/` | Base class for digestive particle systems |
| `CPP_DigestivePathSystem.h/.cpp` | `Source/VirtualAnatomy/.../DigestiveSystem/` | Particle system that follows a spline path |
| `CPP_DigestiveSystemSimulation.h/.cpp` | `Source/VirtualAnatomy/.../DigestiveSystem/` | Main actor that manages spline discovery and particle spawning |

### Content Assets

| File | Location | Purpose |
|------|----------|---------|
| `NS_DigestiveFlow.uasset` | `Content/Niagara/` | Niagara particle system for digestive particles |
| `BP_DigestiveSystemSimulation.uasset` | `Content/Blueprints/DigestiveSystemSimulation/` | Blueprint actor placed in the level |

## Class Hierarchy and Relationships

### Inheritance Hierarchy

```
AActor
   └── ACPP_DigestiveParticleSystemBase  (base class)
           └── ACPP_DigestivePathSystem  (derived class, follows splines)

AActor
   └── ACPP_DigestiveSystemSimulation    (main actor, spawns path systems)
```

> **Note:** `ACPP_DigestivePathSystem` **inherits from** `ACPP_DigestiveParticleSystemBase`. 
> The base class handles simulation event subscription, and the derived class adds spline-following behavior.

<!-- See accompanying diagrams: digestive-system-class-diagram.png, digestive-system-runtime.png, etc. -->

## How It Works

### 1. Initialization Flow

1. **BP_DigestiveSystemSimulation** is placed in the Explorer level (must have SplineComponents and DigestiveParticleSystem set)

2. In `BeginPlay()`, **ACPP_DigestiveSystemSimulation**:
   - Gets reference to the SimulationManager via `AnatomyUtils::GetSimulationManager(GetWorld())`
   - Calls `GetComponents<USplineComponent>()` to find all spline components on this actor
   - For each spline, calls `SpawnDigestiveParticle(SplineComp)` which:
     - Spawns a new `ACPP_DigestivePathSystem` actor in the world
     - Calls `Init(SimulationManager, Spline, DigestiveParticleSystem)` on it
   - Stores all spawned actors in `SpawnedDigestiveParticles` array

3. Each **ACPP_DigestivePathSystem** in its `Init()`:
   - Calls base class `ACPP_DigestiveParticleSystemBase::Init()` which subscribes to simulation events
   - **Manually copies spline points** from the source spline to its own `SplineToFollow` component (because components can't be shared between actors)
   - Sets actor scale to `2.0, 2.0, 2.0` (hardcoded)
   - Destroys the default DigestiveParticleComponent 
   - Uses `UNiagaraFunctionLibrary::SpawnSystemAttached()` to create a new Niagara component attached to the spline
   - The Niagara system's "Vein to follow" parameter uses "Attached Parent" mode to automatically reference the parent SplineComponent

### 2. Simulation Events

The digestive system responds to the same simulation manager events as other systems:

| Event | Handler | Behavior |
|-------|---------|----------|
| `StartSimulationEventDelegate` | `HandleSimulationStart()` | Activates particles, reinitializes the Niagara system |
| `StopSimulationEventDelegate` | `HandleSimulationEnd()` | Deactivates particles |
| `UpdateSimulationEventDelegate` | `HandleSimulationUpdate()` | Recalculates particle lifetime and speed |
| `ChangeDiagnosisEventDelegate` | `HandleDiagnosisChange()` | Updates parameters based on diagnosis |

### 3. Particle Movement Logic

Particles follow the spline using a custom **Scratch Pad module** in Niagara called `MakeParticleFollowSpline`. The logic works as follows:

```
Position along spline (U) = Particles.Age × Module.Speed
```

Where:
- `U` is a normalized value from 0 to 1 (start to end of spline)
- `Particles.Age` is how long the particle has existed (in seconds)
- `Module.Speed` is calculated as `1 / LifeTime`

This means when `Age = LifeTime`, `U = 1.0` and the particle reaches the end of the spline.

### 4. Lifetime and Speed Calculation

The C++ code in `SetDigestiveParticleLifeTime()` calculates particle parameters based on spline length and simulation speed:

```cpp
float SplineLength = SplineToFollow->GetSplineLength();
float AdjustedSpeed = Speed * DigestiveSpeedMultiplier; // 50.0f
float LifeTime = SplineLength / AdjustedSpeed;

// Speed should be 1/Lifetime so that when Age=Lifetime, U=1.0 (end of spline)
float NiagaraSpeed = 1.0f / LifeTime;

// Set Niagara parameters
DigestiveParticleComponent->SetVariableFloat(FName("LifeTime"), LifeTime);
DigestiveParticleComponent->SetVariableFloat(FName("Speed"), NiagaraSpeed);

// Also calculates a SpawnRate (but see bandaid fix below)
float ParticleCount = 5.0f;
float BurstDuration = LifeTime * 0.1f; // First 10% of journey
float SpawnRate = ParticleCount / BurstDuration;
DigestiveParticleComponent->SetVariableFloat(FName("SpawnRate"), SpawnRate);
```

The `DigestiveSpeedMultiplier` (50.0) makes digestive particles move slower than blood particles.

## NS_DigestiveFlow (Niagara System)

The Niagara system `NS_DigestiveFlow` was copied from `NS_Blood` and modified for digestive flow. Key components:

### Scratch Pad Module: MakeParticleFollowSpline

This custom module in the **Particle Update** stage:

**Inputs:**
- `Module.Vein to follow` - Spline Data Interface (auto-attached from parent)
- `Module.Speed` - Float (set by C++ code)
- `Particles.Age` - Float (built-in particle attribute)

**Logic:**
1. Multiplies `Speed × Age` to get `U` value (0-1)
2. Calls `SampleSplinePositionByUnitDistanceWS(Spline, U)` to get world position
3. Sets `Particles.Position` to the sampled position

**Output:**
- `Particles.Position` - Updated position along the spline

### Emitter Settings

| Setting | Value | Notes |
|---------|-------|-------|
| Life Cycle Mode | Self | Required for loop behavior |
| Loop Behavior | Infinite | Particles respawn when killed |
| Loop Duration | ~5 seconds | **Bandaid fix** - controls respawn timing |

### Spawn Configuration

> ⚠️ **KNOWN ISSUE / BANDAID FIX**
> 
> The C++ code calculates and sets a `SpawnRate` parameter, but the Niagara system **does not use it effectively**. Instead, the current implementation uses **4 Spawn Burst Instantaneous** modules with different spawn times (delays) to simulate staggered particle spawning. This is a workaround because proper continuous spawning with loop-based respawning was not achieved.
> 
> **Future improvement needed:** Either:
> - Make Niagara actually use the `SpawnRate` parameter from C++ with a Spawn Rate module
> - OR implement proper spawn logic where particles respawn at the beginning of the spline immediately after being killed at the end

Current spawn setup in Niagara (bandaid fix):
- Multiple `Spawn Burst Instantaneous` modules in Emitter Update
- Each with a different `Spawn Time` delay (e.g., 0s, 1s, 2s, 3s)
- Emitter Life Cycle Mode = Self
- Loop Behavior = Infinite
- Loop Duration = ~5 seconds (must roughly match particle lifetime)
- Creates batches of particles that travel together and repeat when loop restarts

### User Parameters (Set from C++)

| Parameter | Type | Set By |
|-----------|------|--------|
| `LifeTime` | Float | `SetDigestiveParticleLifeTime()` |
| `Speed` | Float | `SetDigestiveParticleLifeTime()` |
| `SpawnRate` | Float | `SetDigestiveParticleLifeTime()` |

## Setting Up the Blueprint

### BP_DigestiveSystemSimulation Setup

1. Create a Blueprint inheriting from `CPP_DigestiveSystemSimulation`
2. Add **Spline Components** as children to trace the digestive tract path
3. Set the `DigestiveParticleSystem` property to `NS_DigestiveFlow`

Structure:
```
BP_DigestiveSystemSimulation
├── DefaultSceneRoot
│   ├── SplineComponent_Esophagus
│   ├── SplineComponent_Stomach
│   ├── SplineComponent_SmallIntestine
│   └── SplineComponent_LargeIntestine
```

### Spline Creation Tips

- Draw splines to trace the path food would take through the digestive system
- Start points should be at the entry of each segment (mouth, stomach entrance, etc.)
- End points at the exit of each segment
- The system will automatically find and use all splines on the actor

## Integration with Simulation Manager

The digestive system integrates with the existing simulation infrastructure:

```cpp
// Subscribe to events in Init()
SimulationManager->StartSimulationEventDelegate.AddUObject(this, &HandleSimulationStart);
SimulationManager->StopSimulationEventDelegate.AddUObject(this, &HandleSimulationEnd);
SimulationManager->UpdateSimulationEventDelegate.AddUObject(this, &HandleSimulationUpdate);
SimulationManager->ChangeDiagnosisEventDelegate.AddUObject(this, &HandleDiagnosisChange);
```

> **IMPORTANT:** Always unsubscribe from events in `EndPlay()` and `BeginDestroy()` to prevent crashes. The base class handles this automatically.

## Differences from Blood Flow Simulation

| Aspect | Blood Flow | Digestive Flow |
|--------|------------|----------------|
| Speed Multiplier | Higher (faster) | 50.0 (slower) |
| Parent Blueprint Location | CirculatorySystem folder | DigestiveSystemSimulation folder |
| Niagara System | NS_Blood | NS_DigestiveFlow |
| C++ Base Class | CPP_ArteriesBloodFlowSimulation | CPP_DigestiveParticleSystemBase |
| Timing | Tied to heartbeat | Independent timing |

## Known Issues and Future Improvements

### Current Issues

1. **Bandaid Spawn Fix**: The spawn system uses multiple burst modules with delays instead of proper continuous spawning. Particles don't smoothly respawn when killed.

2. **Single Particle Appearance**: Despite spawn count settings, sometimes only one particle appears visible due to timing/synchronization issues.

3. **Loop Duration Dependency**: The respawn behavior depends on matching Loop Duration with particle lifetime, which can desync.

### Recommended Future Improvements

1. **Proper Continuous Flow**: Implement spawn logic where:
   - Particles are continuously spawned at a rate
   - When a particle reaches the end of the spline (U >= 1), it gets killed
   - New particles spawn to replace killed ones
   - Creates seamless flowing animation

2. **Decouple from Loop Duration**: Remove dependency on emitter loop duration for respawn timing.

3. **Peristalsis Animation**: Add wave-like motion to simulate intestinal peristalsis (the DigestivePulse() function exists but is not fully implemented).

4. **Per-Segment Speed**: Different speeds for different parts of the digestive system (faster in esophagus, slower in stomach, etc.).

## File Locations Summary

```
Virtual-Anatomy-UE/
├── Source/VirtualAnatomy/
│   ├── Public/Simulation/DigestiveSystem/
│   │   ├── CPP_DigestiveParticleSystemBase.h
│   │   ├── CPP_DigestivePathSystem.h
│   │   └── CPP_DigestiveSystemSimulation.h
│   └── Private/Simulation/DigestiveSystem/
│       ├── CPP_DigestiveParticleSystemBase.cpp
│       ├── CPP_DigestivePathSystem.cpp
│       └── CPP_DigestiveSystemSimulation.cpp
├── Content/
│   ├── Niagara/
│   │   └── NS_DigestiveFlow.uasset
│   └── Blueprints/DigestiveSystemSimulation/
│       └── BP_DigestiveSystemSimulation.uasset
```

## Related Documentation

- [Blood Flow Simulation](BloodFlowSimulaiton.md) - Original blood simulation (similar architecture)
- [General Simulation Overview](GeneralOverview.md) - Event-driven simulation architecture

