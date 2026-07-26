---
sidebar_position: 4
---

# 发布 Mod

本教程介绍如何将开发完成的 Mod 发布到 **Steam 创意工坊**（Steam Workshop），以及如何为非 Steam 用户提供手动分发。DCCMTool 内置了完整的创意工坊上传与挂载功能，无需额外配置 Steamworks SDK。

:::warning

- 已完成 [第一个 Mod](/docs/dev/tutorial/first-mod) 教程，拥有可构建的 Mod 项目
- 已安装 **Steam 客户端**，并以拥有 Dead Cells 的账号登录
- Steam 客户端需保持运行状态，DCCMTool 通过本地 Steam API 与创意工坊通信

:::

## 准备发布

构建完成后，Mod 产物位于项目的 `bin\Debug\net10.0\output` 目录（参见[构建与测试](/docs/dev/tutorial/first-mod#构建与测试)）。在发布前，需要确保 `modinfo.json` 中的元信息完整。

### modinfo.json 关键字段

MDK 在构建时根据 `.csproj` 配置自动生成 `modinfo.json`。以下是与发布直接相关的字段：

| 字段 | MSBuild 来源 | 说明 |
| ---- | ---------- | ---- |
| `name` | `ModName` | Mod 名称，用于 Steam 搜索配对与本地挂载路径 |
| `version` | `Version` | 版本号，每次发布更新时递增 |
| `tags` | `ModTag` 项组 | 创意工坊标签，见下方说明 |
| `repositoryUrl` | `RepositoryUrl` | 源码仓库地址（可选），上传时不参与 Steam 流程 |
| `dccmversion` | 自动检测 | 构建时的 DCCM 版本号 |

### 配置标签

创意工坊通过标签帮助玩家发现 Mod。在 `.csproj` 中使用 `ModTag` 项组声明：

```xml
<ItemGroup>
  <ModTag Include="Gameplay" />
</ItemGroup>
```

多个标签可并列声明：

```xml
<ItemGroup>
  <ModTag Include="Gameplay" />
  <ModTag Include="Cosmetic" />
</ItemGroup>
```

DCCM 识别以下已知标签。使用非已知标签会触发警告，但仍会上传：

| 标签 | 含义 |
| ---- | ---- |
| `Gameplay` | 玩法修改 |
| `Test` | 测试用途 |
| `Language` | 语言/翻译 |
| `Cosmetic` | 外观/美化 |

### 配置版本号

版本号由 `<Version>` 属性控制，遵循 `主版本.次版本.修订号` 格式：

```xml
<PropertyGroup>
  <Version>1.0.0</Version>
</PropertyGroup>
```

每次向创意工坊上传更新前，应递增版本号以便玩家识别变更。

### 配置预览图

Steam 创意工坊条目需要一张预览图。在项目根目录放置以下文件之一，MDK 构建时会自动识别：

- `preview.png`
- `preview.jpg`
- `preview.gif`

也可通过 `<Preview>` 属性指定自定义路径：

```xml
<PropertyGroup>
  <Preview>Assets/workshop-preview.png</Preview>
</PropertyGroup>
```

上传时可通过 `-p` 参数指定预览图，覆盖构建时的默认值。

## Steam 上传

使用 `DCCMTool steam upload` 命令将 Mod 上传到创意工坊。

### 基本命令

```bash
DCCMTool steam upload -i <Mod目录> [-t 更新说明] [-p 预览图路径]
```

参数说明：

| 参数 | 必需 | 说明 |
| ---- | ---- | ---- |
| `-i, --input` | 是 | Mod 产物目录路径，即 `output` 目录 |
| `-t, --update-text` | 否 | 更新说明文本，显示在创意工坊条目的更新日志中 |
| `-p, --preview` | 否 | 预览图路径，不指定则使用 Mod 目录下的 `preview.*` 文件 |

### 上传示例

```bash
# 上传到创意工坊（使用默认更新文本和自动检测的预览图）
DCCMTool steam upload -i "bin\Debug\net10.0\output"

# 上传并附带更新说明
DCCMTool steam upload -i "bin\Debug\net10.0\output" -t "修复了武器伤害异常的问题"

# 指定自定义预览图
DCCMTool steam upload -i "bin\Debug\net10.0\output" -p "Assets\my-preview.png"
```

不指定 `-t` 时，更新文本默认为 `Update to v{version}`。

### 首次上传

首次上传时，DCCMTool 的行为：

1. 在创意工坊中搜索 `dccm_modname` 标签匹配的条目，未找到则判定为首次发布
2. 自动创建新的 Workshop 条目，标题格式为 `[DCCM] ModName`
3. 设置条目可见性为**公开**
4. 为条目添加 `dccm_modname` 键值标签，用于后续更新匹配
5. 上传 Mod 目录的全部内容
6. 输出 Workshop 条目链接，格式为 `https://steamcommunity.com/sharedfiles/filedetails/?id=XXXXXX`

:::info

DCCM 在上传时会为每个 Workshop 条目注入一个隐藏的 `dccm_modname` 键值标签，值为 Mod 的 `name` 字段。后续上传时通过此标签匹配已有条目，实现自动识别"首次 vs 更新"。该标签对普通玩家不可见。

:::

### 更新已有条目

当创意工坊中已存在同名（`dccm_modname` 匹配）的条目时，DCCMTool 自动进入更新模式：

1. 找到已有的 Workshop 条目 ID
2. 仅更新内容和标签（标题不再修改）
3. 提交更新文本

:::warning

如果创意工坊中存在**多个**同名条目（`dccm_modname` 重复），上传会报错并退出。这种情况通常不会出现，但如发生需手动去 Steam 创意工坊管理页面清理重复条目。

:::

### 上传进度

上传过程中 DCCMTool 会显示实时进度，包含三个阶段：

- **Preparing** — 准备配置和内容
- **Uploading** — 上传文件（显示字节进度）
- **Finalizing** — 完成提交

上传结束后输出 Workshop 条目链接，可直接在浏览器中访问。

### 故障处理

上传失败时，DCCMTool 会输出错误码。常见问题：

- **Steam 客户端未运行** — 确保 Steam 登录并处于在线状态
- **网络问题** — 创意工坊上传依赖 Steam 服务器，检查网络连接
- **权限不足** — 确认 Steam 账号拥有 Dead Cells 且未被社区封禁

详细错误码含义可参考 [Steamworks API 文档](https://partner.steamgames.com/doc/api/ISteamUGC#SubmitItemUpdateResult_t)。

## 版本管理

DCCM 不强制版本格式，但推荐遵循语义化版本规范。

每次发布更新时的建议流程：

1. 修改 `.csproj` 中 `<Version>` 的值
2. 重新执行 `dotnet build` 生成新的 `modinfo.json`
3. 运行 `steam upload -t "v1.0.1: 修复了……"` 上传更新

不指定 `-t` 时，工具自动生成 `Update to v{version}` 格式的更新文本。建议在维护性更新时使用简短的自定义说明。

## Steam 挂载

当你的 Mod 依赖了其他已发布到创意工坊的 Mod 时，需要使用 `steam mount` 将其挂载到本地。MDK 在编译时只能从本地 `coremod/mods/` 目录解析依赖，无法直接加载创意工坊中的 Mod。

:::info

假设你的 Mod 在 `.csproj` 中声明了依赖：

```xml
<ItemGroup>
  <ModDependency Include="LibraryMod" />
  <Reference Include="LibraryMod" />
</ItemGroup>
```

而 `LibraryMod` 仅发布在创意工坊上，`dotnet build` 会因找不到依赖而失败。此时使用 `steam mount` 将其挂载到本地即可正常编译。参见[模组依赖](/docs/dev/mdk#模组依赖)。

:::

```bash
DCCMTool steam mount -n <Mod名称> [-g 游戏目录] [-a false]
```

参数说明：

| 参数 | 必需 | 说明 |
| ---- | ---- | ---- |
| `-n, --name` | 是 | Mod 名称，对应 `modinfo.json` 中的 `name` 字段 |
| `-g, --game` | 否 | 游戏根目录路径，默认从 `DEAD_CELLS_GAME_PATH` 环境变量读取 |
| `-a, --mod-auto-subscribe` | 否 | 自动订阅并下载未安装的 Mod，默认 `true` |

### 挂载示例

```bash
# 挂载创意工坊 Mod（用于编译时依赖解析）
DCCMTool steam mount -n LibraryMod

# 指定游戏路径
DCCMTool steam mount -n LibraryMod -g "C:\Program Files\Steam\steamapps\common\Dead Cells"

# 禁用自动下载（仅挂载已安装的 Mod）
DCCMTool steam mount -n LibraryMod -a false
```

### 工作原理

1. 在已订阅的创意工坊项目中通过 `dccm_modname` 标签搜索
2. 未找到则在全部创意工坊中继续搜索
3. 若 Mod 未安装且 `-a` 为 `true`（默认），自动订阅并下载
4. 在 `{游戏目录}\coremod\mods\{Mod名称}` 创建符号链接指向 Workshop 安装路径

:::warning

`steam mount` 通过 `Directory.CreateSymbolicLink` 创建目录符号链接。在 Windows 上，创建符号链接需要以下条件之一：

- 以**管理员身份**运行终端
- 在 Windows 设置中启用**开发人员模式**（设置 → 更新和安全 → 开发者选项）
- 拥有 `SeCreateSymbolicLinkPrivilege` 用户权限

如果权限不足，命令会失败。Linux 上通常无需额外配置。

:::

挂载后，MDK 的依赖解析器（`DependenciesResolver`）即可在 `coremod/mods/` 下找到该 Mod 的 `modinfo.json` 和 DLL，完成编译。符号链接会自动跟随 Workshop 更新。

:::warning

`steam upload` 用于**发布**你的 Mod 到创意工坊，`steam mount` 用于**编译时**将创意工坊中的依赖 Mod 挂载到本地。运行游戏时，Steam 用户通过 DCCM 的 Workshop 加载器自动获取已订阅 Mod，无需手动挂载。

:::

:::warning

如果本地 `coremod/mods/` 下已存在与 Workshop Mod **同名**（`modinfo.json` 中 `name` 字段相同）的 Mod，**本地版本优先**。ModLoader 先扫描本地目录后扫描 Workshop 路径，同名 Workshop Mod 会被跳过并输出警告日志。在开发调试时应注意此行为，避免因本地旧版本覆盖 Workshop 新版本。

:::

## 手动分发

对于不使用 Steam 的玩家，可以将 Mod 产物打包为 zip 文件手动分发。

### 打包步骤

Mod 构建产物位于 `bin\Debug\net10.0\output`，目录结构如下：

```text
SimpleMod/
├── modinfo.json
├── SimpleMod.dll
├── res.pak          # 如启用了 GenerateSinglePakFile
└── ...
```

将此目录打包为 zip：

```bash
# Windows PowerShell
Compress-Archive -Path "bin\Debug\net10.0\output\*" -DestinationPath "MyMod-v1.0.0.zip"
```

### 安装说明

玩家解压 zip 后，需将内容放入游戏的 `coremod/mods/{ModName}` 目录。Mod 目录名必须与 `modinfo.json` 中 `name` 字段一致。

---

发布完成后，你可以在 Steam 创意工坊中管理条目信息、查看订阅数据和玩家反馈。如果发布了源码仓库的链接（`repositoryUrl`），建议在仓库 README 中反向链接到 Workshop 页面。
