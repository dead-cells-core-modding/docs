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

#### 3. 校验 modinfo.json

`modinfo.json` 是加载器识别 Mod 的唯一依据。安装前请确认以下内容：

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | Mod 名称，**必须与文件夹名完全一致**（区分大小写和空格） |
| `version` | 是 | Mod 版本号 |
| `type` | 是 | Mod 类型，通常为 `"normal"` |
| `dependencies` | 否 | 依赖的其他 Mod 名称列表 |
| `dccmversion` | 否 | 目标 DCCM 版本。若填写，主版本号必须与当前 DCCM 主版本一致，否则会收到版本不匹配警告 |

示例：

```json
{
  "name": "SampleHook",
  "version": "1.0.0",
  "type": "normal",
  "dependencies": [],
  "dccmversion": "2.0.0"
}
```

:::warning

如果 `name` 与文件夹名不一致，加载器可能：

- 将 Mod 识别为另一个名称，导致资源路径错误
- 与同名 Mod 冲突时被跳过加载
- 无法正确关联 Mod 的存储数据

:::

#### 4. 启动游戏

通过 `DeadCellsModding.exe` 启动游戏，加载器会自动扫描 `coremod/mods` 目录并加载所有有效的 Mods。

你可以在游戏主菜单左下角看到 DCCM 版本号，确认核心已正常加载。

## 常见问题

### Mod 无法加载

- 确认 `modinfo.json` 文件格式正确（可使用 JSON 验证工具）
- 确认 `name` 字段与文件夹名完全一致（包括大小写和空格）
- 查看游戏日志中的错误信息

### DLL 依赖缺失

- 确认 Mod 所需的 `.dll` 文件已放在 Mod 文件夹内
- 确认已安装 [.NET 10 运行时](./install-core.md#先决条件)
- 查看游戏日志中是否有 `FileNotFoundException` 或 `DllNotFoundException`

### DCCM 版本不匹配

- 确认 `modinfo.json` 中的 `dccmversion` 主版本号与当前 DCCM 主版本一致
- 若不需要版本检查，可删除 `dccmversion` 字段
- 查看游戏日志中是否有版本警告信息

### Mod 加载后无效果

- 查看游戏日志是否有警告或错误

### 多个 Mod 冲突

- 检查 Mod 依赖关系
- 尝试逐个启用 Mod 进行排查
- 查看游戏日志是否有警告或错误
