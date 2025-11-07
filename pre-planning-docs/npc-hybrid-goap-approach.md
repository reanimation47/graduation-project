# AI Design: The Hybrid (Planner/Reactor) Model

## 1. The Big Idea: Brain vs. Nervous System

Your concern is 100% correct: a "planner" (GOAP) is slow. If you ask it to make a plan every time the player swings a sword, the AI will be dead before it "thinks."

The solution is a **Hybrid AI**, also known as a "Deliberative/Reactive" system. We give the AI two components that work together:

* **The "Planner" (The GOAP Brain):** This is the slow, smart part. It's the "strategist." It answers big questions like, "What's my overall goal? How do I win this fight?"
* **The "Reactor" (The Nervous System):** This is the fast, "dumb" (instinctive) part. It's the "reflex." It doesn't *think* at all. It just *reacts* to immediate danger.

The "Reactor" **always** has priority and can *instantly* interrupt the "Planner."

---

## 2. The Two-Layer System Explained

### Layer 1: The "Reactor" (The Nervous System)

This is a simple script that runs on high-priority events (like `OnTakeDamage`) or does very fast checks in `Update()`. It doesn't know *why* it's doing something; it just follows simple, non-negotiable rules.

* **Job:** Handle immediate survival and interrupts.
* **When it runs:** On events, or every frame (for sensor checks).
* **What it does:** *Instantly* takes control of the character to perform a single, hard-coded action (like `Dodge` or `Block`).
* **Key Feature:** It **cancels** any plan the "Planner" (GOAP) was executing.

**Reactor Examples for Your Game:**
* `if (PlayerAttack_IsUnblockable)` -> **EXECUTE** `DodgeRoll()`.
* `if (TookDamage)` -> **EXECUTE** `PlayFlinchAnimation()`.
* `if (PlayerIsDirectlyInFront_And_PlayerAttack_IsStandard)` -> **EXECUTE** `RaiseShield()`. (This is a basic "panic block").
* `if (Health < 20%)` -> **SET_STATE** `(IsInDesperationMode)`.

### Layer 2: The "Planner" (The GOAP Brain)

This is your full **GOAP** system. It is **not** allowed to run every frame. It only runs when it's "given permission" to think.

* **Job:** Create a short, strategic plan (a queue of 1-3 actions).
* **When it runs (Infrequently):**
    1.  When its current plan is finished.
    2.  When the "Reactor" interrupts it (e.g., after a `DodgeRoll` is complete).
    3.  When the AI is idle and the world state changes significantly.
* **What it does:** Looks at the **World State** (e.g., `(MyHealth_IsLow)`, `(PlayerIsFarAway)`, `(PlayerIsCastingSpell)`, `(IHaveStamina)`) and a **Goal** (e.g., `(KillPlayer)` or `(StayAlive)`) to generate a new plan.

**Planner (GOAP) Examples for Your Game:**
* **Goal:** `(KillPlayer)`
* **World State:** `(PlayerIsFarAway)`, `(IHaveArrows)`
* **New Plan:** `[Action: FireBow] -> [Action: FireBow]`

* **Goal:** `(KillPlayer)`
* **World State:** `(PlayerIsClose)`, `(IHaveSword)`
* **New Plan:** `[Action: MeleeAttack] -> [Action: MeleeAttack] -> [Action: RaiseShield]`

---

## 3. How They Work Together: A Combat Scenario

This is the magic. Let's see how the layers interact.

1.  **PLAN (GOAP):** The AI's **Planner** runs. It sees the Player is mid-range and its goal is `(KillPlayer)`. It creates a plan: `[Action: CastFireball]`.
2.  **EXECUTE:** The AI starts executing `CastFireball`. The animation begins.
3.  **EVENT:** You, the player, are faster. You lunge forward with an unblockable *Stab* attack.
4.  **INTERRUPT (Reactor):** The AI's "Reactor" (a sensor) detects `(IncomingUnblockableAttack)`.
5.  **REFLEX (Reactor):** The Reactor *instantly* cancels the `CastFireball` plan. It takes control and executes `DodgeRoll()`. The AI's iFrames pass right through your stab.
6.  **RE-PLAN (GOAP):** The `DodgeRoll` animation finishes. The AI is now idle. This signals the **Planner** to run again.
7.  **NEW PLAN (GOAP):** The Planner looks at the *new* world state: `(PlayerIsVeryClose)`, `(MyMagicIsOnCooldown)`. The old `CastFireball` plan is invalid. It creates a new, better plan: `[Action: EquipMeleeWeapon] -> [Action: MeleeAttack]`.

**The result:** The AI looks like a genius. It tried to cast a spell, saw you coming, "reflexively" dodged your "unblockable" attack, and then pulled out a sword to fight you up close. This is the dynamic, smart-looking combat you want.

---

## 4. Integrating Your *Souls*-like Mechanics

Your new ideas are *perfect* for this model because they can be used by **both** layers.

### 1. Dodge Rolls (with iFrames)

* **As a "Reactor" Action:** This is the AI's "panic button." When the Reactor's sensors detect an immediate, unblockable threat, it fires the `DodgeRoll()` action purely for the **iFrames** to survive.
* **As a "Planner" (GOAP) Action:** The GOAP system gets a new action: `Action_RepositionDodge`.
    * **Preconditions:** `(HasStamina)`, `(PlayerIsTooClose)`.
    * **Effects:** `(PlayerIsMidRange)`, `(StaminaReduced)`.
    * A ranged/magic-based AI will *strategically plan* to use this action to create space so it can use its bow or spells. It's using the **movement** of the dodge, not just the iFrames.

### 2. Directional Shield Blocking

This is a fantastic **GOAP Action** that makes the AI's decisions interesting.

* **Action:** `Action_BlockWithShield`
* **Preconditions:** `(HasShield)`, `(HasStamina)`, `(PlayerIsInFront)`, `(PlayerIsAttacking)`
* **Effects:** `(DamageIsBlocked)`, `(StaminaIsDraining)`
* **The "Cost":** This action has a `cost` (it drains stamina).

Now, your GOAP planner can make a "smart" decision. If the Player attacks:
* `DodgeRoll` is *safe* (iFrames!) but has a high stamina cost.
* `BlockWithShield` is *efficient* (low cost) but riskier (it just blocks, and what if the player breaks the guard?).

A "tank" teammate's GOAP AI might be programmed to favor `BlockWithShield`, while a "rogue" teammate's AI would be programmed to favor `DodgeRoll`. This is how you create AI "personalities."

---

## 5. General Implementation in Unity

1.  **The "Reactor" (`AI_Reactor.cs`):**
    * A `MonoBehaviour` script.
    * Use `Update()` or `FixedUpdate()` for fast sensor checks (e.g., `SphereCast` to detect nearby players, `RayCast` to check for line of sight).
    * Use Unity's Event System or `OnTriggerEnter` to detect `(IncomingProjectile)`.
    * Have public methods like `HandleImmediateThreat(threatType)` that the main AI controller can call.
    * This script's *only* job is to call "interrupt" functions on the AI's animation/movement controllers (e.g., `animator.SetTrigger("DodgeRoll")`).

2.  **The "Planner" (GOAP):**
    * Use a library like **ReGoap** (from the Unity Asset Store/GitHub). This will save you 100+ hours.
    * It will run its "plan" function *on-demand* (e.g., `planner.RunPlanner()`), **not** in `Update()`.
    * You'll call `RunPlanner()` from a central "brain" script whenever the AI is idle or interrupted.

3.  **The "Actions" (ScriptableObjects):**
    * This is the best architecture. Create a `ScriptableObject` base class called `GoapAction`.
    * Create new actions by right-clicking in the Project window: `Create > AI > GoapAction > Action_FireBow`.
    * In the inspector for `Action_FireBow`, you'll define its **Preconditions** (`HasBow`, `HasAmmo`) and **Effects** (`PlayerHealthReduced`).
    * Your GOAP "Planner" just gets a `List<GoapAction>` of all the actions this *specific* AI knows how to do. This makes your system incredibly flexible.