---
sidebar_position: 10
---

# GameInfo

`ModCore.GameInfo` 是一个公共静态类，用于获取当前运行的游戏与已安装 DCCM 的信息。它在启动阶段被填充，Hashlink 虚拟机就绪后即可在模组中读取。

## 成员

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `GameInfo.GameVersion` | `int` | 游戏版本号（`Main.GAME_VERSION`），例如 `35` |
| `GameInfo.BuildDate` | `string` | 游戏构建日期（`GitVersion.BUILD_DT`） |
| `GameInfo.DCCMVersion` | `Version` | 当前加载的 DCCM 程序集版本 |
| `GameInfo.Platform` | `PlatformKind` | 游戏运行的商店/平台 |
| `GameInfo.HlbootHash` | `byte[]` | 已加载 `hlboot.dat` 的 SHA-384 哈希，在字节码加载时设置 |

## PlatformKind

```csharp
public enum PlatformKind
{
    Unknown,
    Steam,
    GOG
}
```

可用于区分商店平台：

```csharp
using ModCore;

if (GameInfo.Platform == GameInfo.PlatformKind.Steam)
{
    // Steam 专属逻辑
}
```

:::info

`GamePlatformIdentity` 是值相同的旧枚举，已过时。请使用 `GameInfo.PlatformKind`。

:::

## 示例

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

`GameInfo.DCCMVersion.Major` 是当前 DCCM 构建所针对的游戏版本。两者不一致时，DCCM 会在启动时输出警告：

```csharp
if (GameInfo.GameVersion != GameInfo.DCCMVersion.Major)
{
    Logger.Warning("DCCM targets game v{Target}, but the game is v{Current}",
        GameInfo.DCCMVersion.Major, GameInfo.GameVersion);
}
```

## HlbootHash

`GameInfo.HlbootHash` 来自游戏的 `hlboot.dat` 内容，当生成数据依赖游戏字节码时，可作为缓存键：

```csharp
var cache = new CacheFile("MyMod.from-game.bin");
cache.UpdateMetadata("hlboot", GameInfo.HlbootHash);
```

`CacheFile` 的用法参见[目录与缓存](./folders-cache.md)。

## 注意事项

- `GameVersion` 与 `BuildDate` 读取的是游戏 Haxe 静态类中的值，因此需要 Hashlink 代理/虚拟机已初始化。在模组中，`Initialize()` 执行时这些值已经可用。
- `HlbootHash` 在游戏字节码加载前为空。
- `GameInfo` 位于 `ModCore` 命名空间（不是 `ModCore.Utilities`）。
