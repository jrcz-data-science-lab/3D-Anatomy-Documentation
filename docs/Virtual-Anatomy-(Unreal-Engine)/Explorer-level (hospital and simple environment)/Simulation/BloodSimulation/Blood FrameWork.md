# Blood framework

This page describes the framework behind the Niagara-based blood systems and how they integrate with the Simulation Manager.

## Overview

There are multiple actors representing blood in the simulation, such as:
- Blood following specific arterial/venous paths.
- Blood squirting from the body during hypovolemic shock.

All of these actors are connected to the `UCPP_SimulationManager` and listen to the same set of simulation events (start, stop, slider updates, and diagnosis changes).

To avoid duplicating logic and to follow the **DRY (Don't Repeat Yourself)** principle, a small framework was created:
- A single base class, `ACPP_BloodParticleSystemBase`, implements all shared behavior:
  - Wiring to Simulation Manager delegates.
  - Heartbeat scheduling from BPM.
  - Default Niagara parameter updates and safe reinitialization.
  - Safe teardown and null-guards.
- Concrete blood actors derive from this base class and implement only their specific behavior, such as:
  - `ACPP_BloodPathSystem` – blood flowing along splines in the circulatory system.
  - `ACPP_RupturedArtery` – a local Niagara squirt effect driven by diagnosis type.

## Base class: `ACPP_BloodParticleSystemBase`

### Responsibilities

- Initialization via `Init` wires Simulation Manager delegates:
  - Start → `HandleSimulationStart()`
  - Stop  → `HandleSimulationEnd()`
  - Update → `HandleSimulationUpdate(const FSimulationSlideBarsParameters&)`
  - ChangeDiagnosis → `HandleDiagnosisChange(UCPP_Diagnosis&)`
- Schedules a heartbeat timer from BPM via:
  - `SetHearthBeatInterval(AnatomyUtils::ConvertBeatsPerMinuteToBeatsPerSecond(...))`
- Default simulation update handler `HandleSimulationUpdate`:
  - Guards against invalid state (`this`, shutdown flag, SimulationManager, BloodParticleComponent, Niagara readiness).
  - Defers updates by one tick when the Niagara instance isn’t ready (`SetTimerForNextTick`).
  - Recalculates the heartbeat interval from current BPM.
  - Sets Niagara variables `Speed` and `BloodThickness` from the updated parameters.
  - Calls `BloodParticleComponent->ReinitializeSystem()`.
- Diagnosis handler `HandleDiagnosisChange`:
  - Guards against invalid actor, invalid SimulationManager, shutdown state, and invalid BloodParticleComponent.
  - Recalculates the heartbeat interval from the diagnosis’s BPM.
  - Reads `FSimulationSlideBarsParameters` via `selectedDiagnosis.GetSimulationParameters()`.
  - Updates Niagara `Speed` and `BloodThickness` (base does **not** call `ReinitializeSystem()` here so that overrides can add their own logic before/after reinit).
- Safe teardown in `BeginDestroy` / `EndPlay`:
  - Sets `bIsShuttingDown = true`.
  - Clears the heartbeat timer.
  - Unbinds all Simulation Manager delegates using `RemoveAll(this)`.
- Base tick is disabled (`PrimaryActorTick.bCanEverTick = false`).

### Public API (C++)

- `void Init(UCPP_SimulationManager* SimManager, UNiagaraSystem* Blood)`
- `UFUNCTION(BlueprintCallable) void Init(UCPP_SimulationManager* SimManager)`
- `void SetHearthBeatInterval(float interval)`
- `virtual void Tick(float DeltaTime)` (present but unused in base; tick is off by default)

### Notes on safety and deferral

- Guard checks throughout on:
  - `IsValid(this)` and a shutdown flag `bIsShuttingDown`.
  - `SimulationManager` pointer validity.
  - `BloodParticleComponent` validity, registration state, and presence of a Niagara system instance.
- If the Niagara system instance is not yet created, `HandleSimulationUpdate` logs a warning and defers updates by one tick using `SetTimerForNextTick`, then retries.

## `ACPP_BloodPathSystem`

### Role

Represents blood following an artery/vein spline. It duplicates spline data into a local `USplineComponent` and attaches a Niagara system to it.

### Components

- `USplineComponent* SplineToFollow` – local copy of source spline points.
- `UNiagaraComponent* BloodParticleComponent` attached to `SplineToFollow`.

### Initialization

- `Init(UCPP_SimulationManager* SimManager, USplineComponent* Spline, UNiagaraSystem* Blood)`:
  - Calls `ACPP_BloodParticleSystemBase::Init(SimManager, Blood)` to wire delegates and initial heartbeat.
  - Copies spline points from the source `Spline` into `SplineToFollow` (world-space positions).
  - Sets actor scale and stores `BloodNiagaraSystem`.
  - Destroys any existing `BloodParticleComponent`.
  - Spawns a new Niagara system attached to `SplineToFollow` using `UNiagaraFunctionLibrary::SpawnSystemAttached`.

### Runtime behavior

- `HandleSimulationStart`:
  - Calls base `HandleSimulationStart` (recalculates heartbeat interval from BPM).
  - Calls `SetBloodParticleLifeTime(SimulationManager->GetSimulationParameters()->Speed)` to configure Niagara `LifeTime` based on spline length and speed.

- `HandleSimulationUpdate(const FSimulationSlideBarsParameters& UpdatedParameters)`:
  - Calls `Super::HandleSimulationUpdate(UpdatedParameters)` to:
    - Guard state.
    - Update `Speed` / `BloodThickness`.
    - Recalculate heartbeat interval.
    - Call `ReinitializeSystem()`.
  - If actor and `BloodParticleComponent` are still valid, recomputes `LifeTime = SplineLength / (Speed * 220)` and sets it on the Niagara system.
  - If the particle system is active, calls `ReinitializeSystem()` again to apply the updated `LifeTime`.

- `HandleDiagnosisChange(UCPP_Diagnosis& selectedDiagnosis)`:
  - Logs the diagnosis change.
  - If `BloodParticleComponent` is valid, calls `HandleSimulationUpdate(*selectedDiagnosis.GetSimulationParameters())` to reuse the full update logic (including reinit from the base).
  - If the particle system is active, calls `ReinitializeSystem()` again after the update.

- `HeartBeat`:
  - On each scheduled beat, checks `SimulationManager->GetIsSimulationRunign()` and activates `BloodParticleComponent` if true.

- Tick:
  - `PrimaryActorTick.bCanEverTick = true`.
  - Current `Tick` override simply calls `Super::Tick(DeltaTime)`; all meaningful behavior is event- and heartbeat-driven.

## `ACPP_RupturedArtery`

### Role

Represents a local Niagara effect (e.g., a squirting ruptured artery) that activates only for specific diagnoses.

### Components

- A `UNiagaraComponent* BloodParticleComponent` attached to the root component.

### Initialization

- In `BeginPlay`:
  - Calls `Init(AnatomyUtils::GetSimulationManager(GetWorld()))` to wire delegates and heartbeat.
  - Immediately calls `BloodParticleComponent->Deactivate()` so the effect is off by default.

### Runtime behavior

- `HandleDiagnosisChange(UCPP_Diagnosis& selectedDiagnosis)`:
  - Uses `SimulationManager->CompareDiseaseTypes(EDiagnosisType::HypovolemicShock)` to check the active diagnosis.
  - Activates `BloodParticleComponent` when hypovolemic shock is selected.
  - Deactivates `BloodParticleComponent` for all other diagnoses.

- `HandleSimulationUpdate(const FSimulationSlideBarsParameters& UpdatedParameters)`:
  - Intentionally left empty.
  - This avoids reinitializing the Niagara system on slider changes, which would otherwise cause the blood effect to restart/squirt unintentionally.

- `HandleSimulationEnd()`:
  - Deactivates `BloodParticleComponent` when the simulation stops.

- Tick:
  - `PrimaryActorTick.bCanEverTick = false`.

## Event flow summary

- `UCPP_SimulationManager` broadcasts:
  - Start: `FStartSimulation` – all blood actors receive `HandleSimulationStart`.
  - Stop: `FStopSimulation`  – all blood actors receive `HandleSimulationEnd`.
  - Update: `FUpdateSimulation(const FSimulationSlideBarsParameters&)` – drives `HandleSimulationUpdate` with the latest slider parameters.
  - ChangeDiagnosis: `FChangeDiagnosis(UCPP_Diagnosis&)` – drives `HandleDiagnosisChange`.
- Base `ACPP_BloodParticleSystemBase`:
  - Handles common logic for updates and heartbeat.
  - Reinitializes Niagara safely on slider updates when the system is ready.
- `ACPP_BloodPathSystem`:
  - Uses both diagnosis and slider updates to refresh speed, thickness, and lifetime, reinitializing as needed.
- `ACPP_RupturedArtery`:
  - Reacts only to diagnosis type (and start/stop) and ignores slider updates to keep behavior stable.
