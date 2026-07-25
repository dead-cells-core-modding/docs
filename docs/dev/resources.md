---
sidebar_position: 3
---

# Resources & Data

Mods can use DCCM's resource packaging system to bundle custom assets (images, text, data, etc.) into `.pak` files and inject them at game load time.

## Packaging Resources

Configure `PackAssets` and `GenerateSinglePakFile` in the `.csproj` file:

```xml
<PropertyGroup>
    <!-- Merge all PackAssets into a single res.pak -->
    <GenerateSinglePakFile>true</GenerateSinglePakFile>
</PropertyGroup>

<ItemGroup>
    <!--
        Include: source file path (supports wildcards)
        RootInPak: root path of the resource inside the pak, used for later access
    -->
    <PackAssets Include="assets/**/*" RootInPak="my_mod" />
</ItemGroup>
```

After building, the generated `res.pak` is placed in the mod output directory, alongside `modinfo.json`.

## Loading Resources

Implement the `IOnAfterLoadingAssets` interface in your mod and load `res.pak` via `FsPak`:

```csharp
using ModCore.Events.Interfaces;
using ModCore.Modules;
using ModCore.Utilities;

public class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingAssets
{
    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        // Get the full path of res.pak
        var pakPath = Info.ModRoot!.GetFilePath("res.pak");

        // Load the pak into the game filesystem
        FsPak.Instance.FileSystem.loadPak(pakPath.AsHaxeString());
    }
}
```

`IOnAfterLoadingAssets` fires after game assets finish loading, making it the ideal moment to inject custom resources.

## Accessing Resources

Once loaded, access resources via `Res.Class.load()`, using the path format `RootInPak/relativePath`:

```csharp
void IOnGameEndInit.OnGameEndInit()
{
    // Load a text resource (RootInPak is "my_mod", file is assets/data/config.txt)
    var text = Res.Class.load("my_mod/data/config.txt".AsHaxeString());
    Logger.Information("Resource content: {content}", text.toText());
}
```

## Key API — Resources

| API | Description |
| ----- | ------ |
| `GenerateSinglePakFile` | MSBuild property; when `true`, merges `PackAssets` into a single res.pak |
| `PackAssets` | MSBuild item; declares resource files to package along with their `RootInPak` root path |
| `Info.ModRoot.GetFilePath(name)` | Gets the full path of a file under the mod root directory |
| `FsPak.Instance.FileSystem.loadPak(path)` | Loads a `.pak` file into the game resource system |
| `Res.Class.load(path)` | Reads data from a loaded resource by path |
| `.AsHaxeString()` | Converts a C# string to a Haxe string (extension method in `ModCore.Utilities`) |

## FAQ — Resources

**`Res.Class.load()` returns `null`?**

Make sure the `RootInPak` prefix matches the load path. For example, with `RootInPak="my_mod"`, the access path should be `"my_mod/xxx.txt"`.

**Resources not taking effect after loading?**

`IOnAfterLoadingAssets` fires early; some resources can only be accessed after `IOnGameEndInit`. For resources that need to load at a specific time, call `loadPak` in the corresponding lifecycle event.

---

## Data Modification (CastleDB)

CastleDB (CDB) is the game's core data table, defining all content properties: weapons, monsters, skills, and more. DCCM supports two CDB modification approaches: compile-time diff generation and runtime direct modification.

### Compile-Time Diffs

Enable `GenerateDiffCDB` in `.csproj` and specify `GameVersion`:

```xml
<PropertyGroup>
    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
</PropertyGroup>
```

At build time, MDK compares `data.cdb` against the specified version's CDB via DCCMTool, generating only a difference patch. The patch is then merged into `res.pak`. Enabling `GenerateDiffCDB` automatically turns on `GenerateSinglePakFile`.

`GameVersion` locates the version-specific CDB in the MDK template database (`mdk/databases/v{version}/data.cdb`). If the mod directory does not contain a `data.cdb`, CDB operations are done solely through runtime modification.

### Runtime Modification

Implement the `IOnAfterLoadingCDB` interface to directly manipulate the `_Data_` object after CDB loading completes:

```csharp
using dc;
using ModCore.Events.Interfaces.Game;

public class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingCDB
{
    public void OnAfterLoadingCDB(_Data_ cdb)
    {
        // Modify an existing entry: set all weapon prices to 1
        cdb.item.byId.get("DeathMoney".AsHaxeString()).cellCost = 1;

        // Access data in a specific table via byId
        var weapon = cdb.weapon.byId.get("SomeWeapon".AsHaxeString());
        // ... modify weapon properties ...
    }
}
```

`_Data_` contains all game data tables (`item`, `weapon`, `mob`, etc.). Query entries by ID via `byId.get()`, or iterate over all entries via `all`.

### Key API — CastleDB

| API | Description |
| ----- | ------ |
| `GenerateDiffCDB` | MSBuild property; when `true`, generates a difference patch instead of a full CDB |
| `IOnAfterLoadingCDB.OnAfterLoadingCDB(_Data_ cdb)` | Fires after CDB loading completes; allows reading/writing `cdb` |
| `cdb.{sheet}.byId.get(id)` | Gets an entry from the specified table by ID |
| `cdb.{sheet}.all` | Gets the list of all entries in the specified table |
| `cdb.{sheet}.byId.set(id, entry)` | Adds or overwrites an entry in the specified table |

### FAQ — CastleDB

**Compile-time diffs vs. runtime modification: which to choose?**

Compile-time diffs are suited for adding brand-new entries (new weapons, new skills). Runtime modification is suited for tweaking existing properties (adjusting values, unlock conditions). Both can be used together: runtime modifications take effect after diffs and can directly manipulate `_Data_` to override values from the patch.

**`GenerateDiffCDB` says template CDB not found?**

Make sure `GameVersion` matches the currently installed MDK version. The template CDB is located at `<MDK path>/databases/v<GameVersion>/data.cdb`.

**Do diffs and runtime modifications conflict?**

They don't. CDBManager merges patches first, then broadcasts the `IOnAfterLoadingCDB` event. Runtime modifications run inside the event and can override patch content.
