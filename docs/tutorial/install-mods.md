---
sidebar_position: 3
---

# Installing Mods

This tutorial will guide you through installing mods into the game.

Using the **SampleHook** mod as an example, we'll cover the installation process for **Mods**.

:::info
The **Mods directory** referenced in this guide is located at `<DeadCellsGameRoot>/coremod/mods`
:::

## Obtaining Mods

You may acquire **Mods** from any source of your choice

:::tip
Any **valid** Mod should contain a `modinfo.json` file within its root directory
For example

```txt
SampleHook
├─ modinfo.json
├─ SampleHook.dll
└─ SampleHook.pdb
```

:::

## Copying Mod Files

Copy the Mod folder into the **Mods directory**

:::tip

After completing the above steps, the directory structure should resemble:

```txt
<DeadCellsGameRoot>
├─ coremod
│  ├─ mods
│  │  ├─ SampleHook
|  |  |  ├─ modinfo.json
|  |  |  └─ …
|  |  └─ …
|  └─ …
└─ …
```

:::

## Common Issues

### Mod fails to load

- Verify the `modinfo.json` file format is correct (use a JSON validator tool)
- Check the game logs for error messages

### Mod loads but has no effect

- Check the game logs for warnings or errors

### Multiple mods conflict

- Check mod dependencies
- Try enabling mods one by one to troubleshoot
- Check the game logs for warnings or errors
