---
sidebar_position: 2
---

# Creating Your First Mod

This tutorial will guide you through creating a runnable Dead Cells Mod from scratch. We'll build on the **SampleSimple** example project, setting up the project structure step by step, writing entry-point code, and loading custom assets.

:::warning

Although mods are written in **C#**, Dead Cells itself is built with **Haxe** and runs on the **HashLink virtual machine**, not the .NET CLR.

This tutorial assumes you already have:

- Basic C# programming knowledge
- [MDK installed](/docs/dev/tutorial/install-mdk)

:::

:::info
The complete source code for this tutorial is available at [SampleSimple](https://github.com/dead-cells-core-modding/DeadCellsCoreModding/tree/main/sample/SampleSimple).
:::

## Creating the Project

Open a terminal and run the following commands:

```bash
# Create a net10.0 class library project
dotnet new classlib -n SimpleMod -f net10.0

# Enter the project directory
cd SimpleMod

# Add the MDK NuGet package reference
dotnet add package DeadCellsCoreModding.MDK
```

After creating the project, delete the auto-generated `Class1.cs`. We'll create the real entry-point file later.

## Configuring the Project File

Edit `SimpleMod.csproj` and add the following MSBuild properties inside `<PropertyGroup>`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <!-- Mod type, set to mod for normal mods -->
    <ModType>mod</ModType>
    <!-- Mod name, used for output directory and log identification -->
    <ModName>Simple</ModName>
    <!-- Fully qualified entry-point class (Namespace.ClassName) -->
    <ModMain>SampleSimple.SimpleMod</ModMain>

    <!-- Auto-install the mod to the game directory after building -->
    <AutoInstallMod>true</AutoInstallMod>
    <!-- Bundle asset files into a single res.pak -->
    <GenerateSinglePakFile>true</GenerateSinglePakFile>
  </PropertyGroup>
  <!-- ... -->
</Project>
```

Property descriptions:

| Property | Description |
| ------ | ------ |
| `ModType` | Mod type; set to `mod` for normal mods |
| `ModName` | Mod name; affects output path and log identifiers |
| `ModMain` | Fully qualified entry-point class name; MDK uses this to reflectively load the mod |
| `AutoInstallMod` | When `true`, `dotnet build` automatically copies output to the game's mod directory |
| `GenerateSinglePakFile` | When `true`, assets declared in `PackAssets` are bundled into a single `res.pak` file |

## modinfo.json

MDK **automatically generates** `modinfo.json` during the build based on the csproj configuration. You don't need to write it manually, but it's useful to understand its structure:

```json
{
  "name": "Simple",
  "version": "1.0.0",
  "type": "mod",
  "dependencies": [],
  "dccmversion": "1.0.0"
}
```

| Field | Meaning |
| ------ | ------ |
| `name` | Mod name; must match the output folder name |
| `version` | Mod version number |
| `type` | Type, corresponding to `ModType` |
| `dependencies` | List of other mod names this mod depends on |
| `dccmversion` | DCCM version used during the build |

## Writing the Entry-Point Class

Create `SimpleMod.cs` and write the mod's entry-point class. The entry-point class needs to:

1. Inherit from `ModBase`
2. Override `Initialize()` for initialization
3. Implement lifecycle event interfaces to respond to game events

### Basic Skeleton

```csharp
using ModCore.Mods;

namespace SampleSimple
{
    public class SimpleMod(ModInfo info) : ModBase(info)
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }
    }
}
```

`ModBase`'s constructor takes a `ModInfo` parameter containing all the parsed information from `modinfo.json`. The `Info` property is accessible anywhere within the class.

### Implementing Lifecycle Events

DCCM exposes game events through **interfaces**. Implement the corresponding interface to receive callbacks when an event fires:

```csharp
using ModCore.Events.Interfaces;
using ModCore.Events.Interfaces.Game;

public class SimpleMod(ModInfo info) : ModBase(info),
    IOnGameExit,          // Before the game exits
    IOnGameEndInit,       // After game initialization completes
    IOnAfterLoadingAssets // After assets finish loading
{
    public override void Initialize()
    {
        Logger.Information("Hello, World!");
    }

    void IOnGameExit.OnGameExit()
    {
        Logger.Information("Game is exit");
    }

    void IOnGameEndInit.OnGameEndInit()
    {
        // Game initialization is complete; safe to access game data
    }

    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        // Asset loading is complete; safe to load custom assets
    }
}
```

:::tip Event Interface Naming
Event interfaces follow the `IOn<EventName>` naming convention. Interface methods use **explicit implementation** (`void IOnGameExit.OnGameExit()`) to avoid polluting the class's public interface.
:::

## Adding Assets

One of a mod's core capabilities is loading custom assets (textures, data, text, etc.). DCCM uses `PackAssets` to declare asset files and loads them within `IOnAfterLoadingAssets`.

### Declaring Assets

Add the following inside `<ItemGroup>` in the csproj:

```xml
<ItemGroup>
  <PackAssets Include="assets/**/*" RootInPak="sample_simple" />
</ItemGroup>
```

- `Include="assets/**/*"` — includes all files under the `assets` directory for packaging
- `RootInPak="sample_simple"` — these files will have `sample_simple` as their root path inside the pak

Create an `assets` folder in the project root and add a test file `test1.txt` with any content.

### Loading Assets

Load the pak in `IOnAfterLoadingAssets` and access assets in `IOnGameEndInit`:

```csharp
using dc.hxd;
using ModCore.Utilities;

void IOnAfterLoadingAssets.OnAfterLoadingAssets()
{
    var res = Info.ModRoot!.GetFilePath("res.pak");
    FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
}

void IOnGameEndInit.OnGameEndInit()
{
    var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());
    Logger.Information("The content of test1.txt is {text}", test1.toText());
}
```

Key points:

- `Info.ModRoot!.GetFilePath("res.pak")` gets the absolute path of `res.pak` under the mod's root directory
- `FsPak.Instance.FileSystem.loadPak()` mounts the pak into the game's file system
- `Res.Class.load()` accesses resources via their relative path inside the pak
- The `AsHaxeString()` extension method converts a C# string to a Haxe string, a necessary step for HashLink interop

## Complete Code

Putting everything together, here's the full `SimpleMod.cs`:

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
    public class SimpleMod(ModInfo info) : ModBase(info),
        IOnGameExit,
        IOnGameEndInit,
        IOnAfterLoadingAssets
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }

        void IOnAfterLoadingAssets.OnAfterLoadingAssets()
        {
            var res = Info.ModRoot!.GetFilePath("res.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
        }

        void IOnGameEndInit.OnGameEndInit()
        {
            var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());

            Logger.Information("The content of test1.txt is {text}", test1.toText());
        }

        void IOnGameExit.OnGameExit()
        {
            Logger.Information("Game is exit");
        }
    }
}
```

## Building and Testing

```bash
dotnet build
```

After a successful build, the output is located at `bin\Debug\net10.0\output`. If `AutoInstallMod` is enabled, the mod is automatically installed to the game's mod directory.

Launch the game via `DeadCellsModding.exe` and check the logs to confirm the mod loaded:

```text
[13:47:52 INF][Simple] Hello, World!
[13:47:53 INF][Simple] The content of test1.txt is <your text content>
```

## FAQ

### Build Fails

- Make sure [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) is installed
- Make sure the MDK NuGet source is configured (see [Installing MDK](/docs/dev/tutorial/install-mdk))
- Try cleaning and rebuilding:

```bash
dotnet clean
dotnet build
```

### Mod Doesn't Load

1. Check that the `name` in `modinfo.json` matches the output folder name
2. Verify that the class path specified in `ModMain` is fully correct (namespace + class name)
3. Check the logs for error messages

### Game Crashes

- Check whether the mod code throws unhandled exceptions
- Try disabling other mods to isolate the issue
- Make sure no `AsHaxeString()` calls are missing
