# 🧬 Diagnosis System

## Overview

> **NOTE**: This system uses C++ and UObject patterns. It assumes you're familiar with:
>
> - UObject ownership/Outers (`NewObject` and lifetime)
> - References vs. pointers (who owns what)
>
>This system powers the disease and shock simulation in the Virtual Anatomy project. It defines diagnosis metadata (symptoms, simulation parameters, FX, UI indicators, logic) that gets used at runtime.

Currently supported diagnoses include:
- Healthy
- Cardiogenic Shock
- Obstructive Shock
- Distributive Shock

It is designed to be **extensible**, so more diagnoses can be added with minimal effort.

---

## 🧠 Core Design Principles

- All diagnoses are **created and owned** by `UDiagnosisRegistery`.
- Diagnoses are stored as raw `UCPP_Diagnosis*` in a `TArray` owned by the registry.
- External systems (e.g. `UCPP_SimulationManager`, blood systems) never take ownership:
  - They receive **references** (`UCPP_Diagnosis&`) or non-owning pointers.
- Construction of each diagnosis is handled via a `DiagnosisBuilder` using fluent method chaining.
- Runtime behavior (shock logic) is represented by `UShockBehavior` subclasses and set per diagnosis.

> ⚠️ Never manually `delete` a `UCPP_Diagnosis` or transfer ownership.
> All diagnoses are UObjects owned by the registry’s outer and cleaned up by Unreal.

---

## 📦 Components

### `UDiagnosisRegistery`
- Owns all `UCPP_Diagnosis` instances.
- Provides `GetDiagnosisByType(EDiagnosisType)` which returns a reference to the matching diagnosis (or a safe fallback).
- Initializes all diagnoses in `BuildDiagnosisList()`.
- Loads and holds all Niagara FX and UI texture assets used by diagnoses.

### `UCPP_Diagnosis`
- Encapsulates all diagnosis-related data:
  - Simulation parameters (`FSimulationSlideBarsParameters`)
  - Blood pressure (`FBloodPressure`)
  - Title + description
  - Enum type (`EDiagnosisType`)
  - FX (`TArray<FDiagnosisEffect>`)
  - UI Indicators (`TArray<UShockIndicator*>`)
  - Temperature label configuration (sockets, colors, values)
  - Optional runtime logic (`UShockBehavior* RuntimeBehavior`)

### `DiagnosisBuilder`
- Helper / fluent builder used only inside `BuildDiagnosisList()` to construct `UCPP_Diagnosis` instances.
- Has privileged access to `UCPP_Diagnosis` internals (via `friend class DiagnosisBuilder`).
- Ensures diagnoses are set up consistently:
  - Name + description
  - Type
  - Simulation parameters
  - Indicators
  - Temperature display settings

---

## 🧾 Diagnosis Type Enum

The simulation system uses the `EDiagnosisType` enum to identify each diagnosis uniquely.

```cpp
UENUM(BlueprintType)
enum class EDiagnosisType : uint8
{
    Healthy = 0,
    HyperVolumetricShock,
    CardiogenicShock,
    ObstructiveShock,
    DistributiveShock,
    Death
};
```

In the current implementation of the registry, diagnoses are defined for:
- `Healthy`
- `CardiogenicShock`
- `ObstructiveShock`
- `DistributiveShock`

Other types (`HyperVolumetricShock`, `Death`) are available in the enum and can be added later by extending `BuildDiagnosisList()`.
