---
sidebar_position: 1
---

# Manually Installing the DCCM

This tutorial will guide you through installing the Dead Cells Core Modding (DCCM) core.

:::tip

For **Steam** users, [installing from the Workshop](./install-workshop.md) is the best option.

:::

## Prerequisites

- **.NET 10 Runtime** ([Download Link](https://dotnet.microsoft.com/zh-cn/download/dotnet/10.0))
  - If you plan to develop **Mods** for **DCCM**, install the **.NET 10 SDK**.
- **Microsoft Visual C++ Redistributable Packages (2015-2022)** ([Download Link](https://aka.ms/vs/17/release/vc_redist.x64.exe))

## Installation

### Obtain DCCM Core

You can download the latest DCCM core from the [Github Releases](https://github.com/dead-cells-core-modding/core/releases/latest) page

If you wish to use new features not yet officially released, [this](https://nightly.link/dead-cells-core-modding/core/workflows/build/dev) might be more suitable for you

### Install DCCM Core Files

Open the game's root directory in your file manager and create a folder named `coremod`. Extract the DCCM core files obtained in the previous step into this folder

:::tip
After completing the above steps, your directory structure should resemble this:

```txt
<DeadCellsGameRoot>
├─ coremod
│  ├─ core
│  │  ├─ native
│  │  │  └─ …
│  │  ├─ mdk
│  │  │  ├─ install.ps1
│  │  │  ├─ uninstall.ps1
│  │  │  └─ …
│  │  ├─ host
│  │  │  ├─ startup
│  │  │  │  ├─ DeadCellsModding.exe
│  │  │  │  └─ …
│  │  │  └─ …
│  │  └─ …
│  └─ …
├─ deadcells.exe
├─ deadcells_gl.exe
└─ …
```

:::

:::tip
If you prefer, you can copy the `DeadCellsModding.exe` file to the game's root directory
:::

## Launching the Game

After completing the above steps, you can launch the modified game via `DeadCellsModding.exe`.

:::warning
Following initial installation or updates, the **first launch** may take extended time to generate caches (high memory usage).
:::

If everything goes smoothly, the **DCCM** version number will appear in the bottom-left corner of the game's main menu.

![DCCM](img/{D0E4CA71-3773-4DED-9C08-A8ABF9B6E9D9}.png)
