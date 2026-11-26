---
sidebar_position: 9
---

# 目录与缓存（FolderInfo / CacheFile）

DCCM 通过 `ModCore.Storage.FolderInfo` 暴露目录结构，并提供基于元数据校验的 `ModCore.Storage.CacheFile` 文件缓存。模组可以用它们定位 DCCM/游戏文件，并安全地缓存生成数据。

## FolderInfo

`FolderInfo` 表示一个目录。DCCM 已将自己的目录注册为静态属性，模组也可以创建额外目录。

### 预定义目录

| 属性 | 名称（用于覆盖） | 默认位置 | 用途 |
| --- | --- | --- | --- |
| `FolderInfo.CoreRoot` | `CORE_ROOT` | `<游戏目录>/coremod` | DCCM 根目录 |
| `FolderInfo.GameRoot` | `GAME_ROOT` | `{CORE_ROOT}/../` | 游戏根目录 |
| `FolderInfo.NativeRoot` | `CORE_NATIVE_ROOT` | `{CORE_ROOT}/core/native` | 所有原生文件 |
| `FolderInfo.CurrentNativeRoot` | `CORE_CURRENT_NATIVE_ROOT` | `{CORE_NATIVE_ROOT}/{系统}-{架构}` | 当前平台的原生文件（`win-x64`、`linux-x64` 等） |
| `FolderInfo.SaveRoot` | `SAVE_ROOT` | `{GAME_ROOT}/save` | 游戏存档 |
| `FolderInfo.Logs` | `CORE_LOGS` | `{CORE_ROOT}/logs` | 日志（`log_latest.log`） |
| `FolderInfo.Config` | `CORE_CONFIG` | `{CORE_ROOT}/config` | 配置文件与 `modcore.json` |
| `FolderInfo.Cache` | `CORE_CACHE` | `{CORE_ROOT}/cache` | `CacheFile` 数据与生成的缓存 |
| `FolderInfo.Data` | `CORE_DATA` | `{CORE_ROOT}/data` | 模组持久化数据 |
| `FolderInfo.Mods` | `CORE_MODS` | `{CORE_ROOT}/mods` | 已安装模组 |
| `FolderInfo.Plugins` | `CORE_PLUGINS` | `{CORE_ROOT}/plugins` | 核心插件 |

### 模组专属目录

每个已加载模组都可以通过 `ModInfo` 获得两个目录：

| 属性 | 位置 | 用途 |
| --- | --- | --- |
| `Info.ModRoot` | 模组自身目录（可能位于 `coremod/mods` 或创意工坊路径） | 读取模组自带的文件 |
| `Info.ModDataRoot` | `{CORE_ROOT}/data/<模组名>/` | 模组专属的持久化数据 |

```csharp
// 模组自带的文件
var pak = Info.ModRoot.GetFilePath("res.pak");

// 模组拥有的持久化文件
var statePath = Info.ModDataRoot.GetFilePath("state.json");
File.WriteAllText(statePath, "...");
```

访问 `ModDataRoot` 的 `FullPath`/`Info` 时会自动创建目录。`ModRoot` 可能指向只读的创意工坊目录，不要向其中写入。

### API

| 成员 | 说明 |
| --- | --- |
| `Name` | 目录名，统一转为大写 |
| `FullPath` | 绝对路径；目录不存在时会自动创建 |
| `Info` | 目录的 `DirectoryInfo`（同样会创建目录） |
| `GetFilePath(name)` | 将 `FullPath` 与相对文件名组合（文件不必存在） |
| `FolderInfo.GetFolder(name)` | 按名称获取已注册目录（不区分大小写），不存在返回 `null` |
| `FolderInfo.OnRebuildInfo` | 目录路径重建时触发的事件 |

写入文件前，使用 `FullPath` 可以确保目录已存在。

### 创建自定义目录

```csharp
using ModCore.Storage;

var myFolder = new FolderInfo("MY_MOD_FILES", "{CORE_DATA}/MyMod/files");
var path = myFolder.GetFilePath("cache.bin");
```

路径模板支持 `{NAME}` 占位符。解析时先查找同名环境变量，再查找已注册的 `FolderInfo`（名称不区分大小写）。稍后注册的目录也会让依赖它的其他目录重新构建。

### 覆盖目录位置

设置名为 `DCCM_OverridePath_<NAME>` 的环境变量，可以不改动游戏文件而移动目录：

```powershell
# 示例：把 DCCM 缓存放到其他盘
$env:DCCM_OverridePath_CORE_CACHE = "D:\DCCM\cache"
```

`<NAME>` 是大写的 `Name` 值（例如 `CORE_CACHE`、`CORE_DATA`、`CORE_MODS`）。覆盖会在目录创建时生效。

## CacheFile

`CacheFile` 是用于可丢弃生成文件的小型辅助类。数据保存在 `FolderInfo.Cache` 下：

- 数据文件：`{CORE_CACHE}/<name>`
- 元数据文件：`{CORE_CACHE}/<name>.cache`

只有当数据文件的 SHA-384 哈希与记录值一致，**并且**调用方声明的每个元数据项都与记录值一致时，缓存才有效。任意一项变化都会使缓存失效。

### 用法

```csharp
using ModCore.Storage;

var cache = new CacheFile("MyMod.generated.bin");

// 声明缓存依赖的输入。
cache.UpdateMetadata("dccm", typeof(CacheFile).Assembly.GetName().Version?.ToString() ?? "unknown");
cache.UpdateMetadata("game", GameInfo.GameVersion.ToString());
cache.UpdateMetadata("options", "fast");

if (!cache.IsValid)
{
    var data = BuildExpensiveData();
    cache.UpdateCache(data);
}

if (cache.TryGetCache(out var bytes))
{
    // 使用 bytes
}
```

要点：

- 每次运行都应在检查 `IsValid` **之前**调用 `UpdateMetadata`。传入新键或不同值会使缓存失效。
- `UpdateCache(data)` 会写入数据文件并将缓存标记为有效。
- `UpdateCache()` 会重新计算现有数据文件的哈希并标记为有效；适用于你自己写入文件之后（例如通过 `CachePath`）。
- 缓存无效或文件缺失时，`TryGetCache` 返回 `false`。
- 超过 384 字节的元数据值会自动哈希。建议只传版本号或哈希等小值。
- 设置 `DCCM_DEBUG_OPTION_CACHE_INVALID=true` 可强制所有缓存视为无效，便于测试重新生成。

### API

| 成员 | 说明 |
| --- | --- |
| `new CacheFile(name)` | 打开/创建 `FolderInfo.Cache` 下的缓存项 |
| `CachePath` | 数据文件路径 |
| `IsValid` | 当前缓存是否有效 |
| `UpdateMetadata(name, string)` | 声明字符串元数据输入 |
| `UpdateMetadata(name, ReadOnlySpan<byte>)` | 声明二进制元数据输入 |
| `UpdateCache(ReadOnlySpan<byte> data)` | 写入数据并标记缓存有效 |
| `UpdateCache()` | 重新计算数据文件哈希并标记缓存有效 |
| `TryGetCache(out ReadOnlySpan<byte> data)` | 缓存有效时读取数据 |

### 适用场景

- 适合依赖少量输入（DCCM 版本、游戏版本、配置字符串或哈希）的昂贵生成数据。
- 不能替代 `Config<T>`（用户设置）或 `SaveData<T>`（存档绑定数据）。参见[配置与存档](./config-save.md)。
- 缓存条目可随时删除，DCCM 会重新生成自己的缓存。缓存文件位于 `FolderInfo.Cache`（默认 `coremod/cache`），与 `GameProxy.dll`、`hlboot.dat` 及其 `.cache` 元数据文件放在一起。

## 常见问题

**缓存总是无效？**

确认每次运行 `UpdateMetadata` 传入的值完全一致。任何新键或变化的值都会使缓存失效。另外确认数据文件是通过 `UpdateCache(data)` 写入的，或在外部写入后调用了 `UpdateCache()`。

**缓存文件是我自己写的，如何标记为有效？**

写入 `cache.CachePath`，然后调用 `cache.UpdateCache()`。

**模组的持久化文件应该放在哪里？**

使用 `Info.ModDataRoot`（默认 `coremod/data/<模组名>/`）。`CacheFile` 只用于可丢弃的生成数据。

**`FullPath` 会创建目录吗？**

会。访问 `FullPath` 或 `Info` 时，如果目录不存在会自动创建。
