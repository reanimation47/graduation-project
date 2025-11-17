Here is the refactored architecture. This design locks the weapon and ability kit *to the hero*, giving them a distinct identity, while moving the "Personality" directly into their profile for granular control.

You can save this as `Hero_Kit_Architecture.md`.

-----

# 🧬 Hero Kit & Bespoke AI Architecture

This document outlines the "Hero-Centric" data structure. Instead of generic weapons and generic AI personalities, every "Shaper" (Hero) has a **Fixed Ability Kit** and a **Bespoke AI Bias**.

## 1\. The Abstract Ability (`BaseAbility.cs`)

This is the foundation. Every attack, block, or spell is an "Ability" asset. This supports your idea of **upgrading visuals/effects** easily (just create a "Fireball\_Level2.asset" with bigger VFX and swap it in).

```csharp
// Filename: BaseAbility.cs
using UnityEngine;

public abstract class BaseAbility : ScriptableObject
{
    [Header("Ability Data")]
    public string abilityName = "New Ability";
    [TextArea] public string description;
    public Sprite icon;

    [Header("Costs & Cooldowns")]
    public float staminaCost = 10f;
    public float cooldown = 2f;

    [Header("Visuals (For Upgrades)")]
    [Tooltip("The VFX to spawn when used. Upgraded abilities can have cooler VFX.")]
    public GameObject vfxPrefab;
    public AudioClip sfxClip;

    // The main trigger function
    public abstract void Activate(GameObject user);
}
```

-----

## 2\. The Hero Profile (`ShaperProfile.cs`)

This is the "Source of Truth." It binds the Visuals, the Stats, the Kit, and the AI Brain into one unlockable package.

```csharp
// Filename: ShaperProfile.cs
using UnityEngine;
using System.Collections.Generic;

[CreateAssetMenu(fileName = "Shaper_New", menuName = "Team/Shaper Profile")]
public class ShaperProfile : ScriptableObject
{
    [Header("1. Identity & Visuals")]
    public string shaperName;
    [TextArea] public string shaperBio;
    public GameObject shaperPrefab; // The 3D Model
    public Sprite shaperIcon;

    [Header("2. Base Stats")]
    public float maxHealth = 100f;
    public float maxStamina = 100f;
    public float moveSpeed = 5f;

    [Header("3. The Fixed Kit (Loadout)")]
    // These are references to specific BaseAbility assets.
    // To "Upgrade" a character, you simply swap these assets at runtime.
    
    [Tooltip("Primary Attack (Left Click)")]
    public BaseAbility primaryAbility; 

    [Tooltip("Secondary Ability (Right Click / Block)")]
    public BaseAbility secondaryAbility;

    [Tooltip("Defensive Move (Shift / Dodge)")]
    public BaseAbility defensiveAbility; // e.g., "PhaseDash.asset" or "Roll.asset"

    [Tooltip("Ultimate / Special Move (Q or E)")]
    public BaseAbility specialAbility;

    [Header("4. Bespoke AI Personality")]
    // This replaces the generic "Aggressive" asset. 
    // This list defines EXACTLY how *this* hero prefers to use *their* kit.
    public List<ActionCostModifier> aiPreferences;
    
    [Header("4b. AI Goals")]
    public List<GoapGoal> knownGoals;

    // --- Helper Struct for the AI ---
    [System.Serializable]
    public struct ActionCostModifier
    {
        [Tooltip("Which generic GOAP Action are we modifying?")]
        public GoapAction actionType; 
        // e.g., Action_UsePrimary, Action_UseDefensive

        [Tooltip("Negative = Cheaper (Prefers it). Positive = Expensive (Avoids it).")]
        public float costModifier;
    }
}
```

-----

## 3\. How to Configure "Personalities" (The Data)

Here is how you use the Inspector to create the exact scenarios you described. You don't write code for this; you just tweak the `aiPreferences` list in the hero's profile.

### Example A: "The Weapon Specialist" (Rook)

  * **Concept:** "Prefers to use his weapon more."
  * **Kit:** `primaryAbility` = **Axe Cleave** (High Dmg).
  * **AI Configuration:**
      * `Action_UsePrimary`: **-15 Cost** (Extremely Cheap. He will spam this.)
      * `Action_UseSecondary`: **+5 Cost** (A bit expensive.)
      * `Action_UseDefensive`: **+10 Cost** (He hates dodging.)

### Example B: "The Caster" (Volt)

  * **Concept:** "Prefers to use his first ability (Spell) more."
  * **Kit:** `primaryAbility` = **Weak Wand Shot**. `specialAbility` = **Lightning Bolt**.
  * **AI Configuration:**
      * `Action_UsePrimary`: **+0 Cost** (Neutral.)
      * `Action_UseSpecial`: **-20 Cost** (He will use this *every single time* it is off cooldown.)

### Example C: "The Agile Rogue" (Weaver)

  * **Concept:** "Has a better dodge, so she prefers that over standard rolling."
  * **Kit:** `defensiveAbility` = **Phase Dash** (Teleport with iFrames).
  * **AI Configuration:**
      * `Action_UseDefensive` (Triggers Phase Dash): **-10 Cost** (She loves to dash.)
      * `Action_MoveToTarget`: **+5 Cost** (She prefers dashing to walking.)

-----

## 4\. Future-Proofing: The Upgrade System

Because your Kit uses `ScriptableObject` references, your upgrade system is incredibly scalable.

**How to implement "Visual/Effect Upgrades":**

1.  **Create Variants:**
      * Create `Fireball_Lvl1.asset` (Small orange ball, 10 dmg).
      * Create `Fireball_Lvl2.asset` (Medium blue ball, 20 dmg).
      * Create `Fireball_Lvl3.asset` (Huge purple meteor, 50 dmg + explode sfx).
2.  **Runtime Swapping:**
      * When the player buys an upgrade, your `TeamManager` simply does:
        ```csharp
        // "Upgrading" Volt
        voltProfile.specialAbility = Resources.Load<BaseAbility>("Fireball_Lvl3");
        ```
      * Instantly, the Player (when controlling Volt) sees the new VFX, and the AI (when controlling Volt) deals the new damage. The AI Logic (`Action_UseSpecial`) **does not need to change at all.**