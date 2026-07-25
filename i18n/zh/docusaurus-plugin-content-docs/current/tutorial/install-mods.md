---
sidebar_position: 3
---

# 安装 Mods

本教程将指导你如何将 Mods 安装到游戏中。

本文以 **SampleHook** Mod 为例，介绍 Mods 的安装方法。

:::info
本文所提到的 **Mods 目录** 的路径为 `<DeadCellsGameRoot>/coremod/mods`
:::

:::tip

加入 [Discord 服务器](https://discord.gg/7vp38qsYc4) 获取更多帮助。

:::

## 安装方式

### Steam 创意工坊安装

如果你通过 [Steam 创意工坊安装了 DCCM 核心](./install-workshop.md)，可以直接在 Steam 创意工坊中订阅 Mods。

大部分 DCCM Mods 在创意工坊的名称以 `[DCCM]` 开头，订阅后启动游戏即可自动安装。

### 手动安装

如果你 [手动安装了 DCCM 核心](./install-core.md)，请按以下步骤安装 Mods。

#### 1. 获取 Mods

你可以从任何你喜欢的渠道获取 Mods。

:::tip
对于任何一个**有效**的 Mod，其根目录下都应该存在 `modinfo.json`。

例如：

```txt
SampleHook
├─ modinfo.json
├─ SampleHook.dll
└─ SampleHook.pdb
```

:::

#### 2. 复制 Mods 文件

将 Mod 文件夹复制到 **Mods 目录** 下。

:::warning 文件夹命名规则

**文件夹名必须与 `modinfo.json` 中的 `name` 字段完全一致**（区分大小写和空格），否则加载器将无法正确识别该 Mod。

例如，若 `modinfo.json` 中 `name` 为 `"SampleHook"`，则文件夹名也必须为 `SampleHook`，不能是 `samplehook` 或 `Sample Hook`。

:::

:::tip

完成上述操作后，目录结构应该类似于：

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

#### 3. 启动游戏

通过 `DeadCellsModding.exe` 启动游戏，加载器会自动扫描 `coremod/mods` 目录并加载所有有效的 Mods。

你可以在游戏主菜单左下角看到 DCCM 版本号，确认核心已正常加载。

## 加载优先级

ModLoader 按以下顺序扫描 Mods：

1. **本地 Mods**：`coremod/mods/` 下的所有子文件夹
2. **Steam 创意工坊 Mods**：`DCCM_EXTRA_MODS_PATHS` 环境变量指向的 Workshop 路径（由 SteamStartShell 写入）

当本地 Mod 与 Workshop Mod **同名**（`modinfo.json` 中 `name` 相同）时，**本地版本优先**——先扫描到的本地 Mod 被加载后，后续遇到的同名 Workshop Mod 会被跳过并输出警告。

:::info 仅 Steam 启动可用

Steam 创意工坊 Mod 的自动发现依赖 SteamStartShell 写入的环境变量。直接通过 `DeadCellsModding.exe` 启动游戏时仅加载本地 Mods。

:::

