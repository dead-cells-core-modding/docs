---
sidebar_position: 1
---

# Installing MDK

**MDK (Mod Development Kit)** is the mod development toolkit provided by DCCM. This tutorial will guide you through installing and configuring MDK.

:::tip

Join the [Discord server](https://discord.gg/Z7zzVafaP3) for more help.

:::

## Prerequisites

- **.NET 10 SDK** ([Download](https://dotnet.microsoft.com/download/dotnet/10.0))
  - (Optional) Visual Studio 2022
- [DCCM Core Files](/docs/tutorial/install-core)

## Installation Steps

### Running the MDK Installation Script

- Open **File Explorer** and navigate to the `coremod/core/mdk` folder inside the game's root directory
- Right-click the `install.ps1` PowerShell script and select **Run with PowerShell**

## Verifying MDK Installation

Run the following command in PowerShell or Command Prompt:

```bash
dotnet nuget list source
```

You should see output similar to the following:

```text
Registered Sources:
  1.  DeadCoreModdingMDK [Enabled]
```
