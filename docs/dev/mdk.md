---
sidebar_position: 2
---

# MDK (Mod Development Kit) and Mod Dependencies

MDK is DCCM's mod development toolchain, responsible for resource packaging, CDB diff generation,
modinfo.json generation, and automatic installation. It is introduced via the NuGet package `DeadCellsCoreModding.MDK`.

## Installation

After installing DCCM, run the MDK installation script:

```powershell
# Windows
<game directory>/coremod/core/mdk/install.ps1
```

```bash
# Linux
<game directory>/coremod/core/mdk/install-linux.sh
```

The script registers the `DCCM_MDK_ROOT` environment variable via DCCMTool, enabling MSBuild to locate MDK build files.

## Project Reference

Add the NuGet reference in your `.csproj`:

```xml
<PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
```

The MDK package automatically imports `.props` and `.targets`, providing all build properties and targets.

## Build Properties

Configure the following properties in the `.csproj` `<PropertyGroup>`:

| Property | Required | Description |
| ------ | :--: | ------ |
| `ModName` | No | Mod name, defaults to `$(AssemblyName)`. Must match the `name` field in `modinfo.json` |
| `ModType` | Yes | Mod type: `mod` (regular mod) or `library` (library) |
| `ModMain` | Yes | Fully qualified name of the mod entry class, e.g. `MyMod.MainClass` |
| `AutoInstallMod` | No | When set to `true`, automatically copies output to `coremod/mods/` after build |
| `GenerateSinglePakFile` | No | When set to `true`, merges all `PackAssets` into a single `res.pak` |
| `GenerateDiffCDB` | No | When set to `true`, generates a CastleDB diff patch. **`GameVersion` must also be specified** |
| `GameVersion` | Conditional | Required when `GenerateDiffCDB=true`, e.g. `35` |
| `NoModCore` | No | When set to `true`, does not automatically reference framework assemblies such as ModCore |
| `Preview` | No | Path to the mod preview image (`.png`/`.jpg`/`.gif`), defaults to `preview.png` in the project directory |
| `ModTag` | No | Mod tag, written to the `tags` field in `modinfo.json` |
| `RepositoryUrl` | No | Mod repository URL, written to `modinfo.json` |
| `ModDependency` | No | Names of dependent mods (`<ModDependency Include="ModName" />`) |

## Resource Packaging Items

Declare in `<ItemGroup>`:

| Item | Description |
| ---- | ------ |
| `PackAssets` | Resource files to package. The `RootInPak` attribute specifies the root path inside the pak |
| `ModDependency` | Names of dependent mods |
| `OutputFiles` | Additional files to output to the build directory |

Example:

```xml
<ItemGroup>
    <PackAssets Include="assets/**/*" RootInPak="my_mod" />
    <ModDependency Include="LibraryMod" />
</ItemGroup>
```

## Build Process

During `dotnet build`, MDK executes in the following order:

1. **Resolve dependencies** — Locate mods specified by `<ModDependency>`, write to `modinfo.json`'s `dependencies`
2. **Generate modinfo.json** — Auto-generated from MSBuild properties. If a `modinfo.json` template exists in the project directory, merge with it
3. **Package resources** — DCCMTool packages `PackAssets` into intermediate `.pak` files
4. **Generate CDB patch** (when `GenerateDiffCDB=true`) — DCCMTool compares
   `data.cdb` against the `v$(GameVersion)` template, generating a diff `.pak`
5. **Merge PAK** — Merge intermediate `.pak` files into `res.pak`
6. **Output** — Copy DLL, `res.pak`, and preview image to `$(OutputPath)/output/$(ModName)/`
7. **Auto-install** (when `AutoInstallMod=true`) — Copy to `coremod/mods/$(ModName)/`

## DCCMTool

The core command-line tool of MDK, providing the following mod development commands:

| Command | Description |
| ------ | ------ |
| `pak pack files -i source=target_path -o output.pak` | Pack files into a PAK |
| `pak pack dir -i directory -o output.pak` | Pack a directory into a PAK |
| `pak merge -i a.pak -i b.pak -o merged.pak` | Merge multiple PAKs |
| `pak unpack -i file.pak -o output_dir` | Unpack a PAK |
| `cdb diff -i mod.cdb -t template.cdb -o diff.pak` | Generate a CastleDB diff |
| `steam upload` | Upload to Steam Workshop |
| `steam mount` | Mount Workshop mods locally |
| `atlas unpack -i atlas_file` | Unpack an atlas |
| `atlas colorswap decode/encode` | Decode/encode palette |
| `tmx collapse/expand` | TMX binary/XML conversion |

Direct invocation (in an MDK-installed environment):

```powershell
dotnet "$env:DCCM_MDK_ROOT/tools/DCCMTool.dll" pak unpack -i res.pak -o ./unpacked
```

## FAQ — MDK

**`dotnet build` says MDK not found?**

Ensure you have run `install.ps1` and that the system environment variable `DCCM_MDK_ROOT` points to the correct MDK path.

**`GenerateDiffCDB` errors with template CDB not found?**

The version specified by `GameVersion` must have a corresponding `v{version}/data.cdb` in MDK's `databases/` directory.

**No `res.pak` in the build output?**

Ensure `<PackAssets>` is declared in `<ItemGroup>` and `GenerateSinglePakFile` is not explicitly set to `false`.

---

## Mod Dependencies

Mods can reference other installed mods as dependencies. Declared via `<ModDependency>`, MDK automatically handles assembly references and `modinfo.json` generation.

## Declaring Dependencies

Dependency declaration comes in two steps:

```xml
<ItemGroup>
    <!-- Step 1: Add the dependent mod's directory to assembly search paths -->
    <ModDependency Include="LibraryMod" />

    <!-- Step 2: Reference that mod's DLL so its types are available at compile time -->
    <Reference Include="LibraryMod" />
</ItemGroup>
```

`<ModDependency>` adds the mod's directory to `AssemblySearchPaths`,
enabling MSBuild to find its DLL. `<Reference>` actually references the DLL for compilation,
making its types usable in code. **Both are required.**

You can specify a version requirement:

```xml
<ModDependency Include="SomeLib-1.2.0" />
```

The format is `ModName` or `ModName-version`. When a version is specified, MDK verifies that the installed version is not lower than required. `<Reference>` does not include the version number.

## How It Works

During build, MDK performs the following steps:

1. **Locate** — Search under `coremod/mods/` for `<ModName>/modinfo.json`, verify the `name` field matches the directory name
2. **Validate version** — If a version requirement is specified, check whether the installed version satisfies it
3. **Add to search paths** — Add the dependent mod's directory to `AssemblySearchPaths`, enabling `<Reference>` to resolve its DLL
4. **Write modinfo** — Write the dependency name to the `dependencies` array in `modinfo.json`

Generated `modinfo.json` example:

```json
{
    "name": "MyMod",
    "version": "1.0.0",
    "type": "mod",
    "main": "MyMod.MainClass",
    "dependencies": ["LibraryMod", "SomeLib"]
}
```

## Library Mods

Dependent mods should be of `library` type (`<ModType>library</ModType>`). Library mods do not include `ModMain` and only provide reusable types and methods.

## FAQ — Dependencies

**Dependent mod not found?**

Ensure the dependent mod is installed to `coremod/mods/<ModName>/` and that the directory name exactly matches the `name` field in `modinfo.json`.

**Types from the dependent mod's DLL are not visible?**

Ensure the dependent mod is correctly installed (`coremod/mods/<ModName>/<ModName>.dll` exists) and the `<ModType>` matches.
