# AREON — Unity Implementation Blueprint

**Target Unity version:** `6000.3.15f1`  
**Prototype type:** 3D, single-player, educational Mars outpost simulation  
**Working title:** AREON — Mars STEM v1.0  
**Primary goal:** Build a playable mechanics prototype with primitive objects only. Final art and models will be supplied by the team later.

---

## 0. How to Use This Blueprint

This document is the source of truth for the implementation. Complete the work in **10 stages containing 5 tasks each (50 tasks total)**. Treat each stage as one manageable Arena AI conversation. Do not ask an AI to implement all 50 tasks in one conversation.

### Tool workflow

- **Arena AI:** Use this blueprint to produce the C# code, implementation plan, exact Unity setup instructions, Inspector values, and validation steps for the current stage.
- **Antigravity + Unity MCP:** Carry out the Unity Editor work: create folders and scripts, configure objects and components, create assets and scenes, enter Inspector values, run the project, inspect Console errors, and verify the stage.
- **Developer:** Review the result and approve the stage before moving to the next one.

### Starting a new Arena AI conversation

Paste this instruction, then provide the relevant stage from this blueprint:

> You are implementing AREON in Unity `6000.3.15f1`. Treat `blueprint.md` as the source of truth. Work only on the stage I provide; do not implement later stages. First inspect the current Unity project and existing files. Preserve all existing work. Do not create duplicate scripts, scenes, folders, or systems. Produce production-quality, beginner-readable C# code and exact Unity Editor instructions for Antigravity to execute through Unity MCP. Use only Unity primitive objects (cubes, planes, spheres and capsules) as temporary prototypes. Do not create or request finished models, textures, animations, environment art, or visual designs. Do not add packages or change Project Settings unless required; explain and justify any required change first. Keep all gameplay code modular and configurable in the Inspector or ScriptableObjects. After implementation, give a test procedure and explicit acceptance criteria. Report every created or modified file and every scene/object/component change.

### Moving to a new conversation

At the end of each stage, ask Arena AI to produce a **Stage Handoff** containing:

1. Stage completed and tasks completed (for example, `Stage 3: Tasks 11–15`).
2. Files created or modified, with their paths.
3. Scene objects created or modified and their hierarchy paths.
4. Important public APIs, serialized fields, events and dependencies introduced.
5. Inspector values and Project Settings changes.
6. Tests performed and their results.
7. Known issues or unfinished work.
8. Exact next stage to implement.

Paste that handoff into the next Arena AI conversation along with the next stage. The new conversation must inspect the actual Unity project before assuming the handoff is still accurate.

---

# 1. Game Definition and Scope

## 1.1 High concept

AREON is a 3D Mars outpost resource-management simulation for players aged approximately 7–12. The player is responsible for keeping **Outpost Ares** operational while collecting science telemetry and responding to environmental hazards.

The defining decision is:

**Science activities and survival systems compete for the same limited power reserve.**

The player moves between three facilities, monitors oxygen, power and the protective shield, and chooses which actions to perform before the environment becomes more dangerous.

## 1.2 Main locations

The playable area represents a 1 km Mars outpost grid. For the mechanics prototype, use a compact, clearly labelled test map; do not attempt to represent a literal 1 km walking distance.

| Facility | Purpose | Required interaction |
|---|---|---|
| Base Dome | Habitat and life support | Inspect status and use the shield control to reinforce the shield |
| Research Center | Science telemetry | Spend 15 Power to download a science data packet |
| Power Generator | Solar collection and battery storage | Recharge Power while daylight and scenario conditions allow it |

The player begins at or near the Base Dome. The player travels on foot in the first prototype. Rover gameplay is not specified well enough to implement and is out of scope for the initial version.

## 1.3 Main resources

| Resource | Meaning | Main behaviour |
|---|---|---|
| Oxygen | Habitat life-support reserve | Depletes over time; Sol 2085 drains at 1.5× baseline |
| Power | Shared electrical reserve | Spent by configured systems and player actions; rechargeable by solar power when allowed |
| Defense Shield | Outpost radiation protection | Depletes over time; C- and X-class flare scenarios drain at 2× baseline |

The website shows `88% Oxygen`, `64% Power` and `72% Shield` as a HUD example. These are **prototype starting defaults only**, not confirmed mission-specific values. Keep all starting values configurable per mission.

## 1.4 Explicit action costs

- Download Science Telemetry: **15 Power**.
- Reinforce Protective Shield: **10 Power**.

Actions must be rejected cleanly when the player does not have enough Power. A rejected action must not partially apply its effect or spend any Power.

## 1.5 Mission scenarios

These are curated historical training scenarios, not three consecutive Martian days and not live telemetry.

| Scenario | Pressure | Solar event | Storm | Oxygen drain multiplier | Shield drain multiplier | Solar charging |
|---|---:|---|---|---:|---:|---|
| Sol 800 — Orientation | 887 Pa | None | Clear | 1.0× | 1.0× | Allowed during daylight |
| Sol 852 — Onset | 924 Pa | C-class | Regional storm | 1.0× | 2.0× | Allowed during daylight unless later specified otherwise |
| Sol 2085 — Crisis | 772 Pa | X-class | Severe global storm / black skies | 1.5× | 2.0× | Disabled |

**Important data rule:** The website states that pressure below 800 Pa can trigger high-velocity dust fronts, but Sol 852 is explicitly a storm scenario at 924 Pa. Therefore, do not derive all storm states from pressure alone. The selected scenario's explicit storm status is authoritative. Pressure may be shown as telemetry and may trigger a secondary rule only when that does not contradict the scenario configuration.

## 1.6 Daylight

- The website specifies a restricted **five-minute daylight window**.
- Implement this as a configurable gameplay timer, defaulting to 300 seconds.
- Display the scenario's daylight start label as `05:00` if a clock label is needed.
- Solar charging is possible only while the daylight timer is active and the mission allows solar charging.
- Sol 2085 disables solar charging because the global storm blocks sunlight.
- Do not assume that five minutes represents a complete real Martian day/night cycle; it is a compressed gameplay window.

## 1.7 Data provenance and honesty

Use the following curated records locally:

```csv
sol,pressure_pa,solar_flare,storm_status
800,887,none,SAFE
852,924,C,WARNING
2085,772,X,WARNING
```

The website identifies these sources:

- Curiosity REMS / NASA Planetary Data System: historical Mars atmospheric pressure.
- NASA DONKI: solar-event context.
- NASA OSDR: astronaut health context mentioned for an AstroGuru feature.

For the initial prototype, use only the supplied curated scenario data. **Do not call NASA APIs, claim live data, or invent missing observations.** Present the values as curated historical training scenarios. AstroGuru AI is not specified sufficiently and is out of scope.

## 1.8 Out of scope for this prototype

- Final character, building, rover or environment models.
- Custom art, textures, animations, cinematic sequences or designed visual effects.
- Combat, weapons or enemies.
- Multiplayer, networking, accounts or cloud saves.
- Live NASA API integration.
- AstroGuru AI implementation.
- Rover-driving mechanics.
- Mobile controls unless separately requested.
- Audio and voice narration beyond simple temporary Editor/UI feedback.

Use primitives and straightforward UI only. Prioritize functioning systems, clear code and reliable integration.

---

# 2. Technical Architecture

## 2.1 Architecture rules

1. Keep gameplay logic separate from visual placeholder objects.
2. Use one authoritative owner for each resource value. No station or HUD script may maintain its own duplicate Oxygen, Power or Shield value.
3. Use ScriptableObjects for mission/scenario configuration, not for mutable runtime resource values shared between play sessions.
4. Use serialized references and Inspector wiring where practical. Avoid fragile scene-wide `FindObjectOfType`/name lookups as the normal dependency system.
5. Avoid hardcoded object names, tags and scene paths where a serialized reference or component reference is appropriate.
6. Use events or explicit interfaces for communication between systems. Unsubscribe from events when objects are disabled or destroyed.
7. Make rates, costs, durations, thresholds and starting values configurable.
8. Validate null references and invalid Inspector values. Fail with useful error messages rather than silent failures.
9. Keep scripts beginner-readable, with focused classes and clear names.
10. Do not introduce third-party packages unless a task cannot reasonably be completed with the installed Unity features.

## 2.2 Proposed folder structure

Create this structure inside `Assets/AREON/`. Do not reorganize unrelated team or project folders.

```text
Assets/AREON/
├── Scenes/
│   └── AREON_Prototype.unity
├── Scripts/
│   ├── Core/
│   ├── Player/
│   ├── Interaction/
│   ├── Resources/
│   ├── Missions/
│   ├── Environment/
│   ├── Stations/
│   ├── Telemetry/
│   └── UI/
├── Data/
│   ├── Missions/
│   └── Telemetry/
├── Prefabs/
│   ├── Player/
│   ├── Stations/
│   └── Environment/
└── Materials/
```

Only create folders when the corresponding stage needs them. Keep all project-specific scripts and assets under `Assets/AREON/` so the mechanics prototype can later be exported or migrated without bringing unrelated project content.

## 2.3 Core components and responsibilities

Names below are architectural guidance. Arena AI may use a different name only when it records the change and preserves the responsibilities.

| Component / data type | Single responsibility |
|---|---|
| `GameBootstrap` | Validates scene references and initializes the selected mission in a defined order |
| `MissionScenarioData` (ScriptableObject) | Stores one scenario's static configuration |
| `MissionManager` | Starts, tracks and ends the active mission |
| `ResourceManager` | Owns current Oxygen, Power and Shield values; validates and applies resource changes |
| `PlayerController` | Handles player locomotion only |
| `PlayerInteractor` | Detects the nearest usable interaction target and sends the interact request |
| `IInteractable` | Common contract for objects the player can use |
| `WeatherManager` | Applies scenario weather state and exposes its effects/multipliers |
| `DaylightManager` | Tracks the five-minute daylight window and reports whether it is active |
| `SolarGenerator` | Requests Power recharge when daylight and scenario rules allow it |
| `ScienceTerminal` | Requests a 15-Power science download and reports success/failure |
| `ShieldConsole` | Requests a 10-Power shield reinforcement and applies the configured shield amount |
| `ObjectiveManager` | Tracks mission objectives and raises objective progress events |
| `TelemetryManager` | Provides the selected scenario's curated data to the UI/science interaction |
| `HUDController` | Displays resource, weather, timer and interaction information; does not calculate gameplay values |
| `MissionResultsController` | Displays success/failure results and summary information |

### Resource ownership rule

`ResourceManager` is the only runtime authority for Oxygen, Power and Shield. Stations call methods such as `TrySpendPower`, `TryRestoreShield` or `TryRechargePower`; they do not directly edit UI values or keep separate resource copies.

### Configuration and runtime-state rule

`MissionScenarioData` contains starting values and drain/recharge settings. At mission start, `ResourceManager` copies those settings into runtime state. Do not modify the ScriptableObject asset while a mission is running.

## 2.4 Interaction contract

Use a small reusable interaction interface. Each interactable should provide:

- A prompt or display name.
- Whether it can currently be used.
- An interaction method receiving the player or interaction context.
- A reason/message when interaction is unavailable, where relevant.

The player should see one clear prompt for the best/nearest valid target. Do not let every nearby station independently display overlapping prompts.

## 2.5 Resource simulation rules

- Use `Time.deltaTime` or a consistent simulation tick for continuous drain.
- Expose baseline drain rates in units per second or percentage points per second; use one convention consistently.
- Apply scenario multipliers in one place only.
- Clamp resources to their valid ranges (normally 0–100 for percentage resources).
- Prevent negative values and accidental double charging/spending.
- Do not invent final drain/recharge rates. Use clearly labelled, Inspector-configurable **prototype tuning values** and record that they require playtesting.
- Power reaching zero should disable any action that costs Power. Whether zero Power itself causes failure must be a configurable mission rule, not an assumed universal rule.

## 2.6 Mission completion and failure rules

The source description does not provide exact win/loss thresholds or exact mission durations. Make them configurable in scenario data.

Recommended initial prototype policy:

- A mission succeeds when all required objectives are completed and the configured survival/time condition is satisfied.
- Oxygen reaching zero causes failure.
- Shield reaching zero during a hazardous solar-event scenario may cause failure; expose this as a configurable rule.
- Power reaching zero alone does not automatically cause failure; it prevents power-cost actions and may lead to a later failure.
- Optional objectives must not block mission completion.

Clearly identify these as prototype gameplay rules to be tuned and approved, not as facts supplied by the website.

---

# 3. The 50-Task Implementation Plan

Complete the stages in order. Each stage has five tasks, a result and a test gate. Do not begin a later stage until the current stage passes its gate.

## Stage 1 — Project Inspection and Safe Foundation

**Goal:** Prepare the mechanics prototype without damaging unrelated project content.

1. Inspect the open Unity project and confirm the Editor version is `6000.3.15f1`.
2. Inspect existing scenes, packages, input settings and Console errors before changing anything.
3. Create the `Assets/AREON/` folder structure described above, without moving or deleting unrelated files.
4. Create and save `AREON_Prototype.unity`; keep the existing project startup scene unchanged unless explicitly approved.
5. Add a simple test floor, a directional light and clearly labelled primitive placeholders. Use only cubes, planes, spheres and capsules.

**Stage gate:** The project opens, the new scene loads, the placeholder floor is visible, and there are no new Console errors. Record the initial project state and any pre-existing errors.

## Stage 2 — Player and Basic Navigation

**Goal:** Make the player move reliably around the prototype map.

6. Create a capsule player with a `CharacterController` or another simple built-in controller; avoid Rigidbody/CharacterController conflicts.
7. Implement walking movement with configurable speed and gravity.
8. Implement a simple third-person follow camera using built-in camera logic; do not add Cinemachine unless it is already installed and clearly beneficial.
9. Read the project's current input configuration and use the already-supported input path. Do not silently switch the project's Active Input Handling setting.
10. Add basic movement boundaries and test walking, stopping, turning, camera following and collision with the floor.

**Stage gate:** Player movement and camera work in Play Mode; the player cannot fall through the floor or pass through basic blocking geometry. Document the controls.

## Stage 3 — Reusable Interaction System

**Goal:** Allow one player to interact with multiple facilities through a shared system.

11. Define the `IInteractable` contract or an equivalent reusable base abstraction.
12. Implement `PlayerInteractor` using a forward raycast or trigger-based detection, with one consistent method for choosing the active target.
13. Add an on-screen interaction prompt that appears only when a valid target is available.
14. Implement the interact input using the project's existing input setup; make the key configurable.
15. Create a temporary test interactable cube and verify enter-range, leave-range, facing, prompt and interaction behaviour.

**Stage gate:** The player can approach, target and interact with one test object. Prompts do not duplicate or remain visible after leaving range.

## Stage 4 — Resource Management Foundation

**Goal:** Implement authoritative Oxygen, Power and Shield values.

16. Create `ResourceManager` as the single owner of runtime resource values.
17. Add configurable initial Oxygen, Power and Shield values, clamped to 0–100.
18. Implement safe Power spending and recharging methods that return success/failure and never allow negative Power or values above capacity.
19. Implement configurable continuous Oxygen and Shield depletion using delta time; keep baseline rates configurable.
20. Publish resource-change events or provide a clean observation API for UI and other systems; ensure subscribers are safely registered/unregistered.

**Stage gate:** A temporary test component can drain and restore resources. Values remain in range, update consistently over time and cannot be modified through duplicate station-owned values.

## Stage 5 — HUD and Resource Feedback

**Goal:** Make resource state understandable while testing mechanics.

21. Create a simple Canvas HUD showing Oxygen, Power and Defense Shield as labels and sliders/bars.
22. Connect the HUD to `ResourceManager`; the HUD must display values and never own or calculate them.
23. Add configurable warning thresholds and clear low-resource feedback.
24. Add a weather/status text area and a daylight countdown area, initially showing placeholder state until later stages connect them.
25. Test HUD updates when resources drain, recharge, reach warning thresholds and are clamped at zero/maximum.

**Stage gate:** All three resource bars accurately reflect runtime values and never drift from `ResourceManager`.

## Stage 6 — Facilities and Their Interactions

**Goal:** Build the three functional outpost stations with primitive geometry.

26. Create a labelled Base Dome placeholder and a shield-control interactable.
27. Create a labelled Research Center placeholder and a science-terminal interactable.
28. Create a labelled Power Generator placeholder and a solar-generator interactable.
29. Add simple UI panels or interaction feedback for each facility; display action name and Power cost before confirming an action where appropriate.
30. Test travel between all three stations and verify each station calls the shared interaction/resource systems rather than duplicating logic.

**Stage gate:** The player can visit and interact with all three stations. Each station is clearly identified using temporary text labels and primitive objects.

## Stage 7 — Power Actions, Solar Charging and Daylight

**Goal:** Implement the core power economy and the daylight restriction.

31. Implement science download: cost exactly 15 Power, only once per configured packet unless the scenario explicitly allows repeat downloads; show success/failure feedback.
32. Implement shield reinforcement: cost exactly 10 Power and restore a configurable amount of Shield; reject the action if Power is insufficient or Shield is already full.
33. Implement `DaylightManager` with a configurable 300-second daylight window and a visible countdown; expose `IsDaylightActive`.
34. Implement `SolarGenerator` with configurable recharge rate and capacity checks; it must recharge only during daylight and only when the active scenario permits solar charging.
35. Test the action economy: exact costs, insufficient Power, full Shield, daylight expiry, and solar charging disabled.

**Stage gate:** The 15-Power and 10-Power actions behave correctly. Solar charging starts/stops at the correct time and cannot run in a scenario that disables it.

## Stage 8 — Mission Data and Environmental Conditions

**Goal:** Configure the three curated Sol scenarios and apply their effects.

36. Create `MissionScenarioData` ScriptableObject structure with Sol number, pressure, solar flare, storm state, starting resources, drain multipliers, daylight settings, charging permission and objective configuration.
37. Create the three data assets: Sol 800, Sol 852 and Sol 2085 using the values in Section 1.5. Keep unspecified rates as editable prototype tuning fields.
38. Implement `WeatherManager` to load and expose the selected scenario's explicit storm/flare state and resource multipliers.
39. Apply Oxygen and Shield multipliers through one central resource simulation path: Sol 2085 Oxygen 1.5×; Sol 852 and Sol 2085 Shield 2×; other listed rates 1×.
40. Add clear status messages for clear weather, regional dust storm, severe global storm, C-class flare and X-class flare. Pressure below 800 Pa may trigger a dust-front rule, but the explicit scenario storm state must not be contradicted.

**Stage gate:** Switching scenarios produces the correct pressure display, flare/storm status, drain multipliers and solar-charging permission. Verify specifically that Sol 852 remains a regional storm even though its supplied pressure is 924 Pa.

## Stage 9 — Mission Flow, Objectives and Telemetry

**Goal:** Make each scenario playable from start to completion using local curated data.

41. Implement `MissionManager` to select, initialize, start, complete and fail a mission in a predictable order.
42. Implement `ObjectiveManager` with configurable required and optional objectives; display progress in a simple objective list.
43. Configure Sol 800 tutorial objectives: visit facilities, learn resource monitoring, recharge during daylight and perform the first science upload. Keep the exact ordering configurable.
44. Configure Sol 852 and Sol 2085 objectives to test resource triage and survival; do not invent extra locations or systems. Make mission duration and completion/failure policy configurable.
45. Implement `TelemetryManager` using the three supplied local scenario records; show Sol, pressure, solar flare and storm status plus a disclosure that this is curated historical training data, not live telemetry.

**Stage gate:** Each of the three scenarios can be started independently, displays the right objectives and data, and reaches a clear success/failure state according to its configured rules. No live API call is made.

## Stage 10 — Results, Integration Safety and Final Validation

**Goal:** Stabilize the prototype and prepare a clean handoff to the team.

46. Create a simple results panel showing mission outcome, completed objectives and final Oxygen, Power and Shield values.
47. Add a safe restart and return-to-mission-selection flow; ensure starting a new mission resets runtime state and timers correctly.
48. Run a full regression test for all three Sol scenarios, all interactions, resource costs, multipliers, daylight expiry and failure conditions; fix all new Console errors and warnings caused by the prototype.
49. Verify that all project-specific scripts, prefabs, data assets and the prototype scene are contained under `Assets/AREON/`; remove only unused AREON test objects/assets after confirming they are not referenced.
50. Prepare an integration handoff: list dependencies, scene setup, required components, input assumptions, known limitations, and the exact folders/assets to export as a Unity package. Do not export unrelated project content.

**Stage gate:** All three missions are playable from start to results, restart works, the Console has no new prototype errors, and the AREON folder is self-contained enough for team review and package export.

---

# 4. Required Mission Configuration Fields

Arena AI should create a ScriptableObject (or equivalent data asset) that exposes at least the following fields. Use enums for flare and storm classifications rather than relying on arbitrary strings internally.

| Field | Type / example | Notes |
|---|---|---|
| `solNumber` | Integer: 800 | Scenario identifier |
| `displayName` | String | Orientation Sol, Onset Sol, Crisis Sol |
| `pressurePa` | Float | Historical scenario value |
| `solarFlareClass` | Enum: None / C / X | Source scenario classification |
| `stormStatus` | Enum: Clear / Regional / SevereGlobal | Explicit scenario status |
| `initialOxygen` | Float 0–100 | Prototype default can begin at 88 |
| `initialPower` | Float 0–100 | Prototype default can begin at 64 |
| `initialShield` | Float 0–100 | Prototype default can begin at 72 |
| `oxygenDrainPerSecond` | Float | Prototype tuning value; not specified by website |
| `oxygenDrainMultiplier` | Float | 1.0, 1.0, 1.5 for listed scenarios |
| `shieldDrainPerSecond` | Float | Prototype tuning value; not specified by website |
| `shieldDrainMultiplier` | Float | 1.0, 2.0, 2.0 for listed scenarios |
| `powerDrainPerSecond` | Float | Optional prototype tuning; define only if passive power drain is used |
| `scienceDownloadPowerCost` | Float | 15 |
| `shieldReinforcePowerCost` | Float | 10 |
| `shieldRestoreAmount` | Float | Configurable; website does not specify amount |
| `daylightDurationSeconds` | Float | 300 by default |
| `solarRechargePerSecond` | Float | Configurable prototype tuning; website does not specify rate |
| `solarChargingAllowed` | Boolean | False for Sol 2085; true for Sol 800 and Sol 852 |
| `missionDurationSeconds` | Float | Configurable; exact duration is not specified |
| `failAtZeroOxygen` | Boolean | Recommended true |
| `failAtZeroShield` | Boolean | Configurable; recommended for active radiation hazard |
| `requiredObjectives` | List | Scenario-specific |
| `optionalObjectives` | List | Scenario-specific |

Do not add a field and then leave it unused. Each field must have a clear runtime consumer or be omitted until needed.

---

# 5. Acceptance Checklist for the Complete Prototype

The prototype is considered ready for team review when all of the following are true:

- [ ] Opens in Unity `6000.3.15f1`.
- [ ] Uses only primitive placeholders for player, stations and environment.
- [ ] Player can navigate the outpost and interact with all three facilities.
- [ ] Oxygen, Power and Shield have one authoritative runtime owner.
- [ ] Oxygen and Shield deplete at configurable rates and scenario multipliers.
- [ ] Science download costs exactly 15 Power.
- [ ] Shield reinforcement costs exactly 10 Power.
- [ ] Insufficient Power prevents the action without partial effects.
- [ ] Solar recharge is limited to the five-minute daylight window.
- [ ] Sol 2085 does not permit solar charging.
- [ ] Sol 800 uses baseline drain multipliers.
- [ ] Sol 852 uses a 2× Shield drain multiplier and displays its regional storm despite 924 Pa pressure.
- [ ] Sol 2085 uses 1.5× Oxygen and 2× Shield drain multipliers and severe global storm status.
- [ ] The UI displays resource values, mission conditions, objectives and daylight state.
- [ ] Telemetry uses only the supplied curated records and is labelled as historical training data, not live telemetry.
- [ ] Mission success, failure, restart and results work.
- [ ] All AREON-specific content is isolated under `Assets/AREON/`.
- [ ] The integration handoff identifies any remaining dependencies or assumptions.

---

# 6. Rules for Every Arena AI Stage

For each stage, Arena AI must deliver these items in order:

1. **Scope confirmation:** Restate the current stage and the five task numbers it covers.
2. **Project inspection:** Identify the existing project state and relevant dependencies before editing.
3. **Implementation plan:** List exact files, GameObjects, components, ScriptableObjects and Inspector changes.
4. **Code:** Provide complete files, not partial snippets, unless a small targeted edit is safer. Include namespaces and required `using` statements.
5. **Unity MCP execution guide:** Give Antigravity explicit Editor actions in the correct order, including hierarchy paths, component attachment and serialized reference assignments.
6. **Validation:** Give reproducible Play Mode tests and expected results.
7. **Change report:** List every created/modified file and scene object; state whether Project Settings or packages changed.
8. **Stage Handoff:** Provide the handoff format defined in Section 0.

### Do not do these things

- Do not implement multiple future stages just because they are related.
- Do not replace or regenerate the whole scene when a targeted edit will work.
- Do not delete, rename or modify non-AREON project files without explicit approval.
- Do not overwrite existing scripts without first inspecting them and explaining why the change is required.
- Do not silently install packages, change input settings, change render pipeline settings or alter build settings.
- Do not fabricate NASA observations, missing mechanics, final resource rates or mission durations.
- Do not make decorative art or final models. Primitive placeholders are intentional.
- Do not claim a task is complete until Antigravity has executed it and the Unity Editor result has been checked.

---

# 7. First Prompt to Use

Start the first Arena AI conversation with:

> Read `blueprint.md`. We are beginning **Stage 1 — Project Inspection and Safe Foundation (Tasks 1–5)** only. Follow Section 6 exactly. First inspect the current Unity project and confirm its version, scene state, input configuration, package state and Console status. Then prepare only the AREON folder structure and the `AREON_Prototype` scene with a primitive test floor, light and labelled placeholder objects. Do not change the existing startup scene, install packages, change Project Settings or touch unrelated files without explaining why and asking for approval. Produce complete scripts only if Stage 1 genuinely requires them. Give Antigravity exact Unity MCP execution instructions and verify the result before providing the Stage 1 Handoff.

**End of blueprint.**
