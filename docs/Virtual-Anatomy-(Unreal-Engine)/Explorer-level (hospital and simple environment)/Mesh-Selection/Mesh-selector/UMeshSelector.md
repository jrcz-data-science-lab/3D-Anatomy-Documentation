# `UMeshSelector`

## Class Description

`UMeshSelector` highlights meshes (parts of anatomy) the user selects and updates the "Clicked On Info" UI panel with the selected name. It also sends a click event with the owning actor.

- Loads a Mesh Knowledge Base data asset for display names (falls back to raw mesh names if missing).
- Attempts to find a `UCPP_ClickedOnInfo` widget and hide it initially (panel may be absent, in which case only highlighting works).
- Broadcasts `OnMeshClicked` with the owning actor of the selected mesh (if any and if listeners are bound).

---
## Public API

### `void Init(UWorld* world)`
Initializes internal references.
- Stores the world as a weak pointer.
- Searches for `UCPP_ClickedOnInfo` widgets (first one is used if found).
- Loads `/Game/DataAssets/DA_MeshKnowledgeBase.DA_MeshKnowledgeBase`.
- If widget and its sub-widgets exist, clears text and hides the panel.
- If world is null or widget missing, logs an error; class still usable for highlight logic.

Parameter:
- `world` – world context.

### `void HighlightActor(AActor* actor)` (deprecated)
Sets highlight state across all mesh components on the actor and updates the info panel with the actor name.

### `void HighlightComponent(UMeshComponent* mesh)`
Highlights only the provided mesh component and updates the panel with a display name.
Workflow:
1. Null-checks the mesh.
2. Un-highlights previously selected mesh component.
3. Sets custom depth + stencil (`1` on, `-1` off).
4. Leaves visibility of skeletal meshes alone; toggles visibility for other mesh types.
5. Resolves display name from knowledge base; uses raw mesh name if not found or asset missing.
6. Shows info panel (if widget exists).
7. Broadcasts `OnMeshClicked` with owning actor if available; logs warning if no owner.

### `void DeselectAllActors()` (deprecated)
Turns off highlight for current and previous actors (all their mesh components) and hides the info panel if present.

### `void DeselectAllComponents()`
Turns off highlight for current and previous mesh components and hides the info panel.

### `UMeshComponent* GetCurrentlySelectedMesh() const`
Returns pointer to currently highlighted mesh component (may be `nullptr`).

### `void TriggerMeshClicked(AActor* ClickedActor)`
Broadcasts the event if any listener is bound; logs a warning if none are bound.

### Event
`UPROPERTY(BlueprintAssignable) FOnMeshClicked OnMeshClicked;`

---
## Behavior Details

Highlight implementation uses Unreal's Custom Depth pass.
- Visible highlight: `RenderCustomDepth = true`, `CustomDepthStencilValue = 1`.
- Hidden highlight: `RenderCustomDepth = false`, `CustomDepthStencilValue = -1`.

Actor highlight (deprecated path):
- Iterates all `UMeshComponent` instances on the actor.
- Skips visibility toggle for components tagged with `VisibleInSideMenu`.

Component highlight:
- Visibility is unchanged for `USkeletalMeshComponent`.
- Visibility toggled for other mesh component types.

Knowledge Base:
- If loaded, looks up `MeshCatalogByMeshName[MeshName].DisplayName`.
- If entry missing, logs Verbose and uses raw name.
- If asset failed to load, logs a Warning and uses raw name.

UI Panel:
- Requires non-null widget and its `SelectedPartOfAnatomy`, `VerticalFlexBox`, `Background` sub-widgets.
- If any are missing, operations proceed without UI update.

Event Broadcast:
- Only sends owning actor; mesh component pointer itself is not broadcast.
- Logs Warning if mesh has no owner or if `OnMeshClicked` has no listeners.

---
## Members (as implemented)

- `UPROPERTY() UMeshKnowledgeBaseDataAsset* m_knowledgeBaseDataAsset = nullptr;`
- `UPROPERTY() AActor* m_currentlySelectedActor = nullptr;` (deprecated path)
- `UPROPERTY() AActor* m_previouslySelectedActor = nullptr;` (deprecated path)
- `UPROPERTY() UMeshComponent* m_currentlySelectedMesh = nullptr;`
- `UPROPERTY() UMeshComponent* m_previouslySelectedMesh = nullptr;`
- `UPROPERTY() UCPP_ClickedOnInfo* m_displayedClickOnNameWidget = nullptr;`
- `TWeakObjectPtr<UWorld> m_world;`

Private helpers (not exposed):
- `SetActorHighlightVisible(AActor*, bool)` (deprecated)
- `SetComponentHighlightVisible(UMeshComponent*, bool)`

---
## Logging & Error Cases

- World null: error logged, initialization aborted.
- Widget missing: error logged; highlight still works.
- Knowledge Base missing: warning logged; raw names used.
- Mesh null passed to `HighlightComponent`: error logged; no further work.
- Mesh without owner: warning logged; event not broadcast.
- Unbound `OnMeshClicked`: warning logged.

---
## Deprecated APIs

- `HighlightActor(AActor*)`
- `DeselectAllActors()`
- `SetActorHighlightVisible(AActor*, bool)`

These remain for compatibility.

---
