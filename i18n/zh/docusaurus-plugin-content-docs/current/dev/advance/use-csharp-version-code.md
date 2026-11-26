---
sidebar_position: 1
---

# 使用 UseCSharpVersionCode 调试游戏代码

:::info

本页面向开发与调试。`UseCSharpVersionCode` 会让游戏执行由 Hashlink 字节码生成的 C# 伪代码，而不是原生字节码，从而可以在 Visual Studio 等 .NET 调试器中设置断点，单步查看游戏内部逻辑。

:::

:::warning

`UseCSharpVersionCode` 目前仍处于实验阶段，尚未稳定。生成的伪代码不保证与 Hashlink 原生语义完全一致，部分 function 在被烘焙后可能出现行为异常、结果错误甚至崩溃。请仅在开发环境中用于调试与代码查看。

:::

## 工作原理

《Dead Cells》的游戏逻辑运行在 Hashlink 虚拟机上。DCCM 生成的 `GameProxy` 程序集包含游戏 Haxe 类的 C# 代理类型。默认情况下，这些代理方法只是把调用转发给原生 Hashlink 函数，因此在正常游戏过程中，即使在这些方法里下断点也不会命中。

开启 `GeneratePseudocodeAssembly` 后，DCCM 还会把游戏的 Hashlink 字节码编译为 C# 方法体。`HaxeProxyUtils.UseCSharpVersionCode` 会把目标 Hashlink 函数的入口切换到这个 C# 实现。此后游戏执行的就是该函数的 C# 版本，调试器可以停在函数内部，查看调用流程、参数、局部变量等。

这只是一个调试辅助功能，请放在开发构建中，而不是普通游戏模组里。

## 先决条件

1. 在 `coremod/config/modcore.json` 中开启伪代码生成：

   ```json
   {
     "GeneratePseudocodeAssembly": true
   }
   ```

   修改后重启游戏。DCCM 会重新生成带伪代码的 `GameProxy`，首次启动可能较慢。开启伪代码后，DCCM 还会把一份副本写入 `coremod/cache/GamePseudocode.dll` 供反编译器/查看器使用，而运行时仍然加载 `coremod/cache/GameProxy.dll`。

2. 调用 `UseCSharpVersionCode` 时保持该选项开启。如果加载的代理程序集不是用伪代码生成的，API 会抛出 `InvalidOperationException`。

3. 引用 `HaxeProxy`（MDK 默认已包含），并导入 `HaxeProxy.Runtime`。

## API

```csharp
public static void UseCSharpVersionCode(Type t, bool throwOnCompiled)
public static void UseCSharpVersionCode(MethodInfo m, bool throwOnCompiled)
```

| 重载 | 行为 |
| --- | --- |
| `Type` | 遍历该类型的所有 public 实例/静态方法。只烘焙带 `[HashlinkFIndex]` 的方法；带 `[HashlinkShouldUseNative]` 的方法会被跳过。 |
| `MethodInfo` | 烘焙单个方法。如果方法带 `[HashlinkShouldUseNative]` 或没有 `[HashlinkFIndex]`，则抛出异常。 |

`throwOnCompiled` 控制目标 Hashlink 函数已经被编译为 .NET 代码（例如被其他类型或模组烘焙过）时的行为：

- `false` —— 静默跳过。适合批量烘焙的辅助方法。
- `true` —— 抛出 `InvalidOperationException`。

:::warning

- 只有由游戏 Haxe 字节码生成到 `GameProxy` 中的方法才带有 `[HashlinkFIndex]`。对自己的 C# 类型调用该 API 不会产生任何效果。
- `[HashlinkShouldUseNative]` 标记的是反编译 IL 无法忠实还原 Hashlink 原生语义的函数，不要尝试强制烘焙它们。
- 该功能尚未稳定。即使函数没有该标记，也可能与原生实现存在差异；遇到可疑行为时请用反编译器/反汇编器交叉验证。
- 烘焙是进程级的：函数一旦切换到 C# 版本，所有调用方都会使用它，且运行时无法撤销。

:::

## 示例

下面的辅助方法会烘焙一组核心启动/UI 类型。请尽可能早地调用。对于普通模组，`Initialize()` 在 `IOnBeforeGameInit` 期间执行，此时原始 Haxe `Boot.main` 方法体尚未继续，对于后续游戏逻辑而言已经足够早。

```csharp
using System;
using System.Collections.Generic;
using dc;
using HaxeProxy.Runtime;
using ModCore.Mods;

namespace DebugHelper
{
    public class DebugHelperMod(ModInfo info) : ModBase(info)
    {
        public override void Initialize()
        {
            BakeCode();
        }

        private void BakeCode()
        {
            List<Type> types =
            [
                typeof(_Boot),
                typeof(Boot),
                typeof(dc.hxd.App),
                typeof(dc.hxd._App),
                typeof(Main),
                typeof(_Main),
                typeof(dc.pr.TitleScreen),
                typeof(dc.pr._TitleScreen),
                typeof(dc.hxd.fmt.pak.FileSystem),
                typeof(dc.hxd.fmt.pak._FileSystem),
                typeof(_Sys),
                typeof(Sys),
                typeof(dc.libs.Process),
                typeof(dc.libs._Process),
                // typeof(Assets),
                // typeof(_Assets),
                typeof(AssetsLibManager),
                typeof(_AssetsLibManager),
                typeof(dc.haxe.MainLoop),
                typeof(dc.haxe._MainLoop),
                typeof(dc.hxd._System),
                typeof(dc.hxd.System),
                typeof(dc.haxe._EntryPoint),
                typeof(dc.hxd.res.BitmapFont),
                typeof(dc.hxd.res._BitmapFont),
            ];

            Logger.Information("Cook types: {num}", types.Count);

            foreach (var t in types)
            {
                HaxeProxyUtils.UseCSharpVersionCode(t, false);
            }

            Logger.Information("Cook types finished!");
        }
    }
}
```

:::tip

上面的名称都是游戏代理类型，请补充对应的 `using` 指令，或使用完全限定名。例如 `App` 是 `dc.hxd.App`，而不是 `dc.App`。

:::

## 调试技巧

- 将调试器附加到游戏进程。
- 在 Visual Studio 中关闭 **仅我的代码**（`工具 > 选项 > 调试 > 常规`），以便单步进入生成的/框架代码。
- 可以开启 **模块加载时禁用 JIT 优化**，以获得更准确的单步体验。
- 在已烘焙的 `GameProxy` 方法中设置断点。下断点前可先反编译 `coremod/cache/GamePseudocode.dll` 了解代码（参见[查看游戏代码](../look-persudo.md)）。
- 如果某个方法因 `[HashlinkShouldUseNative]` 被跳过，它的原生实现仍会运行；可以使用 crashlink/hlbc 查看。

## 常见问题

**`UseCSharpVersionCode` 抛出 `InvalidOperationException`？**

请在 `coremod/config/modcore.json` 中开启 `GeneratePseudocodeAssembly` 并重启。该 API 需要带伪代码的代理程序集。

**断点仍然没有命中？**

确认该类型/方法确实被烘焙。类型级烘焙只覆盖带 `[HashlinkFIndex]` 的 public 方法；构造函数和 `[HashlinkShouldUseNative]` 方法不会被包含。另外确认该函数是否已在别处被编译——`throwOnCompiled: false` 会静默跳过。

**会影响玩家吗？**

会，这是进程级修改。请只在调试/开发时使用；除非你明确希望游戏运行 C# 版本，否则不要留在正式发布的构建中。
