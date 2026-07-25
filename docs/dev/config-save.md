---
sidebar_position: 6
---

# Configuration & Save Data

DCCM provides two data persistence mechanisms: `Config<T>` for mod configuration (JSON files) and `SaveData<T>` for save-game-bound mod data. Additionally, the `IHxbitSerializable<TData>` interface allows game objects to participate in Hxbit (the game's native) serialization.

## Config\<T\>

`Config<T>` is an automatic JSON serialization config system. Config files are stored at `{CORE_ROOT}/config/{name}.json`, with read/write operations completely transparent to the caller.

### Declaration

Define a plain C# class as the config model, then create a static Config instance:

```csharp
using ModCore.Storage;

public class MyModConfig
{
    // Static singleton; constructor parameter is the config file name (without extension)
    public static Config<MyModConfig> Instance { get; } = new("MyMod");

    public double StartTimeout { get; set; } = 5;
    public bool EnableFeature { get; set; } = true;
}
```

`T` must have a parameterless constructor (`new()` constraint). Properties support any JSON-serializable type.

### Read & Write

Read and write config via the `Value` property:

```csharp
// Read
double timeout = MyModConfig.Instance.Value.StartTimeout;

// Modify (auto-saved at appropriate time)
MyModConfig.Instance.Value.EnableFeature = false;
```

The first access to `Value` automatically loads from the JSON file; if the file doesn't exist, a default instance is created and saved, ensuring the config file is always present.

### Saving

`Config<T>` implements the `IOnSaveConfig` interface, and DCCM automatically triggers saving at appropriate times (e.g. game exit, mod unload). Manual calls are generally not needed.

To force an immediate save, call `Save()`:

```csharp
MyModConfig.Instance.Save();
```

### Custom Serialization

Modify `SerializerOptions` to adjust JSON output formatting (indentation is enabled by default):

```csharp
MyModConfig.Instance.SerializerOptions.Formatting = Formatting.None;
```

:::tip External config libraries
DCCM's `Config<T>` handles the mod's JSON config files. If you use another library's config system (e.g. `ConfigurationManager`), the two do not conflict; each manages its own files independently.
:::

## SaveData\<T\>

`SaveData<T>` binds data to the game save file. Data is automatically restored when loading a save and automatically written when saving the game. Data is embedded in the save file as JSON.

### SaveData Declaration

Operate via the `Value` property, just like Config:

```csharp
// Modify data
MyMod.Save.Value.Coins += 100;
MyMod.Save.Value.UnlockedItems.Add("SomeWeapon");

// Read data
int coins = MyMod.Save.Value.Coins;
```

Data is automatically written on save and restored on load. Developers only need to work with `Value`; no need to worry about persistence details.

### Differences from Config

| | Config\<T\> | SaveData\<T\> |
| --- | --- | --- |
| Storage location | `config/{name}.json` | Embedded in game save file |
| Lifecycle | Mod level, shared across saves | Save level, independent per save |
| Use case | Mod settings (keybindings, toggles) | Progression data that varies per save |
| T constraint | `new()` | `class, new()` |

## IHxbitSerializable\<TData\>

The game internally uses the Hxbit protocol to serialize runtime objects (weapons, skills, etc.). When a C# class inherits from a native game type and needs to participate in Hxbit serialization, implement this interface.

### Interface Definition

```csharp
namespace ModCore.Storage
{
    public interface IHxbitSerializable<TData>
    {
        TData GetData();       // Called during serialization; returns the data to persist
        void SetData(TData data); // Called during deserialization; restores state from data
    }
}
```

### Example

In the example below, `OtherDashSword` inherits from `DashSword` (a native game weapon class) and implements `IHxbitSerializable<object>` to participate in saves:

```csharp
using dc.en;
using ModCore.Storage;

public class OtherDashSword : DashSword, IHxbitSerializable<object>
{
    public OtherDashSword(Hero hero, InventItem item) : base(hero, item) { }

    object IHxbitSerializable<object>.GetData()
    {
        // Return the state that needs to be persisted
        return new();
    }

    void IHxbitSerializable<object>.SetData(object data)
    {
        // Restore state from the save
    }
}
```

`TData` is typically `object`; the actual type is determined by the game's Hxbit engine. If your type doesn't need extra state persistence, leaving both methods empty is fine, but implementing the interface itself is required; otherwise, the game may throw errors on load due to a missing serialization handler.

## Key API

| API | Description |
| --- | --- |
| `new Config<T>("name")` | Create a config instance, automatically registered with the event system |
| `Config<T>.Value` | Get/set config values (auto-loaded on first access) |
| `Config<T>.Save()` | Manually save config to a JSON file |
| `Config<T>.SerializerOptions` | JSON serialization settings (`Newtonsoft.Json.JsonSerializerSettings`) |
| `new SaveData<T>("name")` | Create a save data instance, automatically registered with the event system |
| `SaveData<T>.Value` | Read/write save data |
| `IHxbitSerializable<TData>.GetData()` | Extract data during Hxbit serialization |
| `IHxbitSerializable<TData>.SetData(data)` | Restore state during Hxbit deserialization |
