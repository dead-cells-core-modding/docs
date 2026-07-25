---
sidebar_position: 2
---

# MDK（Mod Development Kit）

MDK 是 DCCM 提供的模组开发工具链，负责构建时的资源打包、CastleDB 差异生成、modinfo.json 生成和自动安装。通过 NuGet 包 `DeadCellsCoreModding.MDK` 引入。

## 安装

安装 DCCM 后，运行 MDK 安装脚本：

```powershell
# Windows
<游戏目录>/coremod/core/mdk/install.ps1
```

```bash
# Linux
<游戏目录>/coremod/core/mdk/install-linux.sh
```

脚本通过 DCCMTool 注册 `DCCM_MDK_ROOT` 环境变量，使 MSBuild 能定位 MDK 构建文件。

## 项目引用

在 `.csproj` 中添加 NuGet 引用：

```xml
<PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
```

MDK 包自动导入 `.props` 和 `.targets`，提供所有构建属性与目标。

## 构建属性

以下属性在 `.csproj` 的 `<PropertyGroup>` 中配置：

| 属性 | 必须 | 说明 |
|------|:--:|------|
| `ModName` | 否 | 模组名称，默认取 `$(AssemblyName)`。必须与 `modinfo.json` 的 `name` 一致 |
| `ModType` | 是 | 模组类型：`mod`（普通模组）或 `library`（库） |
| `ModMain` | 是 | 模组入口类的完全限定名，如 `MyMod.MainClass` |
| `AutoInstallMod` | 否 | 设为 `true` 时，构建后自动复制到 `coremod/mods/` |
| `GenerateSinglePakFile` | 否 | 设为 `true` 时，将所有 `PackAssets` 合并为单个 `res.pak` |
| `GenerateDiffCDB` | 否 | 设为 `true` 时生成 CastleDB 差异补丁，**需同时指定 `GameVersion`** |
| `GameVersion` | 条件 | `GenerateDiffCDB=true` 时必须指定，如 `35` |
| `NoModCore` | 否 | 设为 `true` 时不自动引用 ModCore 等框架程序集 |
| `Preview` | 否 | 模组预览图路径（`.png`/`.jpg`/`.gif`），默认取项目目录下的 `preview.png` |
| `ModTag` | 否 | 模组标签，写入 `modinfo.json` 的 `tags` 字段 |
| `RepositoryUrl` | 否 | 模组仓库地址，写入 `modinfo.json` |
| `ModDependency` | 否 | 依赖的其他模组名称（`<ModDependency Include="ModName" />`） |

## 资源打包项

在 `<ItemGroup>` 中声明：

| 项 | 说明 |
|----|------|
| `PackAssets` | 要打包的资源文件，属性 `RootInPak` 指定在 pak 中的根路径 |
| `ModDependency` | 依赖的模组名称 |
| `OutputFiles` | 额外输出到构建目录的文件 |

示例：

```xml
<ItemGroup>
    <PackAssets Include="assets/**/*" RootInPak="my_mod" />
    <ModDependency Include="LibraryMod" />
</ItemGroup>
```

## 构建流程

`dotnet build` 时 MDK 按以下顺序执行：

1. **解析依赖** — 查找 `<ModDependency>` 指定的模组，写入 `modinfo.json` 的 `dependencies`
2. **生成 modinfo.json** — 从 MSBuild 属性自动生成。如果项目目录存在 `modinfo.json` 模板，与之合并
3. **打包资源** — DCCMTool 将 `PackAssets` 打包为中间 `.pak`
4. **生成 CDB 补丁**（`GenerateDiffCDB=true` 时）— DCCMTool 将 `data.cdb` 与 `v$(GameVersion)` 模板对比，生成差异 `.pak`
5. **合并 PAK** — 合并中间 `.pak` 为 `res.pak`
6. **输出** — 复制 DLL、`res.pak`、预览图到 `$(OutputPath)/output/$(ModName)/`
7. **自动安装**（`AutoInstallMod=true` 时）— 复制到 `coremod/mods/$(ModName)/`

## DCCMTool

MDK 的核心命令行工具，提供以下与模组开发相关的命令：

| 命令 | 说明 |
|------|------|
| `pak pack files -i 源=目标路径 -o 输出.pak` | 打包文件为 PAK |
| `pak pack dir -i 目录 -o 输出.pak` | 打包目录为 PAK |
| `pak merge -i a.pak -i b.pak -o 合并.pak` | 合并多个 PAK |
| `pak unpack -i 文件.pak -o 输出目录` | 解包 PAK |
| `cdb diff -i mod.cdb -t 模板.cdb -o 差异.pak` | 生成 CastleDB 差异 |
| `steam upload` | 上传到 Steam 创意工坊 |
| `steam mount` | 挂载创意工坊模组到本地 |
| `atlas unpack -i 图集文件` | 解包图集 |
| `atlas colorswap decode/encode` | 解码/编码调色板 |
| `tmx collapse/expand` | TMX 二进制/XML 互转 |

直接调用（在已安装 MDK 的环境中）：

```powershell
dotnet "$env:DCCM_MDK_ROOT/tools/DCCMTool.dll" pak unpack -i res.pak -o ./unpacked
```

## 常见问题

**`dotnet build` 提示找不到 MDK？**

确认已运行 `install.ps1`，且系统环境变量 `DCCM_MDK_ROOT` 指向正确的 MDK 路径。

**`GenerateDiffCDB` 报错找不到模板 CDB？**

`GameVersion` 指定的版本必须在 MDK 的 `databases/` 目录下有对应的 `v{version}/data.cdb`。

**构建输出中没有 `res.pak`？**

确认 `<ItemGroup>` 中声明了 `<PackAssets>` 且 `GenerateSinglePakFile` 未显式设为 `false`。

---

# 模组依赖

模组可以引用其他已安装的模组作为依赖。通过 `<ModDependency>` 声明，MDK 自动处理程序集引用和 `modinfo.json` 生成。

## 声明依赖

依赖声明分两步：

```xml
<ItemGroup>
    <!-- 步骤1：将依赖模组的目录加入程序集搜索路径 -->
    <ModDependency Include="LibraryMod" />

    <!-- 步骤2：引用该模组的 DLL，使类型在编译时可用 -->
    <Reference Include="LibraryMod" />
</ItemGroup>
```

`<ModDependency>` 将模组文件夹加入 `AssemblySearchPaths`，使 MSBuild 能找到其 DLL。`<Reference>` 实际引用 DLL 进行编译，使代码中可使用该模组的类型。**两者缺一不可**。

可指定版本要求：

```xml
<ModDependency Include="SomeLib-1.2.0" />
```

格式为 `ModName` 或 `ModName-version`。指定版本时 MDK 会校验已安装版本不低于所需版本。`<Reference>` 不包含版本号。

## 工作原理

构建时 MDK 执行以下步骤：

1. **查找** — 在 `coremod/mods/` 下查找 `<ModName>/modinfo.json`，验证 `name` 字段与目录名一致
2. **校验版本** — 若指定了版本要求，检查已安装版本是否满足
3. **加入搜索路径** — 将依赖模组的目录加入 `AssemblySearchPaths`，使 `<Reference>` 能解析其 DLL
4. **写入 modinfo** — 将依赖名称写入 `modinfo.json` 的 `dependencies` 数组

生成的 `modinfo.json` 示例：

```json
{
    "name": "MyMod",
    "version": "1.0.0",
    "type": "mod",
    "main": "MyMod.MainClass",
    "dependencies": ["LibraryMod", "SomeLib"]
}
```

## 库模组

依赖模组应为 `library` 类型（`<ModType>library</ModType>`）。库模组不包含 `ModMain`，仅提供可复用的类型与方法。

## 常见问题

**提示找不到依赖模组？**

确认依赖模组已安装到 `coremod/mods/<ModName>/`，且目录名与 `modinfo.json` 的 `name` 字段完全一致。

**依赖模组的 DLL 中的类型不可见？**

确认依赖模组已正确安装（存在 `coremod/mods/<ModName>/<ModName>.dll`）且 `<ModType>` 匹配。
