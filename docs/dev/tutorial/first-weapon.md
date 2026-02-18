---
sidebar_position: 3
---

# Creating Your First Weapon

This tutorial will guide you through creating a simple mod that modifies a basic in-game feature.

:::info
The mod code for this tutorial is stored on [Github](https://github.com/dead-cells-core-modding/docs-zh/blob/main/modproject/FirstWeapon).

Most of the code in this section comes from **Frostbite**. Thanks to Frostbite for their assistance with this tutorial.
:::

## Creating the Mod Project

Follow the [tutorial](./first-mod.md) to create a mod project named **FirstWeapon**.

## Creating the res.pak

Follow the [tutorial](https://www.bilibili.com/opus/681993184170999904) to create a **diff pak**. The **CDB** should include:

- A new weapon named **OtherDashSword** and its corresponding **item**

:::tip
**OtherDashSword** can be replaced with any other name.
:::

A ready-made [res.pak](https://github.com/dead-cells-core-modding/docs-zh/blob/main/modproject/FirstWeapon/res.pak).

### Copy the res.pak

- Copy the **res.pak** file obtained in the previous step to the project's root directory.
- Modify **FirstWeapon.csproj** by adding the following content

```xml
<ItemGroup>
    <!-- Copy the res.pak file to the bin\Debug\net10.0 directory -->
    <None Update="res.pak">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>

    <!-- Copy res.pak to the final output directory -->
    <OutputFiles Include="res.pak" />
</ItemGroup>
```

## Writing the Code

### Loading res.pak

- Modify the `FirstWeaponMod.FirstWeapon` class to implement the `IOnGameEndInit` interface.

```csharp
// Load res.pak and refresh CDB
void IOnGameEndInit.OnGameEndInit()
{
    var res = Info.ModRoot!.GetFilePath(“res.pak”); // Get absolute path to res.pak in Mod root directory
    FsPak.Instance.FileSystem.loadPak(res.AsHaxeString()); // Load res.pak
    var json = CDBManager.Class.instance.getAlteredCDB(); // Retrieve merged CDB JSON
    Data.Class.loadJson( // Load merged CDB JSON
       json, 
       default);  
}
```

:::info
After successfully loading **res.pak**, you may see log entries similar to:

```text
[14:30:13 INF][FsPak] Loading pak from C:\SteamLibrary\steamapps\common\Dead Cells\coremod\mods\FirstWeapon\res.pak
```

:::

:::info
The `IOnGameExit` and `IOnGameEndInit` interfaces are part of the **event system**. They are called when their corresponding **events** trigger.

- `IOnGameExit` is called before the game exits.
- `OnGameEndInit` is called after the game initializes.
- `IOnHeroUpdate` is called once per frame during gameplay.

:::

### Creating the Weapon Class

Create a `FirstWeaponMod.OtherDashSwordWeapon` class that inherits from `DashSword` and implements the `IHxbitSerializable<object>` interface.

```csharp
// Weapon Class
internal class OtherDashSwordWeapon : 
    DashSword, // Base Class
    IHxbitSerializable<object>
{

    // Default Constructor
    public OtherDashSwordWeapon(Hero o, InventItem i) : base(o, i)
    {
    }

    // Empty
    object IHxbitSerializable<object>.GetData()
    {
        return new(); // TODO
    }

    // Empty
    void IHxbitSerializable<object>.SetData(object data)
    {
        // TODO
    }

     // Test effect — Increases 10 cells per frame
    public override void fixedUpdate()
    {
        base.fixedUpdate();
        bool noStats = false;
        this.owner.addCells(10, new HaxeProxy.Runtime.Ref<bool>(ref noStats));
    }
}
```

:::info
The `IHxbitSerializable<>` interface is used to save game object data.

For simplicity, the weapon class does not directly inherit from the `Weapon` class (the base class for all weapons), but instead inherits from the `DashSword` class.
:::

### Registering a New Weapon

Simply adding the new weapon's information to the **CDB** is still insufficient.

To enable the game to recognize the new weapon, the `FirstWeaponMod.FirstWeapon` class must also be modified by adding the following:

```csharp
 public override void Initialize()
 {
     Logger.Information(“Hello, world”);

     Hook__Weapon.create += Hook__Weapon_create; // Hook $Weapon.create
 }

 private Weapon Hook__Weapon_create(Hook__Weapon.orig_create orig, dc.en.Hero o, InventItem i)
{
    var id = i._itemData.id.ToString(); // Get weapon ID
    if (id == “OtherDashSword”)
    {
        return new OtherDashSwordWeapon(o, i); // Return custom weapon
    }
    else
    {
        return orig(o, i); // Call original method
    }
}
```

### Acquiring New Weapons

Clearly, we cannot obtain the new weapon yet, as there is currently no way to acquire it.

We can create the item corresponding to the new weapon using the following code:

```csharp
// Spawn item
private void SpawnWeapon(Hero hero)
{
    InventItem testItem = new InventItem(new InventItemKind.Weapon(“OtherDashSword”.AsHaxeString()));
    bool test_boolean = false;
    ItemDrop itemDrop = new ItemDrop(hero._level, hero.cx, hero.cy, testItem, true, new HaxeProxy.Runtime.Ref<bool>(ref test_boolean));
    // Must call init after spawning the drop item, or the game will crash
    itemDrop.init();
    itemDrop.onDropAsLoot();
    itemDrop.dx = hero.dx; // Not sure why this step is needed, but the original code does it this way
}
```

For simplicity, use the following code to obtain a **new weapon** in **the game** by pressing the **backslash** key

```csharp
// Implements IOnHeroUpdate interface
void IOnHeroUpdate.OnHeroUpdate(double dt)
{
    if (Key.Class.isPressed(0xDC /**backslash key code**/))
    {
        SpawnWeapon(Game.Instance.HeroInstance!);
    }
}
```
