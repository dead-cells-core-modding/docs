---
sidebar_position: 2
---

# Creating Your First Mod

This tutorial will guide you through creating a simple mod that modifies a basic in-game feature.

:::warning

While **C#** is used for writing **Mods**, remember that **Dead Cells** is written in **Haxe** and runs on the **HaxeLink Virtual Machine**, not the **CoreCLR**.

This tutorial assumes you possess the following skills:

- Basic C# programming knowledge
- Foundational Dead Cells mod creation ([tutorial](https://www.bilibili.com/opus/681293864647000128))

:::

:::tip

Before starting, it's recommended to read the [wiki](https://github.com/HaxeFoundation/hashlink/wiki) for basic information about the **HashLink Virtual Machine**.

:::

:::info
The mod code for this tutorial is hosted on [Github](https://github.com/dead-cells-core-modding/docs-zh/blob/main/modproject/FirstDeadCellsMod).
:::

## Creating the Mod Project

- Open the command-line tool
- Create a new library project:

```bash
dotnet new classlib -n FirstDeadCellsMod -f net10.0
```

- Navigate to the project directory:

```bash
cd FirstDeadCellsMod
```

- Add the Dead Cells Modding MDK NuGet package reference:

```bash
dotnet add package DeadCellsCoreModding.MDK
```

## Create the Mod Main Class File

Create the mod main class file `ModEntry.cs`:

```csharp
using ModCore.Events.Interfaces.Game;
using ModCore.Mods;

namespace FirstDeadCellsMod
{
    public class FirstDeadCells : ModBase,
        IOnGameExit
    {
        public FirstDeadCells(ModInfo info) : base(info) 
        {

        }
        public override void Initialize()
        {
            Logger.Information(“Hello, world”);
        }

        void IOnGameExit.OnGameExit()
        {
            Logger.Information(“Game is exiting”);
        }
    }
}
```

## Configuring the Mod Project

Edit the project file `FirstDeadCellsMod.csproj` and add the following content:

```xml
<PropertyGroup>
    <!--Mod Type-->
    <ModType>mod</ModType>

    <!--FullName of Mod Main Class-->
    <ModMain>FirstDeadCellsMod.FirstDeadCells</ModMain>

    <!--Auto-install Mod during build-->
    <!--<AutoInstallMod>true</AutoInstallMod>-->
</PropertyGroup>

```

## Build the Mod

Run the build command in the project directory:

```bash
dotnet build
```

After successful build, the mod files will be generated in the `bin\Debug\net10.0\output` directory (`$(OutputPath)\output`).

## Test the Mod

- Install the mod following the [tutorial](/docs/tutorial/install-mods.md)
- Launch the game via `DeadCellsModding.exe`
- Check the game logs to confirm mod loading

:::info
You should see log entries similar to:

```text
[13:47:52 INF][FirstDeadCells] Hello, world
```

:::

## QA

### Build Fails

- Ensure .NET 10 SDK is installed
- Verify NuGet packages referenced in the project are available
- Try cleaning the solution and rebuilding:

```powershell
dotnet clean
dotnet build
```

### Mod Doesn't Load

1. Verify the `name` in `modinfo.json` exactly matches the folder name
2. Confirm the classpath specified in `ModMain` is correct
3. Check the log for error messages

### Game Crashes

- Inspect the Mod code for exceptions
- Try disabling other Mods for isolation testing
- Review the log for detailed information
