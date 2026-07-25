---
sidebar_position: 4
---

# Event System

DCCM implements a broadcast event mechanism via `EventSystem`. Each event is defined as a C# interface, marked with the `[Event]` attribute. After a mod implements an event interface, the framework automatically calls the implemented member methods at the corresponding timing — no manual registration required.

## Event Attribute

The `[Event]` attribute supports one optional parameter:

| Parameter | Meaning |
| --- | --- |
| `once: true` (equivalent to `[Event(true)]`) | One-shot event, triggered only once (lifecycle events) |
| `once: false` (equivalent to `[Event(false)]`) | Repeatable (e.g., per-frame events, native resolution events) |
| Default (`[Event]`) | Repeatable |

## All Lifecycle Events

### Framework Initialization

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnCoreModuleInitializing` | During Core initialization, before loading preload modules | `IOnCoreModuleInitializing.cs` |
| `IOnAdvancedModuleInitializing` | Advanced module initialization phase | `IOnAdvancedModuleInitializing.cs` |
| `IOnPluginInitializing` | When a single plugin begins initialization | `IOnPluginInitializing.cs` |
| `IOnPluginInitialized` | When all plugins finish initialization | `IOnPluginInitialized.cs` |
| `IOnSaveConfig` | When config file is saved | `IOnSaveConfig.cs` |

### Resource Loading

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnAfterLoadingAssets` | After game assets are loaded; custom res.pak can be loaded here | `IOnAfterLoadingAssets.cs` |
| `IOnAfterLoadingCDB` | After CDB data is loaded; parameter is `_Data_ cdb` | `Game/IOnAfterLoadingCDB.cs` |
| `IOnLoadingLanguage` | When loading language; parameter is language code `string lang` | `Game/IOnLoadingLanguage.cs` |

### Game Lifecycle

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnBeforeGameInit` | When Haxe main entry executes (before window creation) | `Game/IOnBeforeGameInit.cs` |
| `IOnGameInit` | When window is created, game begins initialization | `Game/IOnGameInit.cs` |
| `IOnGameEndInit` | When game initialization completes | `Game/IOnGameEndInit.cs` |
| `IOnGameExit` | When game is about to exit | `Game/IOnGameExit.cs` |
| `IOnFrameUpdate` | Triggered every frame; parameter is `double dt` (frame delta time) | `Game/IOnFrameUpdate.cs` |

**Difference between `IOnBeforeGameInit` and `IOnGameInit`**: The former triggers when the Haxe main entry executes (earlier), while the latter triggers after window creation. Most scenarios only need `IOnGameInit`.

### Hero

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnHeroInit` | When hero object is initialized | `Game/Hero/IOnHeroInit.cs` |
| `IOnHeroUpdate` | When hero exists, triggered every frame. Parameter `double dt` | `Game/Hero/IOnHeroUpdate.cs` |
| `IOnHeroDispose` | When hero object is destroyed | `Game/Hero/IOnHeroDispose.cs` |

### Save

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnBeforeLoadingSave` | Before loading a save | `Game/Save/IOnBeforeLoadingSave.cs` |
| `IOnAfterLoadingSave` | After save is loaded; parameter `User data` | `Game/Save/IOnAfterLoadingSave.cs` |
| `IOnAfterLoadingModdedSave` | After mod save data is loaded; parameter `Func<string, JObject?> getData` | `Game/Save/IOnAfterLoadingModdedSave.cs` |
| `IOnBeforeSavingSave` | Before saving a save; parameter `EventData(User Data, bool OnlyGameData)` | `Game/Save/IOnBeforeSavingSave.cs` |
| `IOnBeforeSavingModdedSave` | Before saving mod save data; parameter `Action<string, JObject> setData` | `Game/Save/IOnBeforeSavingModdedSave.cs` |
| `IOnAfterSavingSave` | After save is saved | `Game/Save/IOnAfterSavingSave.cs` |
| `IOnCopySave` | When save is copied/moved; parameter `EventData(int SlotFrom, int SlotTo)` | `Game/Save/IOnCopySave.cs` |
| `IOnDeleteSave` | When deleting a save; parameter `int? slot` | `Game/Save/IOnDeleteSave.cs` |

### Menu

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnAfterPauseMenuBuild` | After pause menu is built; parameter `Pause pause` | `Game/Menu/IOnAfterPauseMenuBuild.cs` |

### VM

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnHashlinkVMReady` | When Hashlink VM is ready | `VM/IOnHashlinkVMReady.cs` |
| `IOnResolveNativeLib` | When resolving native libraries; returns `EventResult<nint>` | `VM/IOnResolveNativeLib.cs` |
| `IOnResolveNativeFunction` | When resolving native functions; returns `EventResult<nint>` | `VM/IOnResolveNativeFunction.cs` |

### Mod Discovery

| Interface Name | Trigger Timing | Source File |
| --- | --- | --- |
| `IOnFindingMods` | When scanning mod directories; parameter `Action<string> findMod` | `Mods/IOnFindingMods.cs` |
| `IOnRegisterModsType` | When registering mod types; parameter `AddModType add` | `Mods/IOnRegisterModsType.cs` |
| `IOnCollectedModInfo` | When the loader processes each mod info; parameter `ModInfo info` | `Mods/IOnCollectedModInfo.cs` |

## Code Example

The following excerpt from the SampleSimple mod demonstrates how to implement multiple event interfaces:

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
    // Implements multiple event interfaces simultaneously
    public class SimpleMod(ModInfo info) : ModBase(info),
        IOnGameExit,           // Game exit
        IOnGameEndInit,        // Game initialization complete
        IOnAfterLoadingAssets  // Asset loading complete
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }

        // Load custom resource pack
        void IOnAfterLoadingAssets.OnAfterLoadingAssets()
        {
            var res = Info.ModRoot!.GetFilePath("res.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
        }

        // Load custom text resource after game initialization
        void IOnGameEndInit.OnGameEndInit()
        {
            var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());
            Logger.Information("The content of test1.txt is {text}", test1.toText());
        }

        // Log when game exits
        void IOnGameExit.OnGameExit()
        {
            Logger.Information("Game is exit");
        }
    }
}
```

Key points:

1. The mod class inherits from `ModBase` and implements the required event interfaces.
2. Event methods use **explicit interface implementation** (`IOnGameExit.OnGameExit()`), which is the required pattern by the framework.
3. The framework automatically calls these methods at the corresponding timing; no manual registration or subscription is needed.
4. Event interface namespaces are under `ModCore.Events.Interfaces` and its sub-namespaces.
