---
sidebar_position: 3
---

# 创造第一把武器

本教程将带你创建一把自定义武器，从 CastleDB 数据配置、Hook 注册到最终打包，覆盖完整流程。

:::info
本教程代码来自 **Frostbite** 的 SampleWeapon 示例项目。
加入 [Discord 服务器](https://discord.gg/Z7zzVafaP3) 获取更多帮助。
:::

## 前置要求

你必须已完成 [创造第一个 Mod](./first-mod.md) 教程，理解以下概念：

- 使用 `dotnet new classlib` 创建项目
- 配置 `ModType`、`ModMain` 等 csproj 属性
- `ModBase` 基类与 `Initialize()` 生命周期

如果还没完成，请先回到上一篇教程。

## 创建项目

按照前一篇教程的方式，创建名为 **SampleWeapon** 的类库项目，并添加 MDK 包引用：

```bash
dotnet new classlib -n SampleWeapon -f net10.0
cd SampleWeapon
dotnet add package DeadCellsCoreModding.MDK
```

## 准备 CastleDB

游戏通过 CastleDB 来定义所有武器、物品、技能等数据。要让游戏识别新武器，你需要一个包含新武器数据的 **diff CDB** 文件。

### 什么是 Diff CDB

CastleDB 的完整数据文件是 `data.cdb`，包含了原版所有内容。你不需要修改整个 `data.cdb`，只需要提供一个 **diff（差异）文件**，描述你对原版数据做了哪些增改。游戏加载时会自动将 diff 合并到原版数据上。

### 配置 GenerateDiffCDB

MDK 提供了自动生成 diff CDB 的功能。在 `SampleWeapon.csproj` 中添加：

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <ModName>SampleWeapon</ModName>
    <ModType>mod</ModType>
    <ModMain>SampleSimple.SimpleMod</ModMain>
    <AutoInstallMod>true</AutoInstallMod>
    <Authors>你的名字</Authors>

    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
</PropertyGroup>
```

关键属性说明：

- **`GenerateDiffCDB`**：设为 `true` 后，构建时 MDK 会自动将 `data.cdb` 与对应版本的原版 CDB 做比对，生成只包含差异的 diff 文件并打包进 `res.pak`。
- **`GameVersion`**：指定你使用的原版 CDB 对应的游戏版本号。这个值必须和你引用的 `data.cdb` 版本一致，否则 diff 生成会出错。

### 放置 data.cdb

将包含新武器数据的 `data.cdb` 文件放到**项目根目录**下。这个文件可以通过 CastleDB 编辑器创建，具体方法参考 [Bilibili 基础 Mod 制作教程](https://www.bilibili.com/opus/681993184170999904)。

一个最简单的 data.cdb 应包含：

- 一个 **武器定义**（weapon），比如名为 `OtherDashSword`
- 一个对应的 **物品定义**（item），指向该武器

构建时，MDK 会自动读取这个文件并生成 diff。

:::tip
如果暂时不想自己编辑 CDB，可以用教程附带的 [res.pak](https://github.com/dead-cells-core-modding/docs-zh/blob/main/modproject/FirstWeapon/res.pak) 直接跳到代码部分。只需在 csproj 中关闭 GenerateDiffCDB 并手动复制 res.pak 即可。
:::

## 资源打包

武器除了数据还需要贴图、动画等资源。这些资源放在 `assets/` 目录下，通过 csproj 中的 `PackAssets` 配置来打包。

```xml
<ItemGroup>
    <PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
</ItemGroup>

<ItemGroup>
    <PackAssets Include="assets/**/*" RootInPak="sample_simple" />
</ItemGroup>
```

- **`PackAssets Include`**：指定要打包的资源文件，`assets/**/*` 表示 assets 目录下所有文件。
- **`RootInPak`**：指定资源在 pak 内的根路径。这里设为 `sample_simple`，加载后游戏内路径为 `sample_simple/xxx`。

构建后，MDK 会自动调用 `GenerateSinglePakFile`，将 CDB diff 和 assets 资源合并输出为一个 `res.pak`，放到最终输出目录。

## Hook 武器创建

游戏在创建武器对象时，会调用 `tool.$Weapon.create()` 方法。我们需要 Hook 这个方法，将自定义武器的创建逻辑插入进去。

### Hook_WeaponCreate 模式

创建一个 `Hook_WeaponCreate.cs` 文件：

```csharp
using dc.en;
using dc.pr;
using dc.shader;
using dc.tool;

namespace SampleSimple
{
    public static class Hook_WeaponCreate
    {
        // 映射表：武器名称 → 创建函数
        public static Dictionary<string, Func<Hero, InventItem, Weapon>> WeaponCreateMap
            = new Dictionary<string, Func<Hero, InventItem, Weapon>>();

        // 委托：匹配原版 create 方法的签名
        public delegate Weapon orig_create(Hero hero, InventItem item);

        // Hook 函数
        public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
        {
            // 在映射表中查找自定义创建函数
            if (WeaponCreateMap.TryGetValue(item._itemData.id.ToString(), out var creator))
            {
                return creator(hero, item);  // 使用自定义逻辑
            }
            else
            {
                return orig(hero, item);     // 回退到原版逻辑
            }
        }
    }
}
```

**工作原理**：

1. `WeaponCreateMap` 是一个字典，key 是武器名称（字符串），value 是创建该武器对象的工厂函数。
2. `orig_create` 是委托类型，匹配 `$Weapon.create` 的原始签名。
3. `Hook_create` 是实际的 Hook 函数。当游戏调用 `$Weapon.create` 时，会先进入这个函数。
4. 它根据武器 ID 查找映射表：**如果找到**自定义创建函数，就用它来创建武器。**如果没找到**，就调用 `orig(hero, item)` 走原版逻辑。
5. 关键在于 `orig` 参数；它是对原始函数的引用。不调用 `orig` 就等于完全替换了原版行为；调用 `orig` 则是在原版基础上扩展。

### 注册 Hook 和映射

在 `SimpleMod.Initialize()` 中注册 Hook：

```csharp
public override void Initialize()
{
    // 添加 Hook
    var hooks = HashlinkHooks.Instance;
    hooks.CreateHook("tool.$Weapon", "create", Hook_WeaponCreate.Hook_create).Enable();

    // 将自定义武器名称与创建函数绑定
    Hook_WeaponCreate.WeaponCreateMap.Add(
        OtherDashSword.name,
        (hero, item) => new OtherDashSword(hero, item)
    );
}
```

- **`HashlinkHooks.Instance.CreateHook(...)`**：创建一个对 HashLink 函数的 Hook。参数依次为：类名、方法名、Hook 函数。
- **`.Enable()`**：激活 Hook。
- **`WeaponCreateMap.Add(...)`**：将武器名称 `"OtherDashSword"` 注册到映射表，指向 `OtherDashSword` 的构造函数。

## 自定义武器类

创建 `OtherDashSword.cs`，定义武器行为：

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
        // 武器名称，需要和 data.cdb 中的定义一致
        public static string name = "OtherDashSword";

        // 构造函数，接收 Hero 和 InventItem
        public OtherDashSword(Hero hero, InventItem item) : base(hero, item)
        {
        }

        // 每帧更新：在原版逻辑基础上额外增加 10 细胞
        public override void fixedUpdate()
        {
            base.fixedUpdate();
            bool noStats = false;
            this.owner.addCells(10, new HaxeProxy.Runtime.Ref<bool>(ref noStats));
        }

        // IHxbitSerializable 接口：用于存档序列化
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

**要点**：

- **继承基类**：这里继承 `DashSword`（突刺剑）而非直接继承 `Weapon`。你可以根据需求选择任意现有武器作为基类。`DashSword` 包含了闪避突刺的完整逻辑，你只需 override 想改的部分。
- **`name` 字段**：必须和 `data.cdb` 中定义的武器名称完全一致。这个名称会同时用于映射表注册和 `InventItem` 创建。
- **`IHxbitSerializable<object>`**：序列化接口，用于在存档中保存/恢复武器状态。如果武器没有额外数据需要保存，`GetData()` 和 `SetData()` 可以留空。
- **`fixedUpdate()`**：每帧调用。调用 `base.fixedUpdate()` 保留基类原有行为，然后附加新逻辑。

## 加载资源与获取武器

### 加载 res.pak

在 `SimpleMod` 中实现 `IOnAfterLoadingAssets`，在游戏加载资源后载入 pak：

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

### 获取武器

通过按下反斜杠键来生成武器，方便测试：

```csharp
[DllImport("user32.dll", CharSet = CharSet.Auto, ExactSpelling = true)]
private static extern short GetAsyncKeyState(int vkey);

private static bool isLastFramePressed = false;
private const int VK_OEM_5 = 0xDC; // 反斜杠键码

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
    // 生成掉落物后必须调用 init，否则游戏崩溃
    itemDrop.init();
    itemDrop.onDropAsLoot();
    itemDrop.dx = hero.dx;
}
```

:::info
`GetAsyncKeyState` 配合 `isLastFramePressed` 实现按键检测，确保只在按下的第一帧触发而非每帧重复触发。
:::

## 完整代码

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
            Logger.Information("Frostbite向你问候!");

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
    <Authors>你的名字</Authors>

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

## 构建与测试

```bash
dotnet build
```

构建成功后，在 `bin\Debug\net10.0\output` 目录下会生成 Mod 文件以及自动打包好的 `res.pak`。

通过 `DeadCellsModding.exe` 启动游戏，进入关卡后按下反斜杠键，角色脚下会生成一把 `OtherDashSword`。拾取后每帧自动获得 10 细胞。

## 常见问题

### Q1: 构建时 GenerateDiffCDB 报错

- 确认 `GameVersion` 与 `data.cdb` 对应的游戏版本一致
- 确认 `data.cdb` 文件存在于项目根目录

### Q2: 按下按键没有武器掉落

- 检查 `IOnHeroUpdate` 是否已实现并在 csproj 中正确注册
- 确认 `res.pak` 已加载，查看日志中是否有 `Loading pak from ...` 字样
- 检查 `OtherDashSword.name` 与 data.cdb 中的 id 是否一致

### Q3: 拾取武器后游戏崩溃

- 确认 `ItemDrop.init()` 在生成掉落物后被调用
- 检查武器类是否实现了 `IHxbitSerializable<object>` 接口
- 查看日志中的异常堆栈
