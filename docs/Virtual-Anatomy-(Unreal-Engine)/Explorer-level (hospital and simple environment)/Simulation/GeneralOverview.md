# Simulation of Blood Flow 

This section explains the architecture of the simulation and the rationale behind creating different C++ classes for inheritance in Blueprints.

## Why Simulation 

Nursing students often struggle to visualize how the human body operates. This simulation aims to bring the 3D anatomy model to life, enhancing understanding through interactive learning. By incorporating animations, we will simulate blood behavior and the oxygen transition within the human body, providing a dynamic educational tool. 

## Event Driven Design

Given that many different components need to interact and transform data, the simplest approach might seem to be directly calling functions across various parts of the simulation. However, this is not feasible in Unreal Engine without obtaining the correct actors and subsequently calling their functions, which can be inefficient.

To overcome this issue, we decided to use `delegates`, which are C++ mechanisms for dispatching events. This will enable us to signal different states of the simulation to various actors, allowing them to respond appropriately.

High level overview can be seen in this image:

<figure markdown="span">
  ![Simulation event graph](https://jrcz-data-science-lab.github.io/VirtualAnatomy-Documentation/images/simulation-event-graph.png)
  <figcaption>Event graph of simulation</figcaption>
</figure>

The simulation is being triggered from the Simulation button class which is the green button on the top right corner of application. This action is going to drigger Simulation manager's `FStartSimulation` delegate which will broadcast the event to all of its subscribers. 

In order to listen to the event in any function we are need to access the `SimulationManager` class.


## Simulation Trigger

The simulation is triggered from the Simulation button class, represented by a green button located in the top right corner of the application. The `CPP_StartSimulation` button holds all of the logic for the button the `WB_SimulationButton` is ther to ensure smooth designing experience. When pressed, this button triggers the `FStartSimulation` or `FStopSimulation` delegate in the `SimulationManager` class, which then broadcasts the event to all of its subscribers.

To listen to this event in any function, we need to access the `SimulationManager` class.

## Simulation manager and listening to the events 

To access any class instance in Unreal Engine, it typically needs to be placed in the world, and then we must search the world to retrieve the instance we want. This approach, however, can become inefficient over time. To optimize access, the `SimulationManager` is instantiated directly within the `GameMode` class. 

In the file `ACPP_GameMode.h`, you’ll find a `TObjectPtr` to the `SimulationManager` class. Additionally, a helper function in `CPP_AnatomyUtils.h` was created to retrieve this pointer conveniently.

### Example Usage

```c++
#include "CPP_AnatomyUtils.h"

Class::EventBeginPlay()
{
    Super::EventBeginPlay();
    SimulationManger* simManager = AnatomyUtils::GetSimulationManager(GetWorld());
}
```

> **CAUTION**: this function can **only be called in the actors that have access to the World object** and **ONLY** during the `EventBeginPlay` or `NativeConstruct`; otherwise, the function will not find the `GameMode` object, resulting in a segmentation fault and a crash of the editor.

Once the simulation manager is retrieved, either in the `EventBeginPlay` or `NativeConstruct` event, we can assign functions that will be executed once the simulation manager fires `stop/start` events.

To do this, we have to assign functions to the delegates of the simulation manager class.

```C++
void Class::BeginPlay(){
	SimulationManager->StartSimulationEventDelegate.AddUObject(this, &ACPP_ArteriesBloodFlowSimulation::OnSimulationStart);
	SimulationManager->StopSimulationEventDelegate.AddUObject(this, &ACPP_ArteriesBloodFlowSimulation::OnSimulationEnd);
}
```

> **CAUTION**: this can only be done inside the event begin play or native construct otherwise the `this` is `nullptr`

Once the functions are assigned every time the Simulation manger broadcasts the event the functions assigned to it are exectued. 


## Deletion of added functions

Due to the reseons unrelevant to explain here unreal engine will cause crash since functions added through `AddUObject` methods were not deleted. Therefore we have to handle it. To do this you have to overwrite method that is responsible for deleting the acctor. Examples of such methods are `void NativeDesctructor()` or `void EventDestroy()`. To remove the function from the delegete you can call this method on the Simulation manger delegate.


```c++
void Class::NativeDestructor()
{
  Super::NativeDestructor();
	SimulationManager->StartSimulationEventDelegate.RemoveUObject(this, &ACPP_ArteriesBloodFlowSimulation::OnSimulationStart);
}
```

## Starting the animations on simualtion button 

Since each individual animation blueprint prefixed with `BPA_` is inherited from its own C++ class, and each class is further inherited from `CPP_BaseAnimation`, we can start and stop all animations simultaneously once the simulation manager dispatches the start simulation event. 

If a specific human body animation requires modifications such as a delay or other adjustments these changes can be made in the base class of that animation blueprint or in the animation blueprint itself. 

For more details see [Animation classs specification](/VirtualAnatomy-Documentation/VirtualAnatomy/ExplorerLevel/Animations/BaseAnimationClass/)

## Simulation Manager: Events and Flow (summary)
- Delegates (signatures):
  - `FStartSimulation` (no params)
  - `FStopSimulation` (no params)
  - `FUpdateSimulation(const FSimulationSlideBarsParameters&)`
  - `FChangeDiagnosis(UCPP_Diagnosis&)`
- ChangeDiagnosis order in code:
  1) Clear FX and indicators
  2) Get diagnosis and broadcast `FChangeDiagnosis`
  3) Exit previous behavior (if any); enter new behavior
  4) Spawn indicators for the new diagnosis
- UpdateSimulation order:
  1) Broadcast `FUpdateSimulation(*Params)`
  2) Call `CurrentShockBehavior->OnUpdate(Params)`
- StopSimulation order:
  1) Broadcast `FStopSimulation`
  2) Clear FX and indicators
  3) Exit behavior

See also: `ShockSystem/How-Everything-Connects-At-Runtime.md` for full runtime details.

