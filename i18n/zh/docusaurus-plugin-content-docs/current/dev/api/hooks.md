---
sidebar_position: 2
---

# Hook 系统

Hook 系统是 DCCM 最核心的运行时修改机制。它允许模组拦截游戏的 Hashlink 函数调用，在原始逻辑执行前、后或代替原始逻辑执行自定义的 C# 代码。

## 概述

Dead Cells 的游戏逻辑运行在 Hashlink 虚拟机上。DCCM 通过 IL 织入技术在原生层拦截 Hashlink 函数入口，将调用转发到 .NET 托管代码中的 Hook 处理器。模组开发者无需关心底层实现，只需声明委托并注册 Hook 即可。

## CreateHook 模式

最基本的 Hook 注册方式是通过 `HashlinkHooks.Instance.CreateHook()` 完成。标准流程为：

1. **声明委托**：定义一个与目标函数签名一致的委托类型，命名约定为 `orig_方法名`
2. **编写 Hook 处理器**：实现一个静态方法，**第一个参数**固定为委托类型的 `orig`，后续参数与原函数一致
3. **注册并启用**：调用 `CreateHook` 后返回 `HookHandle`，调用 `.Enable()` 激活

### 委托签名约定

委托的返回值类型和参数列表必须与原函数完全匹配。委托名以 `orig_` 开头，这与 Hook 处理器通过 `orig` 参数调用原始逻辑的设计保持一致。

```csharp
// 假设目标函数原型为: Weapon .method create(Hero hero, InventItem item)
// 委托声明:
public delegate Weapon orig_create(Hero hero, InventItem item);

// Hook 处理器签名:
public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
{
    // 通过 orig 调用原始逻辑
    return orig(hero, item);
}
```

`orig` 是原始函数的一个委托包装。调用 `orig(...)` 会执行原版代码；不调用则完全替换原版行为。

### 示例：条件替换原始逻辑

以下示例来自 `SampleWeapon`，展示如何根据条件决定走自定义逻辑还是原始逻辑。

`Hook_WeaponCreate.cs`：声明委托和 Hook 处理器。

```csharp
using dc.en;

namespace SampleSimple
{
    public static class Hook_WeaponCreate
    {
        // 自定义武器映射表
        public static Dictionary<string, Func<Hero, InventItem, Weapon>> WeaponCreateMap = new();

        // 委托声明：与目标函数签名一致
        public delegate Weapon orig_create(Hero hero, InventItem item);

        // Hook 处理器：第一个参数必须是 orig 委托
        public static Weapon Hook_create(orig_create orig, Hero hero, InventItem item)
        {
            if (WeaponCreateMap.TryGetValue(item._itemData.id.ToString(), out var creator))
            {
                return creator(hero, item);   // 使用自定义逻辑
            }
            else
            {
                return orig(hero, item);      // 回退到原始逻辑
            }
        }
    }
}
```

`SimpleMod.cs`：在 `Initialize()` 中注册 Hook 并填充映射表。

```csharp
public override void Initialize()
{
    var hooks = HashlinkHooks.Instance;

    // 注册并启用 Hook：类名、方法名、处理器
    hooks.CreateHook("tool.$Weapon", "create", Hook_WeaponCreate.Hook_create)
         .Enable();

    // 添加自定义武器的构造映射
    Hook_WeaponCreate.WeaponCreateMap.Add(
        OtherDashSword.name,
        (hero, item) => new OtherDashSword(hero, item)
    );
}
```

这里 `CreateHook` 的第三个参数 `enableByDefault` 默认为 `true`，所以 `.Enable()` 可以省略，但显式调用更清晰。

### 示例：修改前后逻辑

以下示例来自 `SampleHook`，展示如何在原始逻辑执行前后添加额外处理。

```csharp
using dc.en.hero;
using HaxeProxy.Runtime;

namespace SampleHook
{
    public class SampleHookMod(ModInfo info) : ModBase(info)
    {
        // 辅助方法：消耗金钱回复生命值
        private int AddHealth(int count, double ratio) { /* ... */ }

        // Hook 处理器：修改参数后调用 orig
        private void Hook_beheaded_addMoney(
            Hook_Beheaded.orig_addMoney orig, Beheaded self, int val, Ref<bool> noStats)
        {
            val -= AddHealth(val, 0.5f);   // 先扣血回复
            orig(self, val * 10, noStats); // 再调用原始逻辑（金钱翻十倍）
        }

        // Hook 处理器：执行前置逻辑后调用 orig
        private void Hook_beheaded_addCells(
            Hook_Beheaded.orig_addCells orig, Beheaded self, int val, Ref<bool> noStats)
        {
            bool b = false;
            self.addMoney(val * 20, new(ref b)); // 前置：金币奖励
            orig(self, val, noStats);            // 执行原始逻辑
        }

        public override void Initialize()
        {
            // 订阅 Hook 事件（无需手动 Enable）
            Hook_Beheaded.addMoney += Hook_beheaded_addMoney;
            Hook_Beheaded.addCells += Hook_beheaded_addCells;
        }
    }
}
```

这个示例使用了另一种 Hook 注册方式：`Hook_Beheaded` 是框架自动生成的包装类，将目标类型的方法暴露为 C# 事件（`addMoney`、`addCells`）。事件订阅后自动生效，无需调用 `CreateHook` 或 `Enable`。

两种方式本质相同：都是声明符合 `orig` 约定的委托，并在处理器中通过 `orig` 参数调用原始逻辑。选择哪种取决于使用习惯——`CreateHook` 更显式，生成的事件包装更简洁。

## 禁用 Hook

调用 `HookHandle.Disable()` 可临时关闭一个 Hook，恢复原始行为。

```csharp
var handle = HashlinkHooks.Instance.CreateHook("tool.$Weapon", "create", myHandler);
handle.Enable();

// 需要时禁用
handle.Disable();
// 需要时重新启用
handle.Enable();
```

## 注意事项

- **签名必须精确匹配**：委托的参数类型和顺序必须与目标函数完全一致，否则会导致运行时崩溃。
- **orig 是链式调用的起点**：多个模组可能 Hook 同一个函数，`orig` 会依次调用下一个处理器或最终到达原始函数。
- **避免死循环**：在 Hook 处理器内部不要通过其他路径再次触发同一个被 Hook 的函数。
- **手动 Enable 时的默认行为**：`CreateHook` 的 `enableByDefault` 参数默认为 `true`。如果显式传入 `false`，则需要手动调用 `.Enable()`。

# HarmonyX

## 概述

DCCM 内置了对 [HarmonyX](https://github.com/BepInEx/HarmonyX)（MonoMod 生态下的 Harmony 分支）的支持。通过 `HarmonyXModule`（优先级 `-999999` 的 Preload 核心模块），DCCM 将 HarmonyX 的 `PatchManager.ResolvePatcher` 事件桥接到 `HashlinkFunctionPatcher`。对于标记了 `[HashlinkFIndex]` 的代理类型，可直接使用标准 `[HarmonyPatch]` 属性编写 Hook，无需学习 DCCM 的委托约定。

:::info 参考链接

- **HarmonyX GitHub**: https://github.com/BepInEx/HarmonyX — HarmonyX 是 MonoMod 生态中 [Harmony](https://github.com/pardeike/Harmony) 的改进分支，由 BepInEx 团队维护。
- **Harmony 官方文档**: https://harmony.pardeike.net/articles/intro.html — HarmonyX 与 Harmony API 兼容，可直接参考 Harmony 官方文档中的 Patch 编写指南。

:::

## 与 CreateHook 的关系

HarmonyX Patch 与通过 `CreateHook` 注册的 Hook **共用同一 Hook 链**。无论哪种方式，最终都由 `HashlinkHookManager` 按优先级统一排序执行。两者可以**在同一函数上同时使用**，互不冲突——在 `HarmonyXTest.Test_2` 中，HarmonyX Patch 和事件式 `Hook_Bounds.load` 订阅共同作用于 `Bounds.load`，各自按声明的优先级参与链式调用。

## 基本用法

### 声明 Patch 类

使用 `[HarmonyPatch]` 标注一个内部静态类，在其中定义 `Prefix` 和 `Postfix` 方法拦截目标函数。通过 `[HarmonyPriority]` 控制多个 Patch 的执行顺序，值越大越优先。

```csharp
using dc.h2d.col;
using HarmonyLib;

[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(400)]
private static class Patch_MyBoundsLoad
{
    // Prefix：在原方法执行前调用
    // 返回 true 继续执行原方法；返回 false 跳过原方法
    static bool Prefix(Bounds b)
    {
        b.xMin = 114514;
        b.yMax = 123;
        return true;
    }

    // Postfix：在原方法执行后调用
    // __instance 获取方法所属的实例对象
    static void Postfix(Bounds __instance)
    {
        __instance.xMax = 20071003;
    }
}
```

### 应用与取消 Patch

调用 `Harmony.CreateAndPatchAll()` 应用 Patch，返回的 `Harmony` 实例可用于后续取消：

```csharp
using HarmonyLib;

public override void Initialize()
{
    // 应用指定类中声明的所有 Patch
    var harmony = Harmony.CreateAndPatchAll(typeof(Patch_MyBoundsLoad));
}

// 需要取消时，撤销该实例注册的全部 Patch
void OnUnload()
{
    harmony.UnpatchSelf();
}
```

### 多个 Patch 的优先级

若多个模组对同一函数注册了 Patch，通过 `[HarmonyPriority]` 控制执行顺序。优先级值**越大越先执行**：

```csharp
[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(400)]  // 先执行
private static class Patch_HighPriority { /* ... */ }

[HarmonyPatch(typeof(Bounds), nameof(Bounds.load))]
[HarmonyPriority(100)]  // 后执行
private static class Patch_LowPriority { /* ... */ }
```

### 与事件式 Hook 混用

HarmonyX Patch 可与基于事件订阅的 DCCM Hook 在同一函数上共存。以下代码来自 `HarmonyXTest.Test_2`：

```csharp
// 应用 HarmonyX Patch（优先级 400）
var inst1 = Harmony.CreateAndPatchAll(typeof(Patch_HighPriority));

// 应用另一个 HarmonyX Patch（优先级 100）
var inst2 = Harmony.CreateAndPatchAll(typeof(Patch_LowPriority));

// 同时注册 DCCM 事件式 Hook
Hook_Bounds.load += (orig, self, b) =>
{
    b.xMin = 666;
    orig(self, b);  // 继续调用 Hook 链中的下一个处理器
};
```

三者按照各自优先级在**同一条 Hook 链**上依次执行，互不干扰。取消时分别调用 `-=` 取消事件、`UnpatchSelf()` 取消 Patch。

## 与 CreateHook 的对比

| 特性 | HarmonyX | CreateHook |
|------|----------|------------|
| 声明方式 | `[HarmonyPatch]` 属性标注静态类 | 显式调用 `HashlinkHooks.Instance.CreateHook()` |
| 方法签名 | 通过参数注入自动匹配（`__instance`、参数按位置传入） | 需手动声明委托类型，`orig` 必须作为第一个参数 |
| 拦截模式 | Prefix / Postfix / Finalizer，各阶段分离 | 单一处理器，通过是否调用 `orig` 决定前置/后置/替换 |
| 优先级机制 | `[HarmonyPriority]` 属性，值越大越先执行 | 受 Hook 链上注册顺序影响 |
| 启用/取消 | `CreateAndPatchAll()` / `UnpatchSelf()` | `Enable()` / `Disable()` |
| 适用场景 | 熟悉 Harmony 生态；需要在前置和后置分别编写独立逻辑 | 简单直接的函数拦截；需要在单一方法内精确控制 `orig` 的调用时机和参数 |
