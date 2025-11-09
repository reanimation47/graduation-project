# AI Design Doc: Reusable AI Personalities with GOAP

This document outlines an advanced, reusable AI architecture. The core principle is to **separate an AI's strategic "Goals" from its tactical "Personality."**

This allows us to mix and match components to create dozens of unique AI behaviors (for both enemies and teammates) from a small, reusable set of code.

---

## 1. The Core Concept: Goals vs. Personalities

Every "smart" NPC in our game will have two main components to its brain:

1.  **Goals (The "What"):** This is the AI's current **strategic objective**. It's the "G" in GOAP. A Goal is a state of the world the AI wants to achieve (e.g., `(isPlayerDead)`).
2.  **Personality (The "How"):** This is the AI's **tactical bias**. It's a set of preferences that determines *how* the AI *prefers* to achieve its Goal. This component generally does not change.

**Example:**
* An **Enemy Grunt** and a **Boss** might share the same `Goal_KillPlayer`.
* The **Grunt's Personality** (`Aggressive`) makes it rush in recklessly.
* The **Boss's Personality** (`Defensive`) makes it hang back, block, and wait for an opening.
* Same **Goal**, different **Personality** = completely different fight.

---

## 2. The Mechanism: How "Personality" Works

A "Personality" is just a set of **Action Cost modifiers**.

Here's the logic:
* Your GOAP planner is an algorithm (like A*) that finds a path to the Goal.
* It *always* searches for the "cheapest" plan, based on the `cost` of each `Action`.
* A **Personality**, therefore, is a simple data file that tells the planner, "This AI thinks certain actions are 'cheaper' or 'more expensive' than normal."

By changing the `cost`, we change the AI's "preference" and thus its entire behavior.

---

## 3. Implementation: Creating Personalities

Let's start with a baseline cost for all our AI's "Actions."

### `Base_Action_Costs` (The Default)
* `Action_MeleeAttack`: Cost = 10
* `Action_DodgeRoll_Away`: Cost = 15
* `Action_DodgeRoll_Forward`: Cost = 20 (it's riskier)
* `Action_Block`: Cost = 5
* `Action_FireRangedWeapon`: Cost = 8
* `Action_KeepDistance`: Cost = 10

Now, we create "Personality" files that just *modify* these costs.

### Personality 1: `Aggressive / Risky`
This AI loves to attack and close the gap. It "prefers" offensive actions by making them feel *cheaper* to the planner.

* `Action_MeleeAttack`: Cost = **5** (Very cheap! "I love attacking!")
* `Action_DodgeRoll_Away`: Cost = **25** (Expensive! "I hate retreating!")
* `Action_DodgeRoll_Forward`: Cost = **10** (Cheap! "This moves me closer!")
* `Action_Block`: Cost = **15** (Expensive! "Blocking is boring!")
* `Action_KeepDistance`: Cost = **30** (Very expensive! "Get me in there!")

**Result:** When this AI's GOAP planner runs, it will almost *always* generate plans that involve `DodgeRoll_Forward` and `MeleeAttack` because they are the "cheapest" path to its goal.

### Personality 2: `Defensive / Safe`
This AI hates risk and "prefers" to stay alive. It makes offensive actions feel *expensive*.

* `Action_MeleeAttack`: Cost = **25** (Very expensive! "I only attack if it's the *only* option!")
* `Action_DodgeRoll_Away`: Cost = **5** (Very cheap! "Safety first!")
* `Action_DodgeRoll_Forward`: Cost = **30** (Insanely expensive! "Why would I roll *towards* the danger?!")
* `Action_Block`: Cost = **2** (Very cheap! "This is my favorite thing to do!")
* `Action_KeepDistance`: Cost = **5** (Cheap! "I love staying safe.")

**Result:** This AI's plans will heavily favor blocking and dodging away. It will only attack if the planner can't find any other "safe" (cheap) plan.

---

## 4. Reusability: The Mix-and-Match System

Now you can create any AI in your game by simply giving it a **Goal** and a **Personality**. This is incredibly powerful and reusable.

| NPC Type | Goal (The "What") | Personality (The "How") | Resulting Behavior |
| :--- | :--- | :--- | :--- |
| **Enemy Grunt** | `Goal_KillPlayer` | `Aggressive` | A berserker. Rushes you, rolls forward, attacks constantly. |
| **Teammate Berserker**| `Goal_AssistPlayer` | `Aggressive` | Same as the Grunt, but attacks *your* target. |
| **Enemy Archer** | `Goal_KillPlayer` | `Defensive` | Tries to "kite" you. Constantly rolls away, attacks from a distance. |
| **Teammate Healer** | `Goal_AssistPlayer` | `Defensive` | Stays far back, avoids all conflict, focuses on healing actions. |
| **Boss Bodyguard** | `Goal_ProtectClient` | `Aggressive` | "The best defense is a good offense." Rushes *you* to get you away from the boss. |
| **Teammate Tank** | `Goal_ProtectClient` | `Defensive` | Stays *between* you and the enemy, blocks constantly, and uses "Taunt" actions. |

---

## 5. Special Case: The "Protective" Goal

You were exactly right to identify "Protective" as a **Goal**, not a personality.

When an AI's active goal is `Goal_ProtectClient`, it simply "unlocks" a new set of **Actions** for the GOAP planner to use. The AI's **Personality** (`Aggressive` or `Defensive`) will then determine *how* it uses these new actions.

### New Actions for `Goal_ProtectClient`
* **`Action_Taunt`:**
    * **Preconditions:** `(ClientIsBeingAttacked)`
    * **Effects:** `(EnemyIsAttackingMe)`
* **`Action_BodyBlock`:**
    * **Preconditions:** `(ClientIsBeingAttacked)`
    * **Effects:** `(IAmBetweenClientAndEnemy)`
* **`Action_HealClient`:**
    * **Preconditions:** `(ClientHealthIsLow)`, `(IHavePotion)`
    * **Effects:** `(ClientHealthIsFull)`

**Example:**
A **`Defensive`** Tank with `Goal_ProtectClient` will see `Action_BodyBlock` as *very cheap* and will constantly try to get between you and the enemy.
An **`Aggressive`** Tank with `Goal_ProtectClient` will see `Action_Taunt` as *very cheap* and will try to draw all enemies to itself to "protect" you by being the center of attention.

---

## 6. Suggested Unity Architecture: Scriptable Objects

The best way to manage this in Unity is with **Scriptable Objects**.

1.  **`GoapAction.cs` (ScriptableObject):**
    * This asset holds the *base* data for an action: its name (`Action_MeleeAttack`), its preconditions, its effects, and its `BaseCost` (e.g., 10).

2.  **`AIPersonality.cs` (ScriptableObject):**
    * This asset holds a list/dictionary of "cost modifiers."
    * Example: For the `Aggressive` personality, it would have an entry: `(Action_MeleeAttack, -5)`, `(Action_DodgeRoll_Away, +10)`.

3.  **`AI_Controller.cs` (MonoBehaviour):**
    * This is the main "brain" script you attach to your NPC.
    * It has a field: `public AIPersonality myPersonality;`
    * When the GOAP planner runs, it asks for the cost of `Action_MeleeAttack`. The `AI_Controller` gets the `BaseCost` (10) from the Action, then gets the modifier (-5) from its `myPersonality`, and tells the planner the final cost is **5**.

This makes your AI design **modular** and **designer-friendly**. You can create a new AI personality (`Timid`, `Erratic`, `Suicidal`) just by creating a new `AIPersonality` asset in the Unity project window—**no new code required.**