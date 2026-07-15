---
sidebar_position: 1
---

# Installing MDK

**MDK (Mod Development Kit)** is the mod development toolkit provided by DCCM. This tutorial will guide you through installing and performing basic configuration of MDK.

:::tip

Join the [Discord server](https://discord.gg/Z7zzVafaP3) for more help.

:::

## Prerequisites

- **.NET 10 SDK** ([Download Link](https://dotnet.microsoft.com/zh-cn/download/dotnet/10.0))
  - (Optional) Visual Studio 2022
- [DCCM Core Files](/docs/tutorial/install-core)

## Installation Steps

### Run the MDK Installation Script

- Open **File Explorer** and navigate to the `coremod/core/mdk` folder within the game's root directory
- Right-click and select **Run with PowerShell** to execute the `install.ps1` PowerShell script

## Verify MDK Installation

Execute the following command in PowerShell or Command Prompt:

```bash
dotnet nuget list source
```

You should see output similar to:

```text
Registered Sources:
  1.  DeadCoreModdingMDK [Enabled]
```
