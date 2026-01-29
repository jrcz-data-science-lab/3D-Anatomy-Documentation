# ⚙️ How Everything Connects at Runtime

## 🧠 Overview

This document explains how all systems — diagnosis selection, behavior execution, indicator spawning, FX, temperature feedback, and parameter updates — are **coordinated at runtime**.

The central class is `UCPP_SimulationManager`. It owns the simulation state and delegates specific responsibilities to modular subsystems:
- `UCPP_Diagnosis` for static diagnosis data
- `UShockBehavior` for diagnosis-specific logic
- Niagara FX, breathing dot indicators, and floating temperature displays for visual output

---

## 🧩 Initialization Phase

### 1. `UCPP_SimulationManager::BeginPlay()`

- Instantiates `UDiagnosisRegistery` and calls `Initialize(this)`
- Loads Breathing Dot widget class via soft path and removes any editor-placed instances:
  - Uses `UWidgetBlueprintLibrary::GetAllWidgetsOfClass` and removes widgets not marked `bIsRuntimeSpawned`

### 2. `UDiagnosisRegistery::BuildDiagnosisList()`

For each diagnosis:
- Sets metadata: name, type, description
- Sets default simulation values via `FSimulationSlideBarsParameters`
- Registers Niagara FX: `.AddEffect()`
- Registers indicators: `.AddIndicator()`
- Enables temperature feedback: `.SetTemperatureDisplay()`
- Registers behavior logic: `.SetBehavior(NewObject<UYourShockBehavior>())`

Example:

```cpp
DiagnosisBuilder(this)
  .SetName("Cardiogenic Shock")
  .SetSimulationParameters(Params)
  .AddEffect(NS_Sweat, true, "Forehead")
  .AddIndicator("Wrist", "Pulse", "Weak pulse", Icon)
  .SetBehavior(NewObject<UCardiogenicShockBehavior>(this))
  .SetTemperatureDisplay(true, 36.5f, 30.0f, "spine_03", "Wrist")
  .Build();
```

These instances are stored in a `TArray<TUniquePtr<UCPP_Diagnosis>> DiagnosisList`.

---

## 🔁 Changing Diagnosis

Called by UI or developer logic to switch medical scenarios.

### 1. `UCPP_SimulationManager::ChangeDiagnosis(EDiagnosisType NewDiagnosis)`

Actual order in code:
- Set `SelectedDiagnosisType`
- `ClearActiveEffects()` (deactivate/destroy Niagara)
- `ClearActiveIndicators()` (remove widgets & clear links)
- Retrieve `UCPP_Diagnosis& Diagnosis` from registry
- Broadcast `FChangeDiagnosis` with the selected `Diagnosis`
- If there is a previous behavior: `CurrentShockBehavior->OnExit()` and null it
- Set `CurrentShockBehavior = Diagnosis.RuntimeBehavior`
- If behavior present: `CurrentShockBehavior->OnEnter(&Diagnosis, this)`
- `SpawnIndicators(Diagnosis)`

Notes:
- The broadcast happens after FX/indicators are cleared, but before `OnExit/OnEnter` of behaviors.

---

## 📊 Updating Simulation

Simulation parameters are changed in real time by UI sliders.

### 2. `UCPP_SimulationManager::UpdateSimulation()`

Actual order in code:
- Broadcast `FUpdateSimulation` with `*SimulationParameters`
- If a behavior is active: `CurrentShockBehavior->OnUpdate(SimulationParameters)`

---

## 🛑 Stopping Simulation

Called when user exits, resets, or changes context.

### 3. `UCPP_SimulationManager::StopSimulation()`

Actual order in code:
- Set `bIsSimulationRunning = false`
- Broadcast `FStopSimulation`
- `ClearActiveEffects()` and `ClearActiveIndicators()`
- If a behavior is active: `CurrentShockBehavior->OnExit()` and null it

---

## 🔄 Tick-time Updates

`UCPP_SimulationManager::TickComponent()`
- Early-outs if `TargetSkeletalMesh` or `PlayerController` is missing
- For each active `UShockIndicator`: computes screen position from bone/socket and updates its `UBreathingDotWidget` position
- Rotates `AFloatingTemperatureLabel` actors (core/skin) to face the camera (billboard)

---

## 💬 Delegate contracts (signatures)

- `FStartSimulation` — no params
- `FStopSimulation` — no params
- `FUpdateSimulation` — `(const FSimulationSlideBarsParameters&)`
- `FChangeDiagnosis` — `(UCPP_Diagnosis&)`

Use `AddUObject/RemoveAll` to subscribe/unsubscribe (see Blood systems docs for examples).

---

## 💡 Key Class Responsibilities

| Class                  | Role                                                           |
|------------------------|----------------------------------------------------------------|
| `UCPP_SimulationManager` | Orchestrates full simulation lifecycle                        |
| `UCPP_Diagnosis`       | Stores static diagnosis data                                   |
| `UShockBehavior`       | Executes runtime logic like timers, FX, and state changes      |
| `UDiagnosisRegistery` | Owns and builds all diagnosis entries                          |
| `UShockIndicator`      | Represents each breathing dot data                             |
| `UBreathingDotWidget`  | Visual UI representation of an indicator                       |
| `FloatingTemperatureLabel` | Displays core and skin temp near anatomy                   |

---

## ✅ Tips for Developers

- Ensure subscribers tolerate the actual ordering (broadcast ChangeDiagnosis before behavior OnExit/OnEnter).
- On UpdateSimulation, expect broadcast before behavior `OnUpdate`.
- Don’t leak widgets/FX — rely on `ClearActiveIndicators/Effects` and behavior `OnExit()`.
- Indicators are UI widgets that are repositioned every tick; avoid expensive work inside Tick.
