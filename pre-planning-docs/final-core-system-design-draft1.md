Here is the high-level Game Design Document (GDD) overview for your project, focusing on the main architectural pillars we've designed.

You can save this as `Project_Architecture_Overview.md`.

-----

```markdown
# GDD: Scalable AI Team-Action RPG (Project Overview)

This document outlines the high-level architecture for a team-based action RPG. The entire system is designed to be modular, scalable, and network-ready, with a primary focus on a sophisticated "Smart AI" system.

---

## 1. 🎯 Core Concept: "The Nexus Trial"

* **Genre:** 3/4 Perspective, 3D Action RPG.
* **Pillars:**
    1.  **Team Swap:** Instantly "possess" any AI-controlled teammate.
    2.  **Smart AI:** A deep, "Planner/Reactor" (GOAP) system for all entities.
    3.  **Hero Collection:** Progression is based on unlocking new "Shapers" (heroes).
* **Demo Scope:** A 5-10 minute, single-arena "Trial" that introduces all mechanics and AI systems by having the player unlock their 3-person team, one by one, before a final boss.

---

## 2. 🧠 AI Architecture: The Hybrid "Planner/Reactor"

The core of the "Smart AI" is a two-layer system that is both **reactive** and **strategic**.

1.  **The "Reactor" (Nervous System):**
    * **Job:** Instant, high-priority reflexes.
    * **Logic:** A simple, fast script (or FSM) that runs on events (e.g., `OnTakeDamage`) or in `Update()`.
    * **Purpose:** Handles immediate survival: iFrame dodging unblockable attacks, flinching, or panic-blocking.
    * **Key Feature:** It *instantly interrupts* and *cancels* the Planner's current action.

2.  **The "Planner" (The GOAP Brain):**
    * **Job:** Slow, deliberate, strategic planning (using GOAP).
    * **Logic:** A GOAP planner (e.g., ReGoap) that runs *infrequently* (e.g., when idle, or after being interrupted by the Reactor).
    * **Purpose:** To build a short-term plan (a queue of `GoapAction`s) to achieve a `GoapGoal` (e.g., `[Action: MoveToCover] -> [Action: FireBow]`).

---

## 3. 🎨 AI Design: "Goals" vs. "Personalities"

To create reusable and distinct AI, we separate *what* an AI wants from *how* it tries to get it.

* **Goals (The "What"):** A `GoapGoal` ScriptableObject defining a strategic objective (e.g., `Goal_KillPlayer`, `Goal_ProtectClient`).
* **Personalities (The "How"):** An `AIPersonality` ScriptableObject that defines tactical bias by **modifying Action Costs**.

The GOAP planner always finds the "cheapest" plan. By changing action costs, we change the AI's behavior.

**Example: Personality Cost Modifiers**
| Action | Base Cost | `Aggressive` Mod | `Defensive` Mod |
| :--- | :---: | :---: | :---: |
| `Action_MeleeAttack` | 10 | **-5 (Cheap!)** | +15 (Expensive!) |
| `Action_DodgeRoll_Away` | 15 | +10 (Expensive!)| **-10 (Cheap!)** |
| `Action_Block` | 5 | +10 (Expensive!)| **-3 (Cheap!)** |

This system allows us to create any AI by mixing components (e.g., `Goal_ProtectClient` + `Defensive` Personality = "Tank," `Goal_KillPlayer` + `Aggressive` Personality = "Berserker").

---

## 4. 💾 Data Architecture: The `ScriptableObject` Ecosystem

The entire game's data (Heroes, AI, Actions, Goals) is stored in `ScriptableObject`s for a modular, designer-friendly workflow.

```

[ShaperProfile.asset] (The "Hero" Bundle)
|
|--- references ---\> [GameObject Prefab] (3D Model)
|
|--- references ---\> [AIPersonality.asset] (The "How," e.g., "Aggressive")
|                        |
|                        |--- references ---\> [GoapAction.asset] (e.g., "MeleeAttack")
|
|--- references (List) ---\> [GoapGoal.asset] (The "What," e.g., "Goal\_AssistPlayer")
|
|--- references (List) ---\> [GoapAction.asset] (All actions this Hero knows)

```

* **`ShaperProfile.cs` (ScriptableObject):** The central "Hero" file. It bundles a hero's prefab, stats, personality, known goals, and available actions into one package.
* **`GoapAction.cs` (ScriptableObject):** A single AI "ability" (e.g., `Action_Block`).
* **`AIPersonality.cs` (ScriptableObject):** A set of `ActionCostModifier`s (e.g., `Aggressive.asset`).
* **`GoapGoal.cs` (ScriptableObject):** A single strategic objective (e.g., `Goal_KillPlayer.asset`).

---

## 5. 📈 Progression: The "Hero Collector" Model

Instead of a complex "AI chip" inventory, progression is simple, engaging, and character-focused.

* **Core Loop:** The player's main reward for completing missions is **unlocking new "Shapers"** (i.e., their `ShaperProfile.asset`).
* **Simplicity:** Unlocking "Goliath" *is* unlocking the "Tank" class. His `Defensive` personality and `Goal_ProtectClient` are pre-packaged as part of his character.
* **Metagame:** Long-term strategy comes from **Team Composition**—choosing the right 3 (or more) Shapers for a mission.

---

## 6. 🌐 Scalable Architecture (Co-op & Large Battles)

The system is built from the ground up to support co-op and large-scale encounters.

* **The "Controller" Model:** Every "body" (a Shaper or Enemy prefab) is a "puppet." It can be "possessed" by different "brain" scripts (`BaseController.cs`).
    * `PlayerController.cs`: Reads local input.
    * `AIController.cs`: Reads input from the GOAP AI.
    * `NetworkController.cs`: Reads input from a remote player.
    * **Swapping/Co-op is just enabling/disabling these "brain" scripts.**

* **`FactionManager.cs`:** A single, global manager that replaces `TeamManager` and `EnemyManager`. It tracks all entities in "Factions" (e.g., "Player," "Enemy") and their relationships (e.g., "Hostile"). This scales to any number of teams.

* **`ControlManager.cs`:** The authoritative "server" script that manages the "control state" of all entities in the "Player" Faction. It handles all `RequestSwap` commands from local or network players, assigning control to a Player or `AIController` as needed.

* **`ObjectPooler.cs`:** To support "as many enemies as I want," all enemies are pooled. `Instantiate()` and `Destroy()` are *never* called during gameplay, ensuring high performance.
```