---
sidebar_position: 4
---

# 事件系统

DCCM 通过 `EventSystem` 实现广播式事件机制。每个事件定义为一个 C# 接口，用 `[Event]` 属性标记。模组实现事件接口后，框架会在对应时机自动调用实现的成员方法，无需手动注册。

## Event 属性

`[Event]` 属性支持一个可选参数：

| 参数 | 含义 |
| --- | --- |
| `once: true`（等价 `[Event(true)]`） | 一次性事件，只触发一次（生命周期事件） |
| `once: false`（等价 `[Event(false)]`） | 可重复触发（如每帧事件、Native 解析事件） |
| 默认（`[Event]`） | 可重复触发 |

## 全部生命周期事件

### 框架初始化

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnCoreModuleInitializing` | Core 初始化，加载预加载模块前 | `IOnCoreModuleInitializing.cs` |
| `IOnAdvancedModuleInitializing` | 高级模块初始化阶段 | `IOnAdvancedModuleInitializing.cs` |
| `IOnPluginInitializing` | 单个插件开始初始化时 | `IOnPluginInitializing.cs` |
| `IOnPluginInitialized` | 所有插件初始化完成时 | `IOnPluginInitialized.cs` |
| `IOnSaveConfig` | 配置文件保存时 | `IOnSaveConfig.cs` |

### 资源加载

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnAfterLoadingAssets` | 游戏资源加载完成后，可在此加载自定义 res.pak | `IOnAfterLoadingAssets.cs` |
| `IOnAfterLoadingCDB` | CDB 数据加载完成后，参数为 `_Data_ cdb` | `Game/IOnAfterLoadingCDB.cs` |
| `IOnLoadingLanguage` | 加载语言时，参数为语言代码 `string lang` | `Game/IOnLoadingLanguage.cs` |

### 游戏生命周期

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnBeforeGameInit` | Haxe 主入口执行时（窗口创建前） | `Game/IOnBeforeGameInit.cs` |
| `IOnGameInit` | 窗口创建时，游戏开始初始化 | `Game/IOnGameInit.cs` |
| `IOnGameEndInit` | 游戏初始化完成时 | `Game/IOnGameEndInit.cs` |
| `IOnGameExit` | 游戏即将退出时 | `Game/IOnGameExit.cs` |
| `IOnFrameUpdate` | 每帧触发，参数为 `double dt`（帧间隔时间） | `Game/IOnFrameUpdate.cs` |

**`IOnBeforeGameInit` 与 `IOnGameInit` 的区别**：前者在 Haxe 主入口执行时触发（更早），后者在窗口创建后触发。大多数场景使用 `IOnGameInit` 即可。

### 英雄（Hero）

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnHeroInit` | 英雄对象初始化时 | `Game/Hero/IOnHeroInit.cs` |
| `IOnHeroUpdate` | 英雄存在时，每帧触发。参数 `double dt` | `Game/Hero/IOnHeroUpdate.cs` |
| `IOnHeroDispose` | 英雄对象被销毁时 | `Game/Hero/IOnHeroDispose.cs` |

### 存档（Save）

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnBeforeLoadingSave` | 加载存档前 | `Game/Save/IOnBeforeLoadingSave.cs` |
| `IOnAfterLoadingSave` | 存档加载完成后，参数 `User data` | `Game/Save/IOnAfterLoadingSave.cs` |
| `IOnAfterLoadingModdedSave` | 模组存档数据加载完成后，参数 `Func<string, JObject?> getData` | `Game/Save/IOnAfterLoadingModdedSave.cs` |
| `IOnBeforeSavingSave` | 保存存档前，参数 `EventData(User Data, bool OnlyGameData)` | `Game/Save/IOnBeforeSavingSave.cs` |
| `IOnBeforeSavingModdedSave` | 保存模组存档数据前，参数 `Action<string, JObject> setData` | `Game/Save/IOnBeforeSavingModdedSave.cs` |
| `IOnAfterSavingSave` | 存档保存完成后 | `Game/Save/IOnAfterSavingSave.cs` |
| `IOnCopySave` | 存档复制/移动时，参数 `EventData(int SlotFrom, int SlotTo)` | `Game/Save/IOnCopySave.cs` |
| `IOnDeleteSave` | 删除存档时，参数 `int? slot` | `Game/Save/IOnDeleteSave.cs` |

### 菜单（Menu）

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnAfterPauseMenuBuild` | 暂停菜单构建完成后，参数 `Pause pause` | `Game/Menu/IOnAfterPauseMenuBuild.cs` |

### 虚拟机（VM）

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnHashlinkVMReady` | Hashlink 虚拟机就绪时 | `VM/IOnHashlinkVMReady.cs` |
| `IOnResolveNativeLib` | 解析原生库时，返回 `EventResult<nint>` | `VM/IOnResolveNativeLib.cs` |
| `IOnResolveNativeFunction` | 解析原生函数时，返回 `EventResult<nint>` | `VM/IOnResolveNativeFunction.cs` |

### 模组发现

| 接口名 | 触发时机 | 所在文件 |
| --- | --- | --- |
| `IOnFindingMods` | 扫描模组目录时，参数 `Action<string> findMod` | `Mods/IOnFindingMods.cs` |
| `IOnRegisterModsType` | 注册模组类型时，参数 `AddModType add` | `Mods/IOnRegisterModsType.cs` |
| `IOnCollectedModInfo` | 加载器处理每个模组信息时，参数 `ModInfo info` | `Mods/IOnCollectedModInfo.cs` |

## 代码示例

以下摘自 SampleSimple 模组，展示如何实现多个事件接口：

```csharp
using dc;
using dc.hxd;
using ModCore.Events.Interfaces;
using ModCore.Events.Interfaces.Game;
using ModCore.Mods;
using ModCore.Modules;
using ModCore.Utilities;

namespace SampleSimple
{
    // 同时实现多个事件接口
    public class SimpleMod(ModInfo info) : ModBase(info),
        IOnGameExit,           // 游戏退出
        IOnGameEndInit,        // 游戏初始化完成
        IOnAfterLoadingAssets  // 资源加载完成
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }

        // 加载自定义资源包
        void IOnAfterLoadingAssets.OnAfterLoadingAssets()
        {
            var res = Info.ModRoot!.GetFilePath("res.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
        }

        // 游戏初始化完成后加载自定义文本资源
        void IOnGameEndInit.OnGameEndInit()
        {
            var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());
            Logger.Information("The content of test1.txt is {text}", test1.toText());
        }

        // 游戏退出时记录日志
        void IOnGameExit.OnGameExit()
        {
            Logger.Information("Game is exit");
        }
    }
}
```

要点：

1. 模组类继承 `ModBase`，同时实现所需的事件接口。
2. 事件方法使用**显式接口实现**（`IOnGameExit.OnGameExit()`），这是框架要求的写法。
3. 框架在对应时机自动调用这些方法，无需手动注册或订阅。
4. 事件接口的命名空间在 `ModCore.Events.Interfaces` 及其子命名空间下。
