Here is the detailed markdown file for the Data Architecture.

-----

```markdown
# AI GDD: Data Architecture

This document details the complete data architecture for the game's AI and "Shaper" (Hero) systems. The entire architecture is built on **`ScriptableObject`s** in Unity. This approach allows for a modular, reusable, and designer-friendly workflow where new AI, personalities, and heroes can be created without writing new code.

---

## 1. 🧬 Core Philosophy: "Mix-and-Match" Assets

The system is designed to separate *what* an AI does from *how* it does it. This is achieved by linking several small, reusable `ScriptableObject` assets together.

* An **Action** (`GoapAction`) is a single "ability" (e.g., "Melee Attack").
* A **Personality** (`AIPersonality`) is a *modifier* that changes the "cost" of these Actions (e.g., "Aggressive" makes "Melee Attack" cheaper).
* A **Goal** (`GoapGoal`) is the AI's objective (e.g., "Protect the Player").
* A **Hero** (`ShaperProfile`) is a "bundle" that combines a 3D model with a specific Personality, a list of Goals, and a list of Actions.

---

## 2. Diagram: The `ScriptableObject` Ecosystem

This diagram shows how the `ScriptableObject` assets reference each other. The `ShaperProfile` is the central "hub" that assembles all the other pieces.

```

[ShaperProfile.asset] (The "Hero" Bundle)
|
|--- references ---\> [GameObject Prefab] (The 3D Model & Controllers)
|
|--- references ---\> [Sprite] (UI Icon)
|
|--- references ---\> [AIPersonality.asset] (The "How," e.g., "Defensive")
|                        |
|                        |--- references ---\> [GoapAction.asset] (e.g., "Block")
|                        |--- references ---\> [GoapAction.asset] (e.g., "DodgeAway")
|
|--- references (List) ---\> [GoapGoal.asset] (The "What," e.g., "Goal\_ProtectClient")
|
|--- references (List) ---\> [GoapAction.asset] (All actions this Hero knows, e.g., "Block," "MeleeAttack")

```

---

## 3. The 4 Core Data Scripts

These are the C# `ScriptableObject` classes that form the foundation of the system.

### A. `GoapAction.cs` (ScriptableObject)
This is the **"Verb"** or **"Ability."** It's the base class for any action an AI can perform.

* **Purpose:** To define a single, self-contained AI behavior, its requirements, and its outcomes.
* **Examples:** `Action_MeleeAttack.asset`, `Action_Block.asset`, `Action_DodgeRoll_Forward.asset`.
* **Key Fields:**
    * `actionName` (string): The human-readable name (e.g., "Melee Attack").
    * `baseCost` (float): The default "cost" for the GOAP planner. A lower cost is more desirable.
    * `preconditions` (List): The world state required *before* this action can run (e.g., `isWeaponDrawn = true`).
    * `effects` (List): The world state that becomes true *after* this action runs (e.g., `enemyHealthIsLow = true`).
    * *(Abstract Methods)*: `CheckProceduralPrecondition()`, `PerformAction()`, etc. These are implemented by child classes (e.g., `Action_MeleeAttack.cs` would check for range in its `CheckProceduralPrecondition()`).

### B. `AIPersonality.cs` (ScriptableObject)
This is the **"How"** or **"Tactical Bias."** This asset modifies the "cost" of actions to make an AI prefer certain behaviors.

* **Purpose:** To define a reusable AI "personality" (`Aggressive`, `Defensive`) that can be applied to any Hero or enemy.
* **Examples:** `Aggressive.asset`, `Defensive.asset`, `Timid.asset`.
* **Key Fields:**
    * `personalityName` (string): The human-readable name.
    * `costModifiers` (List of `ActionCostModifier`): A list of modifications.
        * `ActionCostModifier` (struct):
            * `action` (GoapAction): A reference to the action to modify (e.g., `Action_MeleeAttack.asset`).
            * `costModifier` (float): The value to **add** to the `baseCost`. Use negative numbers to make an action "cheaper" and thus *more desirable* (e.g., `Aggressive` would give `Action_MeleeAttack` a `-5` modifier).

### C. `GoapGoal.cs` (ScriptableObject)
This is the **"What"** or **"Strategic Objective."** This defines a "success state" that the AI's planner will try to achieve.

* **Purpose:** To define a high-level, strategic goal for the AI.
* **Examples:** `Goal_KillPlayer.asset`, `Goal_ProtectClient.asset`, `Goal_StayAlive.asset`.
* **Key Fields:**
    * `goalName` (string): The human-readable name.
    * `priority` (int): The importance of this goal. An AI will always try to solve its highest-priority valid goal.
    * `goalState` (List of `GoaPState`): The desired world state. The GOAP planner's job is to find a chain of `GoapAction`s to make this state true.

### D. `ShaperProfile.cs` (ScriptableObject)
This is the **"Hero"** or **"The Bundle."** This asset ties all the other pieces together into a single, complete, unlockable character package.

* **Purpose:** To store all data related to a single "Shaper" (Hero) in one designer-friendly asset. This is the "loot" the player unlocks.
* **Examples:** `Goliath_The_Tank.asset`, `Aegis_The_Healer.asset`.
* **Key Fields:**
    * **Identity:** `shaperName` (string), `shaperBio` (string), `shaperIcon` (Sprite).
    * **Prefab:** `shaperPrefab` (GameObject) - A reference to the actual 3D model/controller prefab.
    * **Stats:** `maxHealth` (float), `maxStamina` (float), etc.
    * **Core AI Brain:**
        * `personality` (AIPersonality): A *single* reference to the personality asset this AI uses (e.g., `Defensive.asset`).
        * `knownGoals` (List of `GoapGoal`): A list of *all* goals this AI knows how to pursue (e.g., `Goal_ProtectClient` and `Goal_StayAlive`).
        * `availableActions` (List of `GoapAction`): A list of *all* actions this AI can use to build its plans (e.g., `Action_Block`, `Action_Taunt`, `Action_MeleeAttack`).
```