---
sidebar_position: 3
---

# 资源与数据

模组可以通过 DCCM 的资源打包系统将自定义资源（图片、文本、数据等）打包为 `.pak` 文件，并在游戏加载时注入。

## 打包资源

在 `.csproj` 文件中配置 `PackAssets` 和 `GenerateSinglePakFile`：

```xml
<PropertyGroup>
    <!-- 将所有 PackAssets 合并为单个 res.pak -->
    <GenerateSinglePakFile>true</GenerateSinglePakFile>
</PropertyGroup>

<ItemGroup>
    <!--
        Include: 源文件路径（支持通配符）
        RootInPak: 资源在 pak 中的根路径，用于后续访问
    -->
    <PackAssets Include="assets/**/*" RootInPak="my_mod" />
</ItemGroup>
```

构建后，生成的 `res.pak` 位于 mod 输出目录中，与 `modinfo.json` 同级。

## 加载资源

在 mod 中实现 `IOnAfterLoadingAssets` 接口，通过 `FsPak` 加载 `res.pak`：

```csharp
using ModCore.Events.Interfaces;
using ModCore.Modules;
using ModCore.Utilities;

public class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingAssets
{
    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        // 获取 res.pak 的完整路径
        var pakPath = Info.ModRoot!.GetFilePath("res.pak");

        // 加载 pak 到游戏文件系统
        FsPak.Instance.FileSystem.loadPak(pakPath.AsHaxeString());
    }
}
```

`IOnAfterLoadingAssets` 在游戏资源加载完成后触发，是注入自定义资源的最佳时机。

## 访问资源

资源加载后，通过 `Res.Class.load()` 访问，路径格式为 `RootInPak/相对路径`：

```csharp
void IOnGameEndInit.OnGameEndInit()
{
    // 加载文本资源（RootInPak 为 "my_mod"，文件为 assets/data/config.txt）
    var text = Res.Class.load("my_mod/data/config.txt".AsHaxeString());
    Logger.Information("资源内容: {content}", text.toText());
}
```

## 关键 API — 资源

| API | 说明 |
| ----- | ------ |
| `GenerateSinglePakFile` | MSBuild 属性，设为 `true` 时合并 `PackAssets` 为单个 res.pak |
| `PackAssets` | MSBuild 项，声明要打包的资源文件及 `RootInPak` 根路径 |
| `Info.ModRoot.GetFilePath(name)` | 获取 mod 根目录下指定文件的完整路径 |
| `FsPak.Instance.FileSystem.loadPak(path)` | 将 `.pak` 文件加载到游戏资源系统 |
| `Res.Class.load(path)` | 按路径从已加载的资源中读取数据 |
| `.AsHaxeString()` | 将 C# 字符串转换为 Haxe 字符串（扩展方法，位于 `ModCore.Utilities`） |

## 常见问题 — 资源

**`Res.Class.load()` 返回 `null`？**

确认 `RootInPak` 前缀与加载路径一致。例如 `RootInPak="my_mod"` 时，访问路径应为 `"my_mod/xxx.txt"`。

**加载后资源未生效？**

`IOnAfterLoadingAssets` 触发较早，部分资源需等到
`IOnGameEndInit` 之后才能访问。对于需在特定时机加载的资源，
在对应生命周期事件中调用 `loadPak`。

---

## 数据修改（CastleDB）

CastleDB（简称 CDB）是游戏的核心数据表，定义武器、怪物、技能等所有内容属性。DCCM 支持两种 CDB 修改方式：编译时生成补丁和运行时直接修改。

### 编译时补丁

在 `.csproj` 中启用 `GenerateDiffCDB`，并指定 `GameVersion`：

```xml
<PropertyGroup>
    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
</PropertyGroup>
```

构建时 MDK 通过 DCCMTool 将 `data.cdb` 与指定版本 CDB 对比，
仅生成差异补丁。补丁随后合并入 `res.pak`。
启用 `GenerateDiffCDB` 会自动开启 `GenerateSinglePakFile`。

`GameVersion` 用于定位 MDK 模板数据库中对应版本的
CDB（`mdk/databases/v{version}/data.cdb`）。如果模组目录下没有
`data.cdb`，则仅通过运行时修改操作 CDB。

### 运行时修改

实现 `IOnAfterLoadingCDB` 接口，在 CDB 加载完成后直接操作 `_Data_` 对象：

```csharp
using dc;
using ModCore.Events.Interfaces.Game;

public class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingCDB
{
    public void OnAfterLoadingCDB(_Data_ cdb)
    {
        // 修改已有条目：将所有武器的售价设为 1
        cdb.item.byId.get("DeathMoney".AsHaxeString()).cellCost = 1;

        // 通过 byId 访问指定表的数据
        var weapon = cdb.weapon.byId.get("SomeWeapon".AsHaxeString());
        // ... 修改 weapon 属性 ...
    }
}
```

`_Data_` 包含游戏所有数据表（`item`、`weapon`、`mob` 等），
可通过 `byId.get()` 按 ID 查询，或通过 `all` 遍历全部条目。

### 关键 API — CastleDB

| API | 说明 |
| ----- | ------ |
| `GenerateDiffCDB` | MSBuild 属性，设为 `true` 时生成差异补丁而非完整 CDB |
| `IOnAfterLoadingCDB.OnAfterLoadingCDB(_Data_ cdb)` | CDB 加载完成后触发，可读写 `cdb` |
| `cdb.{sheet}.byId.get(id)` | 按 ID 获取指定表的条目 |
| `cdb.{sheet}.all` | 获取指定表的所有条目列表 |
| `cdb.{sheet}.byId.set(id, entry)` | 向指定表添加或覆盖条目 |

### 常见问题 — CastleDB

**编译时补丁与运行时修改如何选择？**

编译时补丁适合添加全新条目（新武器、新技能），
运行时修改适合调整已有属性（修改数值、解锁条件）。
两者可同时使用——运行时修改在补丁之后生效，
可直接操作 `_Data_` 覆盖补丁中的值。

**`GenerateDiffCDB` 提示找不到模板 CDB？**

确认 `GameVersion` 与当前安装的 MDK 版本一致。模板 CDB 位于 `<MDK路径>/databases/v<GameVersion>/data.cdb`。

**补丁与运行时修改冲突？**

不冲突。CDBManager 先合并补丁，再广播 `IOnAfterLoadingCDB` 事件。运行时修改在事件中执行，可覆盖补丁内容。
