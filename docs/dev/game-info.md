---
sidebar_position: 10
---

# GameInfo

`ModCore.GameInfo` is a public static class that exposes information about the running game and the installed DCCM. It is populated during startup and is available to mods once the Hashlink VM is ready.

## Members

| Member | Type | Description |
| --- | --- | --- |
| `GameInfo.GameVersion` | `int` | Game version number (`Main.GAME_VERSION`), for example `35` |
| `GameInfo.BuildDate` | `string` | Game build date (`GitVersion.BUILD_DT`) |
| `GameInfo.DCCMVersion` | `Version` | Version of the loaded DCCM assembly |
| `GameInfo.Platform` | `PlatformKind` | Store/platform the game is running on |
| `GameInfo.HlbootHash` | `byte[]` | SHA-384 hash of the loaded `hlboot.dat`, set when the bytecode is loaded |

## PlatformKind

```csharp
public enum PlatformKind
{
    Unknown,
    Steam,
    GOG
}
```

Use it to gate store-specific behavior:

```csharp
using ModCore;

if (GameInfo.Platform == GameInfo.PlatformKind.Steam)
{
    // Steam-only logic
}
```

:::info

`GamePlatformIdentity` is an obsolete enum with the same values. Use `GameInfo.PlatformKind` instead.

:::

## Examples

```csharp
using ModCore;

Logger.Information(
    "Game v{game}, built {date}, DCCM {dccm}",
    GameInfo.GameVersion,
    GameInfo.BuildDate,
    GameInfo.DCCMVersion);

if (GameInfo.Platform == GameInfo.PlatformKind.GOG)
{
    Logger.Information("Running on GOG");
}
```

`GameInfo.DCCMVersion.Major` is the game version that the DCCM build targets. DCCM itself warns at startup when it differs from `GameInfo.GameVersion`:

```csharp
if (GameInfo.GameVersion != GameInfo.DCCMVersion.Major)
{
    Logger.Warning("DCCM targets game v{Target}, but the game is v{Current}",
        GameInfo.DCCMVersion.Major, GameInfo.GameVersion);
}
```

## HlbootHash

`GameInfo.HlbootHash` is set from the game's `hlboot.dat` content and can be used as a cache key when generated data depends on the exact game bytecode:

```csharp
var cache = new CacheFile("MyMod.from-game.bin");
cache.UpdateMetadata("hlboot", GameInfo.HlbootHash);
```

See [Folders & Cache](./folders-cache.md) for `CacheFile`.

## Notes

- `GameVersion` and `BuildDate` read values from the game's Haxe static classes, so they require the Hashlink proxy/VM to be initialized. In a mod these values are available by the time `Initialize()` runs.
- `HlbootHash` is empty until the game bytecode has been loaded.
- `GameInfo` is in the `ModCore` namespace (not `ModCore.Utilities`).
