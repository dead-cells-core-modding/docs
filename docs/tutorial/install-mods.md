---
sidebar_position: 3
---

# Installing Mods

This tutorial will guide you through installing Mods into the game.

Using the **SampleHook** mod as an example, this article covers how to install Mods.

:::info
The path to the **Mods directory** mentioned in this article is `<DeadCellsGameRoot>/coremod/mods`
:::

:::tip

Join the [Discord server](https://discord.gg/7vp38qsYc4) for more help.

:::

## Installation Methods

### Steam Workshop Installation

If you [installed DCCM Core via Steam Workshop](./install-workshop.md), you can directly subscribe to Mods in the Steam Workshop.

Most DCCM Mods on the Workshop have names starting with `[DCCM]`. After subscribing, launch the game to automatically install them.

### Manual Installation

If you [manually installed DCCM Core](./install-core.md), follow the steps below to install Mods.

#### 1. Obtain Mods

You can obtain Mods from any channel you prefer.

:::tip
Any **valid** Mod should have a `modinfo.json` file in its root directory.

For example:

```txt
SampleHook
├─ modinfo.json
├─ SampleHook.dll
└─ SampleHook.pdb
```

:::

#### 2. Copy Mod Files

Copy the Mod folder into the **Mods directory**.

:::warning Folder Naming Rules

**The folder name must exactly match the `name` field in `modinfo.json`** (case-sensitive and space-sensitive), otherwise the loader will be unable to correctly recognize the Mod.

For example, if `name` in `modinfo.json` is `"SampleHook"`, the folder name must also be `SampleHook`, not `samplehook` or `Sample Hook`.

:::

:::tip

After completing the above steps, the directory structure should resemble:

```txt
<DeadCellsGameRoot>
├─ coremod
│  ├─ mods
│  │  ├─ SampleHook
│  │  │  ├─ modinfo.json
│  │  │  └─ …
│  │  └─ …
│  └─ …
└─ …
```

:::

#### 3. Launch the Game

Launch the game via `DeadCellsModding.exe`. The loader will automatically scan the `coremod/mods` directory and load all valid Mods.

You can check the DCCM version number in the bottom left corner of the game's main menu to confirm that the core has loaded successfully.

## Loading Priority

The ModLoader scans Mods in the following order:

1. **Local Mods**: all subfolders under `coremod/mods/`
2. **Steam Workshop Mods**: the Workshop path pointed to by the `DCCM_EXTRA_MODS_PATHS` environment variable (written by SteamStartShell)

When a local Mod and a Workshop Mod share the **same name** (same `name` in `modinfo.json`), the **local version takes priority**. Once a local Mod scanned first is loaded, subsequent Workshop Mods with the same name encountered later will be skipped with a warning.

:::info Steam Launch Only

The automatic discovery of Steam Workshop Mods depends on environment variables written by SteamStartShell. Launching the game directly via `DeadCellsModding.exe` will only load local Mods.

:::
