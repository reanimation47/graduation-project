This is the final piece of the puzzle to make the AI truly "Smart."

You are absolutely right: **The AI cannot hard-code logic** like "If I have a bow, stand 10 meters away." Because what if you upgrade that bow to a "Shortbow" (5 meters) or a "Sniper" (30 meters)?

The solution is to add a **Targeting Data** struct to your `BaseAbility`. This acts as the "instruction manual" for the AI.

When the AI considers `Action_UsePrimary`, it doesn't look at the weapon model. It reads this instruction manual to answer two questions:

1.  **"Can I hit them?"** (Precondition: Range Check)
2.  **"Where do I aim?"** (Execution: Prediction & Rotation)

-----

### 1\. The Data Structure: `TargetingData`

We modify `BaseAbility.cs` to include a struct that defines the "shape" and "reach" of the ability.

```csharp
// Inside BaseAbility.cs

    [Header("AI Targeting Instructions")]
    public TargetingData aimingData;

    [System.Serializable]
    public struct TargetingData
    {
        public TargetingMode mode; // Defines WHO to target
        public AimType aimStyle;   // Defines HOW to aim (Ballistic, Instant, Area)
        
        [Header("Constraints")]
        public float minRange;     // Don't use if closer than this (e.g. Bows)
        public float maxRange;     // Max effective reach
        
        [Header("Area Details")]
        public float effectRadius; // For AoE / Explosions
        public float coneAngle;    // For Shotguns / Cleaves (e.g., 45 degrees)
        
        [Header("Projectile Info (If applicable)")]
        public float projectileSpeed; // Used for calculating "Lead" (predicting movement)
    }

    public enum TargetingMode { Enemy, Ally, Self, GroundPoint }
    public enum AimType { InstantHit, Projectile, SelfCast, MeleeCleave }
```

-----

### 2\. The "Brain" Logic: How the AI uses this

Now, your generic GOAP actions (`Action_UsePrimary`, `Action_UseSpecial`) become smart wrappers. They don't contain logic; they ask the **Ability** for logic.

Here is how the AI processes the `TargetingData`:

#### A. The Range Check (Precondition)

Before adding `Action_UsePrimary` to a plan, the AI checks `CanHitTarget()`:

1.  **Melee Cleave:** "Is `Distance(Me, Target) <= maxRange`?"
2.  **Ranged Bow:** "Is `Distance(Me, Target) <= maxRange` AND `Distance >= minRange`?"
3.  **Self Buff:** Always returns `true` (Range is 0).

#### B. The Aiming Calculation (Execution)

When the action actually runs, the AI needs to rotate the character.

1.  **InstantHit / Melee:**

      * **Logic:** `LookAt(Target.position)`
      * The AI simply faces the target directly.

2.  **Projectile (The "Smart" Aim):**
    \*

      * **Logic:** The AI reads `projectileSpeed` from the Ability.
      * It calculates an **Intercept Point**. If the target is running left, the AI aims left.
      * *Note:* You don't need complex math. Unity allows you to estimate this: `Target.position + (Target.velocity * timeToImpact)`.

3.  **Cone / Shotgun:**

      * **Logic:** The AI tries to position itself so the target is within the `coneAngle`. Ideally, if there are *multiple* enemies, it calculates the average position of the group to hit the most targets.

-----

### 3\. Implementation: The `AimAssist` Helper

Create a static helper class. This keeps your AI code clean. Both your **Enemy AI** and your **Teammate AI** (and even the Player's auto-aim assist) can use this.

```csharp
// Filename: AimSystem.cs
using UnityEngine;

public static class AimSystem
{
    // Returns the position the AI should look at
    public static Vector3 GetAimPoint(Vector3 shooterPos, Transform target, BaseAbility.TargetingData data)
    {
        switch (data.aimStyle)
        {
            case BaseAbility.AimType.Projectile:
                return CalculateLead(shooterPos, target, data.projectileSpeed);
            
            case BaseAbility.AimType.MeleeCleave:
            case BaseAbility.AimType.InstantHit:
            default:
                return target.position; // Just aim directly
        }
    }

    // Simple predictive aiming
    private static Vector3 CalculateLead(Vector3 shooterPos, Transform target, float projectileSpeed)
    {
        float distance = Vector3.Distance(shooterPos, target.position);
        float travelTime = distance / projectileSpeed;

        // Get target velocity (requires target to have Rigidbody or NavMeshAgent)
        Vector3 targetVelocity = Vector3.zero;
        if(target.TryGetComponent(out Rigidbody rb)) targetVelocity = rb.velocity;
        // else check NavMeshAgent...

        // Predict where they will be
        return target.position + (targetVelocity * travelTime);
    }
    
    // Checks if a target is valid based on the profile
    public static bool IsTargetValid(Vector3 shooterPos, Transform target, BaseAbility.TargetingData data)
    {
        float dist = Vector3.Distance(shooterPos, target.position);
        if (dist > data.maxRange) return false;
        if (dist < data.minRange) return false;
        
        // Optional: Check Line of Sight using Physics.Raycast here
        return true;
    }
}
```

-----

### 4\. Scenario Examples

Here is how this data makes your characters behave differently without writing new AI code:

#### The Sniper (Long Range, Slow Projectile)

  * **Targeting Data:**
      * `MaxRange`: 30m
      * `MinRange`: 8m (Too close\!)
      * `AimStyle`: `Projectile`
      * `Speed`: 50
  * **Behavior:** The AI will refuse to shoot if the enemy rushes it (failing the `MinRange` check). It will likely trigger a `Action_Mobility_Dodge` (Run Away) action instead. When it does shoot, it uses **Predictive Aiming**.

#### The Shotgunner / Firebreather (Cone AoE)

  * **Targeting Data:**
      * `MaxRange`: 5m
      * `AimStyle`: `MeleeCleave`
      * `ConeAngle`: 60
  * **Behavior:** The AI knows it must get *very* close. It won't fire across the map.

#### The Healer (Ally Target)

  * **Targeting Data:**
      * `TargetMode`: `Ally`
      * `MaxRange`: 15m
      * `AimStyle`: `InstantHit`
  * **Behavior:** The GOAP Planner sees `TargetMode: Ally`. It changes its context. Instead of checking `Enemy` distance, it checks `Teammate` distance.

### 5\. Visualizing the Range (Debug Gizmos)

For your development (and your presentation\!), add this to your `BaseAbility` or a testing script. It lets you "see" what the AI sees.

```csharp
// OnDrawGizmosSelected inside your Character Controller
if (currentAbility != null) {
    Gizmos.color = Color.red;
    // Draw Max Range
    Gizmos.DrawWireSphere(transform.position, currentAbility.aimingData.maxRange);
    
    // Draw Min Range (Deadzone)
    Gizmos.color = Color.yellow;
    Gizmos.DrawWireSphere(transform.position, currentAbility.aimingData.minRange);
}
```

This system makes your AI incredibly robust. If you swap a sword for a sniper rifle, the AI *instantly* adapts its positioning and aiming logic because it's just reading the new "Manual" (TargetingData) you equipped.