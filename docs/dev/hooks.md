---
sidebar_position: 5
---

# Hook System

The Hook system is DCCM's core runtime modification mechanism. It allows mods to intercept the game's Hashlink function calls and execute custom C# code before, after, or in place of the original logic.

## Overview

Dead Cells' game logic runs on the Hashlink virtual machine. DCCM intercepts Hashlink function entries at the native layer through IL weaving, forwarding calls to Hook handlers in .NET managed code. Mod developers don't need to worry about the underlying implementation; they only need to declare delegates and register Hooks.

## CreateHook Pattern

The most basic way to register a Hook is through `HashlinkHooks.Instance.CreateHook()`. The standard flow is:

1. **Declare a delegate**: Define a delegate type whose signature matches the target function. The naming convention is `orig_methodName`
2. **Write a Hook handler**: Implement a static method where the **first parameter** is fixed as the delegate type `orig`, with subsequent parameters matching the original function
3. **Register and enable**: Call `CreateHook` to get a `HookHandle`, then call `.Enable()` to activate it

### Delegate Signature Convention

The delegate's return type and parameter list must exactly match the original function. The delegate name begins with `orig_`, which aligns with the Hook handler's design of calling the original logic through the `orig` parameter.

```csharp
// Suppose the target function prototype is: Weapon .method create(Hero hero, InventItem item)
// Delegate declaration:
public delegate Weapon orig_create(Hero hero, InventItem item);

// Hook handler signature:
public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
{
    // Call original logic through orig
    return orig(hero, item);
}
```

`orig` is a delegate wrapper for the original function. Calling `orig(...)` executes the vanilla code; not calling it completely replaces the vanilla behavior.

### Example: Conditionally Replacing Original Logic

The following example from `SampleWeapon` shows how to decide between custom logic and the original logic based on a condition.

`Hook_WeaponCreate.cs`: declares the delegate and Hook handler.

```csharp
using dc.en;

namespace SampleSimple
{
    public static class Hook_WeaponCreate
    {
        // Custom weapon mapping table
        public static Dictionary<string, Func<Hero, InventItem, Weapon>> WeaponCreateMap = new();

        // Delegate declaration: must match target function signature
        public delegate Weapon orig_create(Hero hero, InventItem item);

        // Hook handler: first parameter must be the orig delegate
        public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
        {
            if (WeaponCreateMap.TryGetValue(item._itemData.id.ToString(), out var creator))
            {
                return creator(hero, item);   // Use custom logic
            }
            else
            {
                return orig(hero, item);      // Fall back to original logic
            }
        }
    }
}
```

`SimpleMod.cs`: registers the Hook and populates the mapping table in `Initialize()`.

```csharp
public override void Initialize()
{
    var hooks = HashlinkHooks.Instance;

    // Register and enable Hook: class name, method name, handler
    hooks.CreateHook("tool.$Weapon", "create", Hook_WeaponCreate.Hook_create)
         .Enable();

    // Add custom weapon constructor mapping
    Hook_WeaponCreate.WeaponCreateMap.Add(
        OtherDashSword.name,
        (hero, item) => new OtherDashSword(hero, item)
    );
}
```

Here, the third parameter `enableByDefault` of `CreateHook` defaults to `true`, so `.Enable()` can be omitted, but calling it explicitly is clearer.

### Example: Modifying Before-and-After Logic

The following example from `SampleHook` shows how to add extra processing before and after the original logic executes.

```csharp
using dc.en.hero;
using HaxeProxy.Runtime;

namespace SampleHook
{
    public class SampleHookMod(ModInfo info) : ModBase(info)
    {
        // Helper method: spend money to restore health
        private int AddHealth(int count, double ratio) { /* ... */ }

        // Hook handler: modify parameters then call orig
        private void Hook_beheaded_addMoney(
            Hook_Beheaded.orig_addMoney orig, Beheaded self, int val, Ref<bool> noStats)
        {
            val -= AddHealth(val, 0.5f);   // Spend for health recovery first
            orig(self, val * 10, noStats); // Then call original logic (10x money)
        }

        // Hook handler: execute pre-logic then call orig
        private void Hook_beheaded_addCells(
            Hook_Beheaded.orig_addCells orig, Beheaded self, int val, Ref<bool> noStats)
        {
            bool b = false;
            self.addMoney(val * 20, new(ref b)); // Pre: gold reward
            orig(self, val, noStats);            // Execute original logic
        }

        public override void Initialize()
        {
            // Subscribe to Hook events (no manual Enable needed)
            Hook_Beheaded.addMoney += Hook_beheaded_addMoney;
            Hook_Beheaded.addCells += Hook_beheaded_addCells;
        }
    }
}
```

This example uses another Hook registration method: `Hook_Beheaded` is a wrapper class auto-generated by the framework, exposing the target type's methods as C# events (`addMoney`, `addCells`). Event subscriptions take effect automatically, with no need to call `CreateHook` or `Enable`.

Both approaches are essentially the same: they both declare delegates following the `orig` convention and call the original logic through the `orig` parameter in the handler. The choice depends on preference — `CreateHook` is more explicit, while the generated event wrappers are more concise.

## Disabling Hooks

Calling `HookHandle.Disable()` temporarily turns off a Hook, restoring the original behavior.

```csharp
var handle = HashlinkHooks.Instance.CreateHook("tool.$Weapon", "create", myHandler);
handle.Enable();

// Disable when needed
handle.Disable();
// Re-enable when needed
handle.Enable();
```

## Important Notes

- **Signatures must match exactly**: The delegate's parameter types and order must be completely consistent with the target function, otherwise it will cause a runtime crash.
- **orig is the starting point of a chain call**: Multiple mods may Hook the same function; `orig` will sequentially call the next handler or eventually reach the original function.
- **Avoid infinite loops**: Do not trigger the same hooked function again through other paths from within the Hook handler.
- **Default behavior when manually enabling**: The `enableByDefault` parameter of `CreateHook` defaults to `true`. If explicitly passed as `false`, you must manually call `.Enable()`.

## HarmonyX

### HarmonyX Overview

DCCM has built-in support for [HarmonyX](https://github.com/BepInEx/HarmonyX) (a Harmony fork in the MonoMod ecosystem). Through `HarmonyXModule` (a Preload core module with priority `-999999`), DCCM bridges HarmonyX's `PatchManager.ResolvePatcher` event to `HashlinkFunctionPatcher`. For proxy types marked with `[HashlinkFIndex]`, you can directly use standard `[HarmonyPatch]` attributes to write Hooks without learning DCCM's delegate conventions.

:::info

- **HarmonyX GitHub**: [github.com/BepInEx/HarmonyX](https://github.com/BepInEx/HarmonyX) — HarmonyX is an improved fork of [Harmony](https://github.com/pardeike/Harmony) in the MonoMod ecosystem, maintained by the BepInEx team.
- **Harmony Official Documentation**: [harmony.pardeike.net](https://harmony.pardeike.net/articles/intro.html) — HarmonyX is API-compatible with Harmony; you can directly reference the Harmony official documentation for Patch writing guides.

:::

### Relationship with CreateHook

HarmonyX Patches and Hooks registered via `CreateHook` **share the same Hook chain**. Regardless of the method, all are ultimately sorted and executed by `HashlinkHookManager` in priority order. Both can be **used simultaneously on the same function** without conflict — in `HarmonyXTest.Test_2`, HarmonyX Patches and event-style `Hook_Bounds.load` subscriptions work together on `Bounds.load`, each participating in the chain call according to their declared priorities.

### Basic Usage

#### Declaring a Patch Class

Use `[HarmonyPatch]` to annotate an internal static class, defining `Prefix` and `Postfix` methods within it to intercept the target function. Use `[HarmonyPriority]` to control the execution order of multiple Patches — higher values take precedence.

```csharp
using dc.h2d.col;
using HarmonyLib;

[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(400)]
private static class Patch_MyBoundsLoad
{
    // Prefix: called before the original method executes
    // Return true to continue with the original method; return false to skip it
    static bool Prefix(Bounds b)
    {
        b.xMin = 114514;
        b.yMax = 123;
        return true;
    }

    // Postfix: called after the original method executes
    // __instance retrieves the instance object that the method belongs to
    static void Postfix(Bounds __instance)
    {
        __instance.xMax = 20071003;
    }
}
```

#### Applying and Removing Patches

Call `Harmony.CreateAndPatchAll()` to apply Patches; the returned `Harmony` instance can be used to remove them later:

```csharp
using HarmonyLib;

public override void Initialize()
{
    // Apply all Patches declared in the specified class
    var harmony = Harmony.CreateAndPatchAll(typeof(Patch_MyBoundsLoad));
}

// When removal is needed, undo all Patches registered by this instance
void OnUnload()
{
    harmony.UnpatchSelf();
}
```

#### Priority of Multiple Patches

If multiple mods register Patches for the same function, use `[HarmonyPriority]` to control execution order. Higher priority values **execute first**:

```csharp
[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(400)]  // Executes first
private static class Patch_HighPriority { /* ... */ }

[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(100)]  // Executes later
private static class Patch_LowPriority { /* ... */ }
```

#### Mixing with Event-Style Hooks

HarmonyX Patches can coexist with event-subscription-based DCCM Hooks on the same function. The following code is from `HarmonyXTest.Test_2`:

```csharp
// Apply HarmonyX Patch (priority 400)
var inst1 = Harmony.CreateAndPatchAll(typeof(Patch_HighPriority));

// Apply another HarmonyX Patch (priority 100)
var inst2 = Harmony.CreateAndPatchAll(typeof(Patch_LowPriority));

// Also register a DCCM event-style Hook
Hook_Bounds.load += (orig, self, b) =>
{
    b.xMin = 666;
    orig(self, b);  // Continue calling the next handler in the Hook chain
};
```

All three execute sequentially on the **same Hook chain** according to their respective priorities, without interfering with each other. To remove them, use `-=` to unsubscribe events and `UnpatchSelf()` to remove Patches.

### Comparison with CreateHook

| Feature | HarmonyX | CreateHook |
| ------ | ---------- | ------------ |
| Declaration method | `[HarmonyPatch]` attribute on static class | Explicit call to `HashlinkHooks.Instance.CreateHook()` |
| Method signature | Auto-matched via parameter injection (`__instance`, parameters passed by position) | Must manually declare delegate type; `orig` must be the first parameter |
| Interception mode | Prefix / Postfix / Finalizer, separated by stage | Single handler; decides pre/post/replace by whether `orig` is called |
| Priority mechanism | `[HarmonyPriority]` attribute; higher values execute first | Affected by registration order on the Hook chain |
| Enable/disable | `CreateAndPatchAll()` / `UnpatchSelf()` | `Enable()` / `Disable()` |
| Use cases | Familiar with the Harmony ecosystem; need separate logic for pre and post stages | Simple, direct function interception; need precise control over `orig` call timing and parameters within a single method |
