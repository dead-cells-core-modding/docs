---
sidebar_position: 3
---

# Creating Your First Weapon

This tutorial will walk you through creating a custom weapon, covering the full workflow from CastleDB data configuration and hook registration to final packaging.

:::info
The code in this tutorial comes from **Frostbite**'s SampleWeapon example project.
Join the [Discord server](https://discord.gg/Z7zzVafaP3) for more help.
:::

## Prerequisites

You must have completed the [Creating Your First Mod](./first-mod.md) tutorial and understand these concepts:

- Creating a project with `dotnet new classlib`
- Configuring csproj properties such as `ModType` and `ModMain`
- The `ModBase` base class and the `Initialize()` lifecycle

If you haven't completed it yet, go back to the previous tutorial first.

## Creating the Project

Following the same approach as the previous tutorial, create a class library project named **SampleWeapon** and add the MDK package reference:

```bash
dotnet new classlib -n SampleWeapon -f net10.0
cd SampleWeapon
dotnet add package DeadCellsCoreModding.MDK
```

## Preparing CastleDB

The game uses CastleDB to define all weapon, item, skill, and other data. For the game to recognize a new weapon, you need a **diff CDB** file containing the new weapon's data.

### What Is a Diff CDB

The complete CastleDB data file is `data.cdb`, which contains all vanilla content. You don't need to modify the entire `data.cdb`. Instead, you provide a **diff file** describing only what you've added or changed relative to the vanilla data. The game automatically merges the diff into the vanilla data when loading.

### Configuring GenerateDiffCDB

MDK provides an automatic diff CDB generation feature. Add the following to `SampleWeapon.csproj`:

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <ModName>SampleWeapon</ModName>
    <ModType>mod</ModType>
    <ModMain>SampleSimple.SimpleMod</ModMain>
    <AutoInstallMod>true</AutoInstallMod>
    <Authors>Your Name</Authors>

    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
</PropertyGroup>
```

Key property descriptions:

- **`GenerateDiffCDB`**: When set to `true`, MDK automatically compares `data.cdb` against the vanilla CDB for the specified version during build, producing a diff file containing only the differences and bundling it into `res.pak`.
- **`GameVersion`**: Specifies the game version number that your vanilla CDB corresponds to. This value must match the version of the `data.cdb` you're referencing, otherwise diff generation will fail.

### Placing data.cdb

Place the `data.cdb` file containing the new weapon data in the **project root directory**. This file can be created using the CastleDB editor; see the [Bilibili Basic Mod Creation Tutorial](https://www.bilibili.com/opus/681993184170999904) for details.

A minimal data.cdb should contain:

- A **weapon definition**, such as one named `OtherDashSword`
- A corresponding **item definition** that points to that weapon

During the build, MDK automatically reads this file and generates the diff.

:::tip
If you don't want to edit the CDB yourself just yet, you can use the [res.pak](https://github.com/dead-cells-core-modding/docs-zh/blob/main/modproject/FirstWeapon/res.pak) included with this tutorial and skip straight to the code. Simply disable `GenerateDiffCDB` in your csproj and manually copy `res.pak`.
:::

## Asset Packaging

In addition to data, a weapon needs assets like textures and animations. These go in the `assets/` directory and are packaged via the `PackAssets` configuration in csproj.

```xml
<ItemGroup>
    <PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
</ItemGroup>

<ItemGroup>
    <PackAssets Include="assets/**/*" RootInPak="sample_simple" />
</ItemGroup>
```

- **`PackAssets Include`**: Specifies the asset files to package. `assets/**/*` means all files under the `assets` directory.
- **`RootInPak`**: Specifies the root path for the assets inside the pak. Here it's set to `sample_simple`, so in-game paths will be `sample_simple/xxx`.

After the build, MDK automatically invokes `GenerateSinglePakFile`, merging the CDB diff and assets into a single `res.pak` placed in the final output directory.

## Hooking Weapon Creation

When the game creates a weapon object, it calls the `tool.$Weapon.create()` method. We need to hook this method and inject our custom weapon creation logic.

### The Hook_WeaponCreate Pattern

Create a `Hook_WeaponCreate.cs` file:

```csharp
using dc.en;
using dc.pr;
using dc.shader;
using dc.tool;

namespace SampleSimple
{
    public static class Hook_WeaponCreate
    {
        // Mapping table: weapon name → creation function
        public static Dictionary<string, Func<Hero, InventItem, Weapon>> WeaponCreateMap
            = new Dictionary<string, Func<Hero, InventItem, Weapon>>();

        // Delegate: matches the signature of the original create method
        public delegate Weapon orig_create(Hero hero, InventItem item);

        // Hook function
        public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
        {
            // Look up a custom creation function in the mapping table
            if (WeaponCreateMap.TryGetValue(item._itemData.id.ToString(), out var creator))
            {
                return creator(hero, item);  // use custom logic
            }
            else
            {
                return orig(hero, item);     // fall back to vanilla logic
            }
        }
    }
}
```

**How it works**:

1. `WeaponCreateMap` is a dictionary where the key is the weapon name (string) and the value is a factory function that creates the weapon object.
2. `orig_create` is a delegate type matching the original signature of `$Weapon.create`.
3. `Hook_create` is the actual hook function. When the game calls `$Weapon.create`, it enters this function first.
4. It looks up the weapon ID in the mapping table: **if a custom creation function is found**, it uses that to create the weapon. **If not found**, it calls `orig(hero, item)` to fall back to vanilla logic.
5. The key is the `orig` parameter; it's a reference to the original function. Not calling `orig` means completely replacing the vanilla behavior; calling `orig` extends the vanilla behavior.

### Registering Hooks and Mappings

Register the hook in `SimpleMod.Initialize()`:

```csharp
public override void Initialize()
{
    // Add hook
    var hooks = HashlinkHooks.Instance;
    hooks.CreateHook("tool.$Weapon", "create", Hook_WeaponCreate.Hook_create).Enable();

    // Bind custom weapon name to its creation function
    Hook_WeaponCreate.WeaponCreateMap.Add(
        OtherDashSword.name,
        (hero, item) => new OtherDashSword(hero, item)
    );
}
```

- **`HashlinkHooks.Instance.CreateHook(...)`**: Creates a hook on a HashLink function. Parameters are: class name, method name, hook function.
- **`.Enable()`**: Activates the hook.
- **`WeaponCreateMap.Add(...)`**: Registers the weapon name `"OtherDashSword"` in the mapping table, pointing to the `OtherDashSword` constructor.

## Custom Weapon Class

Create `OtherDashSword.cs` and define the weapon's behavior:

```csharp
using dc;
using dc.en;
using dc.tool;
using dc.tool.weap;
using HaxeProxy.Runtime;
using ModCore.Storage;

namespace SampleSimple
{
    public class OtherDashSword : DashSword, IHxbitSerializable<object>
    {
        // Weapon name, must match the definition in data.cdb
        public static string name = "OtherDashSword";

        // Constructor, takes Hero and InventItem
        public OtherDashSword(Hero hero, InventItem item) : base(hero, item)
        {
        }

        // Per-frame update: adds 10 cells on top of vanilla logic
        public override void fixedUpdate()
        {
            base.fixedUpdate();
            bool noStats = false;
            this.owner.addCells(10, new HaxeProxy.Runtime.Ref<bool>(ref noStats));
        }

        // IHxbitSerializable interface: for save serialization
        object IHxbitSerializable<object>.GetData()
        {
            return new();
        }

        void IHxbitSerializable<object>.SetData(object data)
        {
        }
    }
}
```

**Key points**:

- **Inheriting a base class**: Here we inherit `DashSword` rather than `Weapon` directly. You can choose any existing weapon as the base class. `DashSword` includes the full dashing lunge logic; you only override what you want to change.
- **`name` field**: Must match exactly the weapon name defined in `data.cdb`. This name is used for both mapping table registration and `InventItem` creation.
- **`IHxbitSerializable<object>`**: Serialization interface for saving/restoring weapon state in save files. If the weapon has no extra data to save, `GetData()` and `SetData()` can be left empty.
- **`fixedUpdate()`**: Called every frame. Calling `base.fixedUpdate()` preserves the base class's behavior, then appends new logic.

## Loading Assets and Obtaining the Weapon

### Loading res.pak

Implement `IOnAfterLoadingAssets` in `SimpleMod` to load the pak after the game loads assets:

```csharp
public class SimpleMod(ModInfo info) : ModBase(info),
    IOnHeroUpdate,
    IOnAfterLoadingAssets
{
    // ...

    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        var res = Info.ModRoot!.GetFilePath("res.pak");
        FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
    }
}
```

### Obtaining the Weapon

Press the backslash key to spawn the weapon for testing:

```csharp
[DllImport("user32.dll", CharSet = CharSet.Auto, ExactSpelling = true)]
private static extern short GetAsyncKeyState(int vkey);

private static bool isLastFramePressed = false;
private const int VK_OEM_5 = 0xDC; // backslash key code

void IOnHeroUpdate.OnHeroUpdate(double dt)
{
    bool isCurrentFramePressed = (GetAsyncKeyState(VK_OEM_5) & 0x8000) != 0;
    Hero hero = Game.Instance.HeroInstance!;
    if (isCurrentFramePressed && !isLastFramePressed && hero != null)
    {
        SpawnWeapon(hero);
    }
    isLastFramePressed = isCurrentFramePressed;
}

private void SpawnWeapon(Hero hero)
{
    InventItem testItem = new InventItem(
        new InventItemKind.Weapon(OtherDashSword.name.AsHaxeString())
    );
    bool test_boolean = false;
    ItemDrop itemDrop = new ItemDrop(
        hero._level, hero.cx, hero.cy,
        testItem, true,
        new HaxeProxy.Runtime.Ref<bool>(ref test_boolean)
    );
    // Must call init after creating a drop, otherwise the game crashes
    itemDrop.init();
    itemDrop.onDropAsLoot();
    itemDrop.dx = hero.dx;
}
```

:::info
`GetAsyncKeyState` combined with `isLastFramePressed` implements key press detection, ensuring it only triggers on the first frame of the press rather than every frame.
:::

## Complete Code

### SimpleMod.cs

```csharp
using dc;
using dc.en;
using ModCore.Events.Interfaces;
using ModCore.Events.Interfaces.Game;
using ModCore.Events.Interfaces.Game.Hero;
using ModCore.Mods;
using ModCore.Modules;
using ModCore.Utilities;
using Serilog;
using System.Runtime.InteropServices;

namespace SampleSimple
{
    public class SimpleMod(ModInfo info) : ModBase(info),
        IOnHeroUpdate,
        IOnAfterLoadingAssets
    {
        [DllImport("user32.dll", CharSet = CharSet.Auto, ExactSpelling = true)]
        private static extern short GetAsyncKeyState(int vkey);

        private static bool isLastFramePressed = false;
        private const int VK_OEM_5 = 0xDC;

        public override void Initialize()
        {
            Logger.Information("Frostbite greets you!");

            var hooks = HashlinkHooks.Instance;
            hooks.CreateHook("tool.$Weapon", "create",
                Hook_WeaponCreate.Hook_create).Enable();

            Hook_WeaponCreate.WeaponCreateMap.Add(
                OtherDashSword.name,
                (hero, item) => new OtherDashSword(hero, item)
            );
        }

        void IOnHeroUpdate.OnHeroUpdate(double dt)
        {
            bool isCurrentFramePressed = (GetAsyncKeyState(VK_OEM_5) & 0x8000) != 0;
            Hero hero = Game.Instance.HeroInstance!;
            if (isCurrentFramePressed && !isLastFramePressed && hero != null)
            {
                SpawnWeapon(hero);
            }
            isLastFramePressed = isCurrentFramePressed;
        }

        private void SpawnWeapon(Hero hero)
        {
            InventItem testItem = new InventItem(
                new InventItemKind.Weapon(OtherDashSword.name.AsHaxeString())
            );
            bool test_boolean = false;
            ItemDrop itemDrop = new ItemDrop(
                hero._level, hero.cx, hero.cy,
                testItem, true,
                new HaxeProxy.Runtime.Ref<bool>(ref test_boolean)
            );
            itemDrop.init();
            itemDrop.onDropAsLoot();
            itemDrop.dx = hero.dx;
        }

        void IOnAfterLoadingAssets.OnAfterLoadingAssets()
        {
            var res = Info.ModRoot!.GetFilePath("res.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
        }
    }
}
```

### Hook_WeaponCreate.cs

```csharp
using dc.en;
using dc.tool;

namespace SampleSimple
{
    public static class Hook_WeaponCreate
    {
        public static Dictionary<string, Func<Hero, InventItem, Weapon>> WeaponCreateMap
            = new Dictionary<string, Func<Hero, InventItem, Weapon>>();

        public delegate Weapon orig_create(Hero hero, InventItem item);

        public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
        {
            if (WeaponCreateMap.TryGetValue(item._itemData.id.ToString(), out var creator))
            {
                return creator(hero, item);
            }
            else
            {
                return orig(hero, item);
            }
        }
    }
}
```

### OtherDashSword.cs

```csharp
using dc;
using dc.en;
using dc.tool;
using dc.tool.weap;
using HaxeProxy.Runtime;
using ModCore.Storage;

namespace SampleSimple
{
    public class OtherDashSword : DashSword, IHxbitSerializable<object>
    {
        public static string name = "OtherDashSword";

        public OtherDashSword(Hero hero, InventItem item) : base(hero, item)
        {
        }

        public override void fixedUpdate()
        {
            base.fixedUpdate();
            bool noStats = false;
            this.owner.addCells(10, new HaxeProxy.Runtime.Ref<bool>(ref noStats));
        }

        object IHxbitSerializable<object>.GetData()
        {
            return new();
        }

        void IHxbitSerializable<object>.SetData(object data)
        {
        }
    }
}
```

### SampleWeapon.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <ModName>SampleWeapon</ModName>
    <ModType>mod</ModType>
    <ModMain>SampleSimple.SimpleMod</ModMain>
    <AutoInstallMod>true</AutoInstallMod>
    <Authors>Your Name</Authors>

    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
  </ItemGroup>

  <ItemGroup>
    <PackAssets Include="assets/**/*" RootInPak="sample_simple" />
  </ItemGroup>
</Project>
```

## Building and Testing

```bash
dotnet build
```

After a successful build, the mod files and the auto-packaged `res.pak` will be generated in the `bin\Debug\net10.0\output` directory.

Launch the game via `DeadCellsModding.exe`, enter a level, and press the backslash key. An `OtherDashSword` will spawn at the character's feet. Upon picking it up, you'll automatically gain 10 cells per frame.

## FAQ

### Q1: GenerateDiffCDB Fails During Build

- Make sure `GameVersion` matches the game version of your `data.cdb`
- Make sure the `data.cdb` file exists in the project root directory

### Q2: Pressing the Key Doesn't Spawn the Weapon

- Check that `IOnHeroUpdate` is implemented and correctly registered in the csproj
- Confirm that `res.pak` is loaded; look for `Loading pak from ...` in the logs
- Check that `OtherDashSword.name` matches the id in data.cdb

### Q3: Game Crashes When Picking Up the Weapon

- Make sure `ItemDrop.init()` is called after creating the drop
- Check that the weapon class implements the `IHxbitSerializable<object>` interface
- Check the exception stack trace in the logs
