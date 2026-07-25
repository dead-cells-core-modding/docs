---
sidebar_position: 4
---

# Publishing a Mod

This tutorial covers how to publish a completed mod to the **Steam Workshop** and how to provide manual distribution for non-Steam users. DCCMTool includes built-in Workshop upload and mount functionality, with no additional Steamworks SDK configuration required.

:::warning Prerequisites

- Completed the [Creating Your First Mod](/docs/dev/tutorial/first-mod) tutorial and have a buildable mod project
- **Steam client** installed and logged in with an account that owns Dead Cells
- The Steam client must remain running; DCCMTool communicates with the Workshop via the local Steam API

:::

## Preparing for Publication

After building, the mod output is located in the project's `bin\Debug\net10.0\output` directory (see [Building and Testing](first-mod.md#building-and-testing)). Before publishing, make sure the metadata in `modinfo.json` is complete.

### Key modinfo.json Fields

MDK automatically generates `modinfo.json` during the build based on the `.csproj` configuration. Here are the fields directly relevant to publishing:

| Field | MSBuild Source | Description |
| ---- | ---------- | ---- |
| `name` | `ModName` | Mod name, used for Steam search matching and local mount paths |
| `version` | `Version` | Version number; increment for each published update |
| `tags` | `ModTag` item group | Workshop tags; see below |
| `repositoryUrl` | `RepositoryUrl` | Source repository URL (optional); not used in the Steam upload process |
| `dccmversion` | Auto-detected | DCCM version used during the build |

### Configuring Tags

Workshop tags help players discover your mod. Declare them in `.csproj` using the `ModTag` item group:

```xml
<ItemGroup>
  <ModTag Include="Gameplay" />
</ItemGroup>
```

Multiple tags can be declared side by side:

```xml
<ItemGroup>
  <ModTag Include="Gameplay" />
  <ModTag Include="Cosmetic" />
</ItemGroup>
```

DCCM recognizes the following known tags. Using an unrecognized tag triggers a warning but will still upload:

| Tag | Meaning |
| ---- | ---- |
| `Gameplay` | Gameplay modification |
| `Test` | Testing purposes |
| `Language` | Language / translation |
| `Cosmetic` | Cosmetic / visual |

### Configuring the Version Number

The version number is controlled by the `<Version>` property, following the `major.minor.patch` format:

```xml
<PropertyGroup>
  <Version>1.0.0</Version>
</PropertyGroup>
```

Before each Workshop update, increment the version number so players can identify changes.

### Configuring a Preview Image

Steam Workshop entries require a preview image. Place one of the following files in the project root and MDK will detect it automatically during the build:

- `preview.png`
- `preview.jpg`
- `preview.gif`

You can also specify a custom path using the `<Preview>` property:

```xml
<PropertyGroup>
  <Preview>Assets/workshop-preview.png</Preview>
</PropertyGroup>
```

During upload, you can override the build-time default with the `-p` parameter.

## Steam Upload

Use the `DCCMTool steam upload` command to upload your mod to the Workshop.

### Basic Command

```bash
DCCMTool steam upload -i <mod directory> [-t update notes] [-p preview image path]
```

Parameter descriptions:

| Parameter | Required | Description |
| ---- | ---- | ---- |
| `-i, --input` | Yes | Path to the mod output directory, i.e., the `output` directory |
| `-t, --update-text` | No | Update notes text, shown in the Workshop entry's changelog |
| `-p, --preview` | No | Preview image path; if not specified, uses the `preview.*` file in the mod directory |

### Upload Examples

```bash
# Upload to Workshop (default update text and auto-detected preview)
DCCMTool steam upload -i "bin\Debug\net10.0\output"

# Upload with update notes
DCCMTool steam upload -i "bin\Debug\net10.0\output" -t "Fixed abnormal weapon damage"

# Specify a custom preview image
DCCMTool steam upload -i "bin\Debug\net10.0\output" -p "Assets\my-preview.png"
```

When `-t` is not specified, the update text defaults to `Update to v{version}`.

### First Upload

On first upload, DCCMTool's behavior:

1. Searches the Workshop for an entry matching the `dccm_modname` tag; if none is found, it's determined to be a first-time publish
2. Automatically creates a new Workshop entry with the title format `[DCCM] ModName`
3. Sets the entry visibility to **Public**
4. Adds a `dccm_modname` key-value tag to the entry for subsequent update matching
5. Uploads the entire contents of the mod directory
6. Outputs the Workshop entry link in the format `https://steamcommunity.com/sharedfiles/filedetails/?id=XXXXXX`

:::info The dccm_modname Tag Mechanism

During upload, DCCM injects a hidden `dccm_modname` key-value tag into each Workshop entry, with the value being the mod's `name` field. Subsequent uploads match existing entries via this tag, enabling automatic "first vs. update" detection. This tag is not visible to regular players.

:::

### Updating an Existing Entry

When a Workshop entry with the same name (matching `dccm_modname`) already exists, DCCMTool automatically enters update mode:

1. Finds the existing Workshop entry ID
2. Only updates the content and tags (the title is not modified)
3. Submits the update text

:::warning

If **multiple** entries with the same name (duplicate `dccm_modname`) exist in the Workshop, the upload will error and exit. This situation rarely occurs, but if it does, you'll need to manually clean up duplicate entries from the Steam Workshop management page.

:::

### Upload Progress

DCCMTool displays real-time progress during the upload, consisting of three phases:

- **Preparing** — preparing configuration and content
- **Uploading** — uploading files (shows byte progress)
- **Finalizing** — completing the submission

After the upload completes, the Workshop entry link is output and can be opened directly in a browser.

### Troubleshooting

When an upload fails, DCCMTool outputs an error code. Common issues:

- **Steam client not running** — make sure Steam is logged in and online
- **Network issues** — Workshop uploads depend on Steam servers; check your network connection
- **Insufficient permissions** — confirm the Steam account owns Dead Cells and is not community-banned

For detailed error code meanings, see the [Steamworks API Documentation](https://partner.steamgames.com/doc/api/ISteamUGC#SubmitItemUpdateResult_t).

## Version Management

DCCM doesn't enforce a version format, but semantic versioning is recommended.

Recommended workflow for each published update:

1. Change the `<Version>` value in `.csproj`
2. Re-run `dotnet build` to generate the new `modinfo.json`
3. Run `steam upload -t "v1.0.1: fixed ..."` to upload the update

When `-t` is not specified, the tool auto-generates update text in the `Update to v{version}` format. For maintenance updates, using a short custom description is recommended.

## Steam Mount

When your mod depends on other mods already published on the Workshop, you need to use `steam mount` to mount them locally. MDK can only resolve dependencies from the local `coremod/mods/` directory during compilation and cannot directly load Workshop mods.

:::info Typical Scenario

Suppose your mod declares a dependency in `.csproj`:

```xml
<ItemGroup>
  <ModDependency Include="LibraryMod" />
  <Reference Include="LibraryMod" />
</ItemGroup>
```

And `LibraryMod` is only published on the Workshop. `dotnet build` will fail because it can't find the dependency. Using `steam mount` to mount it locally allows normal compilation. See [Mod Dependencies](../mdk.md#mod-dependencies).

:::

```bash
DCCMTool steam mount -n <mod name> [-g game directory] [-a false]
```

Parameter descriptions:

| Parameter | Required | Description |
| ---- | ---- | ---- |
| `-n, --name` | Yes | Mod name, corresponding to the `name` field in `modinfo.json` |
| `-g, --game` | No | Path to the game root directory; defaults to the `DEAD_CELLS_GAME_PATH` environment variable |
| `-a, --mod-auto-subscribe` | No | Automatically subscribe to and download uninstalled mods; defaults to `true` |

### Mount Examples

```bash
# Mount a Workshop mod (for compile-time dependency resolution)
DCCMTool steam mount -n LibraryMod

# Specify the game path
DCCMTool steam mount -n LibraryMod -g "C:\Program Files\Steam\steamapps\common\Dead Cells"

# Disable auto-download (only mount already-installed mods)
DCCMTool steam mount -n LibraryMod -a false
```

### How It Works

1. Searches subscribed Workshop items using the `dccm_modname` tag
2. If not found, continues searching across all Workshop items
3. If the mod is not installed and `-a` is `true` (default), automatically subscribes and downloads
4. Creates a symbolic link at `{game directory}\coremod\mods\{mod name}` pointing to the Workshop install path

:::warning Permission Requirements

`steam mount` creates directory symbolic links via `Directory.CreateSymbolicLink`. On Windows, creating symbolic links requires one of the following:

- Running the terminal as **Administrator**
- Enabling **Developer Mode** in Windows settings (Settings → Update & Security → For developers)
- Having the `SeCreateSymbolicLinkPrivilege` user right

If you lack the necessary permissions, the command will fail. On Linux, no additional configuration is usually required.

:::

After mounting, MDK's dependency resolver (`DependenciesResolver`) can find the mod's `modinfo.json` and DLL under `coremod/mods/`, allowing compilation to proceed. The symbolic link automatically follows Workshop updates.

:::warning Difference Between the Two Commands

`steam upload` is for **publishing** your mod to the Workshop. `steam mount` is for mounting Workshop dependency mods locally during **compile time**. When running the game, Steam users automatically get subscribed mods through DCCM's Workshop loader; manual mounting is not needed.

:::

:::warning Same-Name Mod Conflicts

If a mod with the **same name** (matching the `name` field in `modinfo.json`) already exists under the local `coremod/mods/` directory, the **local version takes priority**. ModLoader scans the local directory first, then the Workshop path; same-name Workshop mods are skipped with a warning log. Be aware of this behavior during development and debugging to avoid an outdated local version overriding a Workshop update.

:::

## Manual Distribution

For players who don't use Steam, you can package the mod output as a zip file for manual distribution.

### Packaging Steps

The mod build output is located at `bin\Debug\net10.0\output` with the following directory structure:

```text
SimpleMod/
├── modinfo.json
├── SimpleMod.dll
├── res.pak          # if GenerateSinglePakFile is enabled
└── ...
```

Package this directory as a zip:

```bash
# Windows PowerShell
Compress-Archive -Path "bin\Debug\net10.0\output\*" -DestinationPath "MyMod-v1.0.0.zip"
```

### Installation Instructions

After players extract the zip, they need to place the contents into the game's `coremod/mods/{ModName}` directory. The mod's directory name must match the `name` field in `modinfo.json`.

---

After publishing, you can manage the entry details, view subscription data, and check player feedback on the Steam Workshop. If you published a source repository link (`repositoryUrl`), consider adding a backlink to the Workshop page in the repository README.
