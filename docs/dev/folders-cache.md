---
sidebar_position: 9
---

# Folders & Cache (FolderInfo / CacheFile)

DCCM exposes its directory layout through `ModCore.Storage.FolderInfo` and provides a metadata-validated file cache through `ModCore.Storage.CacheFile`. Mods can use them to locate DCCM/game files and to cache generated data safely.

## FolderInfo

`FolderInfo` represents a directory. DCCM registers its own folders as static properties, and mods can create additional ones.

### Predefined Folders

| Property | Name (for overrides) | Default location | Purpose |
| --- | --- | --- | --- |
| `FolderInfo.CoreRoot` | `CORE_ROOT` | `<game>/coremod` | DCCM root |
| `FolderInfo.GameRoot` | `GAME_ROOT` | `{CORE_ROOT}/../` | Game root |
| `FolderInfo.NativeRoot` | `CORE_NATIVE_ROOT` | `{CORE_ROOT}/core/native` | All native files |
| `FolderInfo.CurrentNativeRoot` | `CORE_CURRENT_NATIVE_ROOT` | `{CORE_NATIVE_ROOT}/{os}-{arch}` | Native files for the current platform (`win-x64`, `linux-x64`, …) |
| `FolderInfo.SaveRoot` | `SAVE_ROOT` | `{GAME_ROOT}/save` | Game saves |
| `FolderInfo.Logs` | `CORE_LOGS` | `{CORE_ROOT}/logs` | Logs (`log_latest.log`) |
| `FolderInfo.Config` | `CORE_CONFIG` | `{CORE_ROOT}/config` | Config files and `modcore.json` |
| `FolderInfo.Cache` | `CORE_CACHE` | `{CORE_ROOT}/cache` | `CacheFile` data and generated caches |
| `FolderInfo.Data` | `CORE_DATA` | `{CORE_ROOT}/data` | Persistent mod data |
| `FolderInfo.Mods` | `CORE_MODS` | `{CORE_ROOT}/mods` | Installed mods |
| `FolderInfo.Plugins` | `CORE_PLUGINS` | `{CORE_ROOT}/plugins` | Core plugins |

### Per-Mod Folders

Each loaded mod gets two folders through its `ModInfo`:

| Property | Location | Use |
| --- | --- | --- |
| `Info.ModRoot` | The mod's own directory (may be under `coremod/mods` or a Workshop path) | Read files shipped with the mod |
| `Info.ModDataRoot` | `{CORE_ROOT}/data/<ModName>/` | Persistent mod-specific data |

```csharp
// A file shipped with the mod
var pak = Info.ModRoot.GetFilePath("res.pak");

// A persistent file owned by the mod
var statePath = Info.ModDataRoot.GetFilePath("state.json");
File.WriteAllText(statePath, "...");
```

`ModDataRoot` is created automatically when its `FullPath`/`Info` is accessed. `ModRoot` may point to a read-only Workshop directory, so do not write there.

### API

| Member | Description |
| --- | --- |
| `Name` | Folder name, normalized to uppercase |
| `FullPath` | Absolute path; creates the directory if it does not exist |
| `Info` | `DirectoryInfo` for the folder (also creates it) |
| `GetFilePath(name)` | Combines `FullPath` with a relative file name (the file need not exist) |
| `FolderInfo.GetFolder(name)` | Gets a registered folder by name (case-insensitive), or `null` |
| `FolderInfo.OnRebuildInfo` | Event raised when a folder path is rebuilt |

`FullPath` is a good way to ensure a folder exists before writing into it.

### Creating a Custom Folder

```csharp
using ModCore.Storage;

var myFolder = new FolderInfo("MY_MOD_FILES", "{CORE_DATA}/MyMod/files");
var path = myFolder.GetFilePath("cache.bin");
```

Path templates support `{NAME}` placeholders. Placeholders are resolved from environment variables first, then from registered `FolderInfo` instances (names are case-insensitive). Registering a folder later also causes other folders that depend on it to rebuild.

### Overriding a Folder

Set an environment variable named `DCCM_OverridePath_<NAME>` to move a folder without changing the game files:

```powershell
# Example: put DCCM's cache on another drive
$env:DCCM_OverridePath_CORE_CACHE = "D:\DCCM\cache"
```

`<NAME>` is the uppercase `Name` value (for example `CORE_CACHE`, `CORE_DATA`, `CORE_MODS`). The override is applied when the folder is created.

## CacheFile

`CacheFile` is a small helper for disposable generated files. It stores data under `FolderInfo.Cache`:

- data file: `{CORE_CACHE}/<name>`
- metadata file: `{CORE_CACHE}/<name>.cache`

A cache is valid only when the data file matches its stored SHA-384 hash **and** every metadata entry declared by the caller matches the stored value. If either changes, the cache is invalid.

### Usage

```csharp
using ModCore.Storage;

var cache = new CacheFile("MyMod.generated.bin");

// Declare the inputs that the cache depends on.
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
    // Use bytes
}
```

Key points:

- Call `UpdateMetadata` on every run **before** checking `IsValid`. Passing a new key or a different value invalidates the cache.
- `UpdateCache(data)` writes the data file and marks the cache valid.
- `UpdateCache()` recomputes the hash of an existing data file and marks it valid. Use this after writing the file yourself (for example via `CachePath`).
- `TryGetCache` returns `false` when the cache is invalid or missing.
- Metadata values larger than 384 bytes are hashed automatically. Prefer small values such as version strings or hashes.
- Set `DCCM_DEBUG_OPTION_CACHE_INVALID=true` to force all caches to be treated as invalid, which is useful for testing regeneration.

### API

| Member | Description |
| --- | --- |
| `new CacheFile(name)` | Opens/creates a cache entry under `FolderInfo.Cache` |
| `CachePath` | Path to the data file |
| `IsValid` | Whether the current cache is valid |
| `UpdateMetadata(name, string)` | Declares a string metadata input |
| `UpdateMetadata(name, ReadOnlySpan<byte>)` | Declares a binary metadata input |
| `UpdateCache(ReadOnlySpan<byte> data)` | Writes data and marks the cache valid |
| `UpdateCache()` | Recomputes the data file hash and marks the cache valid |
| `TryGetCache(out ReadOnlySpan<byte> data)` | Reads the data if the cache is valid |

### When to Use It

- Good for expensive data that depends on a small set of inputs: DCCM version, game version, a config string, or a hash.
- Not a replacement for `Config<T>` (user settings) or `SaveData<T>` (save-bound data). See [Configuration & Save Data](./config-save.md).
- Cache entries are disposable and safe to delete; DCCM regenerates its own caches. Cache files live in `FolderInfo.Cache` (default `coremod/cache`), alongside files such as `GameProxy.dll`, `hlboot.dat`, and their `.cache` metadata files.

## FAQ

**The cache is always invalid?**

Make sure `UpdateMetadata` is called with exactly the same values on every run. Any new key or changed value invalidates the cache. Also ensure the data file is written with `UpdateCache(data)` or marked with `UpdateCache()` after an external write.

**I wrote the cache file myself; how do I mark it valid?**

Write to `cache.CachePath`, then call `cache.UpdateCache()`.

**Where are my mod's persistent files?**

Use `Info.ModDataRoot` (defaults to `coremod/data/<ModName>/`). Use `CacheFile` only for disposable generated data.

**Does `FullPath` create directories?**

Yes. Accessing `FullPath` or `Info` creates the directory if it does not exist.
