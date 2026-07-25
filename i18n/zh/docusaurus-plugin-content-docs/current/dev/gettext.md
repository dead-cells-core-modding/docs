---
sidebar_position: 7
---

# 本地化（GetText）

`GetText` 模块负责加载模组的 `.mo` 翻译文件，将其合并到游戏内建的 `Lang.Class.t.get()` 本地化链中，使模组的 UI 文本、提示等支持多语言切换。

## 概述

游戏使用 GNU gettext 的 `.mo` 二进制格式存储翻译文本。DCCM 拦截游戏加载 `.mo` 的流程，在每次语言切换时将已注册模组的翻译文件按优先级顺序载入，合并到游戏全局的 `texts` 表中。开发者只需调用 `GetText.Instance.GetString("key")` 即可获取当前语言的翻译字符串。

:::info
`GetText` 是 CoreModule，优先级为 `ModulePriorities.Game`，在 `IOnAdvancedModuleInitializing` 阶段完成自身初始化并注册 `dccm-core` 命名空间。
:::

## `.mo` 文件查找路径

模组打包 `.mo` 文件时，命名需遵循固定模式。调用 `RegisterMod(name)` 后，DCCM 按以下顺序依次尝试加载：

| 优先级 | 路径模式 | 说明 |
| --- | --- | --- |
| 1 | `{name}/lang/main.{lang}.mo` | 带命名空间，文件名为 `main`，当前语言 |
| 2 | `{name}/lang/{name}.{lang}.mo` | 带命名空间，文件名与命名空间同名，当前语言 |
| 3 | `lang/{name}.{lang}.mo` | 无命名空间，当前语言 |
| 4 | `{name}/lang/main.en.mo` | 带命名空间，文件名为 `main`，英语（回退） |
| 5 | `{name}/lang/{name}.en.mo` | 带命名空间，文件名与命名空间同名，英语（回退） |
| 6 | `lang/{name}.en.mo` | 无命名空间，英语（回退） |

其中 `{lang}` 为当前游戏语言代码（如 `zh`、`en`、`fr`），`{name}` 为 `RegisterMod()` 传入的命名空间。

6 种路径逐一尝试，找到第一个存在的文件即停止。前 3 种匹配当前语言，后 3 种回退到英语。这意味着：如果你的模组只提供英语翻译，将文件放在满足路径 4~6 之一的位置即可；如果提供了多语言，DCCM 会优先加载当前语言的文件。

## 注册与加载

在模组初始化时调用 `RegisterMod()` 注册命名空间。DCCM 在每次语言切换（`readMo` 被调用）时自动遍历所有已注册名称并加载对应 `.mo` 文件：

```csharp
using ModCore.Events.Interfaces;
using ModCore.Modules;
using ModCore.Utilities;

internal class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingAssets
{
    public override void Initialize()
    {
        // 注册命名空间，DCCM 会在语言切换时自动加载对应 .mo 文件
        GetText.Instance.RegisterMod("DeadCellsMultiplayerX");
    }

    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        var res = Info.ModRoot!.GetFilePath("res.pak");
        FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
    }
}
```

完成加载后，DCCM 广播 `IOnLoadingLanguage` 事件，参数为语言代码字符串。模组可以监听此事件在语言切换后执行自定义逻辑（如刷新 UI）：

```csharp
public class MyMod(ModInfo info) : ModBase(info), IOnLoadingLanguage
{
    void IOnLoadingLanguage.OnLoadingLanguage(string lang)
    {
        // lang 为 "zh"、"en" 等语言代码
        Logger.Information("语言已切换为: {lang}", lang);
    }
}
```

## 获取字符串

模块提供两个重载：

```csharp
// 简单翻译
string GetString(string key)

// 带参数格式化的翻译
string GetString(string key, IDictionary<string, string> @params)
```

项目中常封装一个简短的辅助方法以简化调用：

```csharp
private static string T(string key) => GetText.Instance.GetString(key);
```

使用示例（摘自 DeadCellsMultiplayerX 的 `ClientMain.cs`）：

```csharp
using ModCore.Modules;

// 封装辅助方法
private static string T(string key) => GetText.Instance.GetString(key);

// 在 UI 构建中使用
lobby!.BuildMenuChild("Online", () => lobby.Show(), color: color);

// 带参数的情况：
var text = GetText.Instance.GetString("welcome_message",
    new Dictionary<string, string>
    {
        { "player", playerName },
        { "version", "1.0" }
    });
```

## 完整流程

总结从制作翻译到运行时使用的完整步骤：

1. **创建 `.mo` 文件**：使用 `msgfmt` 或 PO 编辑工具（如 Poedit）将 `.po` 编译为 `.mo`，按路径规范命名（如 `DeadCellsMultiplayerX/lang/main.zh.mo`）
2. **打包**：在 `.csproj` 中通过 `PackAssets` 将 `.mo` 文件纳入 `res.pak`。参见[资源打包](../resources#打包资源)
3. **加载**：实现 `IOnAfterLoadingAssets`，调用 `FsPak.Instance.FileSystem.loadPak()` 加载 `res.pak`
4. **注册**：在 `Initialize()` 中调用 `GetText.Instance.RegisterMod("命名空间")`
5. **使用**：通过 `GetText.Instance.GetString("key")` 获取翻译字符串

## API 速查

| API | 说明 |
| --- | --- |
| `GetText.Instance.RegisterMod(name)` | 注册模组命名空间，DCCM 在语言切换时自动加载该命名空间下的 `.mo` 文件 |
| `GetText.Instance.GetString(key)` | 获取当前语言的翻译字符串（key 不存在时返回 key 本身） |
| `GetText.Instance.GetString(key, params)` | 获取翻译字符串并替换占位符，`params` 为 `IDictionary<string, string>` |
| `IOnLoadingLanguage.OnLoadingLanguage(lang)` | 语言加载/切换事件，参数 `lang` 为语言代码 |
