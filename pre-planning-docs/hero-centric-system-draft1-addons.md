Here is the updated `ShaperProfile.cs`.

I have renamed the field to **`mobilityAbility`** and updated the tooltips to reflect that this single ability serves two distinct purposes for the AI (Engage vs. Escape).

```csharp
// Filename: ShaperProfile.cs
using UnityEngine;
using System.Collections.Generic;

[CreateAssetMenu(fileName = "Shaper_New", menuName = "Team/Shaper Profile")]
public class ShaperProfile : ScriptableObject
{
    [Header("1. Identity & Visuals")]
    public string shaperName = "New Shaper";
    [TextArea(2, 4)] public string shaperBio;
    
    [Tooltip("The 3D Model Prefab to spawn.")]
    public GameObject shaperPrefab; 
    public Sprite shaperIcon;

    [Header("2. Base Stats")]
    public float maxHealth = 100f;
    public float maxStamina = 100f;
    public float moveSpeed = 5f;

    [Header("3. The Fixed Kit (Loadout)")]
    [Tooltip("Primary Attack (Left Click). Usually a low-cost, spammable attack.")]
    public BaseAbility primaryAbility; 

    [Tooltip("Secondary Ability (Right Click). Block, Parry, or Heavy Attack.")]
    public BaseAbility secondaryAbility;

    [Tooltip("Mobility Skill (Shift). Dash, Roll, Teleport.\n" +
             "The AI uses this for TWO generic actions: 'Mobility_Engage' and 'Mobility_Dodge'.")]
    public BaseAbility mobilityAbility; // <--- THE RENAMED FIELD

    [Tooltip("Ultimate / Special Move (Q/E). High impact, high cooldown.")]
    public BaseAbility specialAbility;

    [Header("4. Bespoke AI Personality")]
    [Tooltip("Defines how this specific hero prefers to use their kit.\n" +
             "Example: Make 'Mobility_Engage' cheap to create an aggressive dasher.")]
    public List<ActionCostModifier> aiPreferences;
    
    [Header("4b. AI Goals")]
    [Tooltip("What strategic goals can this hero pursue? (e.g. KillPlayer, ProtectClient)")]
    public List<GoapGoal> knownGoals;

    // --- Helper Struct for the AI Preferences ---
    [System.Serializable]
    public struct ActionCostModifier
    {
        [Tooltip("The Generic GOAP Action type (e.g., Action_Mobility_Engage)")]
        public GoapAction actionType; 

        [Tooltip("Negative = Cheaper (Prefers it). Positive = Expensive (Avoids it).")]
        public float costModifier;
    }
}
```

### How to setup the AI for this new field:

Now, when you configure your **`aiPreferences`** list in the Inspector, you can map this single ability to two different behaviors:

1.  **To make an "Assassin" (Offensive Dasher):**
      * `Action_Mobility_Engage`: **-10 Cost** (Cheap)
      * `Action_Mobility_Dodge`: **+5 Cost** (Expensive)
2.  **To make a "Ranger" (Defensive Kiter):**
      * `Action_Mobility_Engage`: **+50 Cost** (Prohibited)
      * `Action_Mobility_Dodge`: **-10 Cost** (Cheap)

The `mobilityAbility` asset (the actual code that moves the character) doesn't change; only the AI's *intention* for using it changes.