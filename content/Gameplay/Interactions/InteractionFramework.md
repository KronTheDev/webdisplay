# ATL Interaction Framework — Concept & Implementation Documentation

## Overview

A close-range selector interaction framework built for Above the Line's first-person character base (`ATP_FirstPersonCharacter`). Shared across both gameplay variants (Horror, Shooter) with variant-specific hooks and gating. Designed to integrate directly with the existing `StatusEffectsComponent` for hazard-driven interaction resistance, and a breaching tier system for doors, containers, and access points.

---

## Architecture

### Files

```
Source/AboveTheLine/Interaction/
├── InteractionTypes.h           # All enums and shared structs
├── BreachRequirement.h/.cpp     # UBreachRequirement — optional subobject on interactables
├── InteractableComponent.h/.cpp # Placed on any actor that can be interacted with
└── InteractionComponent.h/.cpp  # Attached to ATP_FirstPersonCharacter
```

`InteractionComponent` is added to `ATP_FirstPersonCharacter` alongside `StatusEffectsComponent`, making it available to both `AHorrorCharacter` and `AShooterCharacter` without subclassing.

---

## `InteractionTypes.h`

### `EInteractionType`
Describes what kind of interaction the interactable performs.

| Value | Description |
|---|---|
| `Instant` | Resolves immediately on press with no hold required |
| `Hold` | Requires the player to hold the interact key for a set duration |
| `Breach` | Governed by the breach system; hold duration scaled by `EBreachRank` |

### `EInteractionResult`
Returned by `ResolveInteraction()` to communicate outcome.

| Value | Description |
|---|---|
| `Success` | Interaction completed |
| `Blocked_Hazard` | Blocked due to player's active hazard severity |
| `Blocked_Key` | Missing required key item |
| `Blocked_Tool` | Missing required breach tool |
| `Blocked_Narrative` | Impassable — narrative lock, no feedback beyond designer-set message |
| `Interrupted` | Hold was cancelled mid-way (damage, movement, etc.) |

### `EBreachRank`
Tier enum assigned per interactable to define breach difficulty.

| Value | Hold Multiplier | Description |
|---|---|---|
| `None` | 1.0x | No breach required — opens freely |
| `Locked` | 1.0x | Requires a matching key item |
| `Barred` | 1.5x | Requires a pry tool (crowbar, pipe) |
| `Reinforced` | 2.5x | Requires a heavy impact tool (sledgehammer) |
| `Sealed` | 3.0x | Requires both a tool AND a key — order enforced |
| `Impassable` | — | Cannot be breached; returns `Blocked_Narrative` immediately |

Hold duration base is set on the `InteractableComponent`. The `EBreachRank` multiplier is applied on top.

---

## `UBreachRequirement`

Optional `UObject` subobject placed on `UInteractableComponent`. When absent, no breach logic runs.

### Properties

| Property | Type | Description |
|---|---|---|
| `BreachRank` | `EBreachRank` | Tier of difficulty for this interactable |
| `RequiredKeyTag` | `FGameplayTag` | Tag that must exist in the player's item tags (`Locked`, `Sealed`) |
| `RequiredToolTag` | `FGameplayTag` | Tag matching the required tool class (`Barred`, `Reinforced`, `Sealed`) |
| `ConsumeKey` | `bool` | If true, the key is removed from the player's inventory on success |
| `ConsumeTool` | `bool` | If true, the tool takes durability damage or is consumed on success |
| `ToolDurabilityDrain` | `float` | Amount of durability drained from the tool if `ConsumeTool` is true |

### Resolution Logic (`ResolveBreachAttempt`)

Evaluated in order when an interaction attempt begins:

1. **Rank `None`** — skip all checks, proceed to hold timer
2. **Key check** (`Locked`, `Sealed`) — query player inventory for `RequiredKeyTag`
   - Fail → return `Blocked_Key`
3. **Tool check** (`Barred`, `Reinforced`, `Sealed`) — query player inventory for `RequiredToolTag`
   - Fail → return `Blocked_Tool`, surface appropriate feedback string ("Needs something to pry this open" / "This won't budge — need something heavier")
4. **Both pass** → begin hold timer with `BreachRank` hold multiplier applied
5. On hold complete → consume key/tool per flags → return `Success`

For `Sealed`, key and tool are both required before the hold begins. Order of check: key first, then tool.

---

## `UInteractableComponent`

Attached to any `AActor` that the player can interact with. Provides the interaction definition and candidate filtering interface.

### Core Properties

| Property | Type | Description |
|---|---|---|
| `InteractionLabel` | `FText` | Display name shown in the interaction prompt |
| `InteractionType` | `EInteractionType` | Instant, Hold, or Breach |
| `InteractRadius` | `float` | Max distance at which this component registers as a candidate |
| `BaseHoldDuration` | `float` | Seconds required to complete a hold/breach (pre-multiplier) |
| `BreachRequirement` | `UBreachRequirement*` | Optional breach subobject; null = no breach gating |
| `HazardThreshold` | `float` | Combined hazard severity above which interaction is blocked (0 = no threshold) |
| `bVariantHorrorOnly` | `bool` | If true, only `AHorrorCharacter` can interact |
| `bVariantShooterOnly` | `bool` | If true, only `AShooterCharacter` can interact |

### Interface

**`CanInteract(ACharacter* Interactor)`** — Virtual, overridable in Blueprint or C++.  
Default implementation checks:
- Interactor variant vs. `bVariantHorrorOnly` / `bVariantShooterOnly`
- `StatusEffectsComponent` combined hazard severity vs. `HazardThreshold`
- Delegates to `UBreachRequirement::ResolveBreachAttempt()` if present

**`OnInteracted`** — `DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FInteractedDelegate, ACharacter*, Interactor)`  
Broadcast on successful interaction completion. Blueprint-hookable on the owning actor.

**`OnInteractionFailed`** — `DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FInteractionFailedDelegate, ACharacter*, Interactor, EInteractionResult, Result)`  
Broadcast on any failure. Used by UI to surface feedback strings.

---

## `UInteractionComponent`

Attached to `ATP_FirstPersonCharacter`. Manages candidate scanning, selection, hold progress, and delegates to `UInteractableComponent`.

### Candidate Selection

Runs on tick within `ScanRadius` (configurable, typically slightly larger than any single `InteractRadius`):

1. Sphere overlap for actors with `UInteractableComponent`
2. Filter: only components where `actor.InteractRadius >= distance to player`
3. Filter: `CanInteract()` returns true
4. Sort remaining candidates by **dot product of (candidate direction) · (camera forward vector)** — highest dot product wins (closest to crosshair)
5. Best candidate stored as `CurrentCandidate`; delegate broadcast to HUD on change

### Properties

| Property | Type | Description |
|---|---|---|
| `ScanRadius` | `float` | Max search radius for overlap (should be >= largest interactable's `InteractRadius`) |
| `InteractAction` | `UInputAction*` | Enhanced Input action for interact press/hold |
| `HazardHoldPenaltyScale` | `float` | Multiplier added to hold duration based on combined hazard severity (default 1.0, scales up with gas/O2) |

### Hold & Interruption

- On interact press: call `CanInteract()` → begin hold timer if valid
- Hold progress broadcast each tick via `OnHoldProgress(float Percent)` delegate for UI fill bar
- **Interruption triggers:**
  - `StatusEffectsComponent::ApplyDamage()` called with any severity → cancels hold, broadcasts `Interrupted`
  - Player moves beyond `InteractRadius` of candidate → cancels hold
  - `InteractAction` released before hold complete (for `Hold` / `Breach` types)
- On cancel: resets hold state, broadcasts `OnInteractionFailed` with `Interrupted`

### Hazard Hold Penalty

Combined severity = `GasSeverity + OxygenSeverity` (from `StatusEffectsComponent`, clamped 0–1).  
Effective hold duration = `BaseHoldDuration × BreachRank.Multiplier × (1 + CombinedSeverity × HazardHoldPenaltyScale)`

This makes breaching under gas exposure or oxygen deprivation progressively slower and riskier.

### Horror-Specific: Interaction Stamina Cost

When the interactor is `AHorrorCharacter`, a successful interaction drains sprint meter by `InteractionStaminaCost` (float, set on `UInteractableComponent`, default 0).  
Zero-cost on Shooter variant — the field exists but is ignored.

### Delegates (exposed to HUD / Blueprint)

| Delegate | Params | Purpose |
|---|---|---|
| `OnCandidateChanged` | `UInteractableComponent* NewCandidate` (nullable) | HUD shows/hides prompt |
| `OnHoldProgress` | `float Percent` | HUD fill bar for hold/breach |
| `OnInteractionCompleted` | `EInteractionResult Result` | HUD confirmation flash |
| `OnInteractionFailed` | `EInteractionResult Result` | HUD failure feedback (key missing, tool missing, hazard block) |

---

## Hazard Gating Summary

| Hazard Condition | Effect on Interaction |
|---|---|
| `GasSeverity` or `OxygenSeverity` above `HazardThreshold` | `CanInteract()` returns false — hard block |
| Below threshold but non-zero | Increases effective hold duration via `HazardHoldPenaltyScale` |
| `ApplyDamage()` fired during hold | Immediately cancels hold — `Interrupted` |

---

## Breach Feedback Strings

Surfaced via `OnInteractionFailed`. Suggested strings per result (designer-overridable on `UInteractableComponent`):

| Result | Default Feedback |
|---|---|
| `Blocked_Key` | "It's locked." |
| `Blocked_Tool` (Barred) | "Something's jamming it — need a pry tool." |
| `Blocked_Tool` (Reinforced) | "This won't give. Need something heavier." |
| `Blocked_Narrative` | Designer-set string on the component; no default |
| `Blocked_Hazard` | "Can't focus right now." |

---

## Integration Points

| System | Integration |
|---|---|
| `StatusEffectsComponent` | Hazard thresholds, hold penalty, damage interruption |
| `AHorrorCharacter` | Sprint meter drain on interact (`InteractionStaminaCost`) |
| `ATP_FirstPersonCharacter` | `UInteractionComponent` added as a default subobject |
| Enhanced Input | `InteractAction` bound in `SetupPlayerInputComponent` on the base character |
| Player Inventory (future) | `RequiredKeyTag` / `RequiredToolTag` queried via `FGameplayTag` — inventory system must expose a tag presence check |

---

## Out of Scope (Intentional)

- Inventory management — tags are queried, not managed, by this system
- Animation — montages are the caller's responsibility (same pattern as `ShooterWeapon`)
- Network replication — not scaffolded in this version
- World-state persistence (e.g. door stays open between sessions) — handled externally
