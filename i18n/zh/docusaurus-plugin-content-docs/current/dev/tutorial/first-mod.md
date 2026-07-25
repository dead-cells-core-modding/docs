---
sidebar_position: 1
---

# 创造第一个 Mod

本教程将指导你从零开始创建一个可运行的 Dead Cells Mod。我们将基于 **SampleSimple** 示例项目，逐步搭建项目结构、编写入口代码，并加载自定义资源。

:::warning

虽然编写 Mod 使用 **C#**，但 Dead Cells 本身由 **Haxe** 编写，运行在 **HashLink 虚拟机**上，而非 .NET CLR。

本教程假设你已具备：

- C# 编程基础
- 已安装 [MDK](/docs/dev/tutorial/install-mdk)

:::

:::info
本教程的完整代码可参考 [SampleSimple](https://github.com/dead-cells-core-modding/DeadCellsCoreModding/tree/main/sample/SampleSimple)。
:::

## 创建项目

打开命令行，执行以下步骤：

```bash
# 创建 net10.0 类库项目
dotnet new classlib -n SimpleMod -f net10.0

# 进入项目目录
cd SimpleMod

# 添加 MDK NuGet 包引用
dotnet add package DeadCellsCoreModding.MDK
```

项目创建后，删除自动生成的 `Class1.cs`，后面会创建真正的入口文件。

## 配置项目文件

编辑 `SimpleMod.csproj`，在 `<PropertyGroup>` 中添加以下 MSBuild 属性：

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>

    <!-- Mod 类型，固定为 mod -->
    <ModType>mod</ModType>
    <!-- Mod 名称，将用于输出目录和日志标识 -->
    <ModName>Simple</ModName>
    <!-- 入口类的完整限定名（命名空间.类名） -->
    <ModMain>SampleSimple.SimpleMod</ModMain>

    <!-- 构建后自动安装 Mod 到游戏目录 -->
    <AutoInstallMod>true</AutoInstallMod>
    <!-- 将资源文件打包为单个 res.pak -->
    <GenerateSinglePakFile>true</GenerateSinglePakFile>
  </PropertyGroup>
  <!-- ... -->
</Project>
```

各属性说明：

| 属性 | 作用 |
|------|------|
| `ModType` | Mod 类型，普通 Mod 设为 `mod` |
| `ModName` | Mod 名称，影响输出路径和日志中的标识 |
| `ModMain` | 入口类的完整限定名，MDK 通过此名称反射加载 Mod |
| `AutoInstallMod` | 设为 `true` 时，`dotnet build` 会将产物自动复制到游戏 Mod 目录 |
| `GenerateSinglePakFile` | 设为 `true` 时，`PackAssets` 中的资源会被打包成单个 `res.pak` 文件 |

## modinfo.json

MDK 会在构建时根据 csproj 配置**自动生成** `modinfo.json`。你无需手动编写，但有必要了解其结构：

```json
{
  "name": "Simple",
  "version": "1.0.0",
  "type": "mod",
  "dependencies": [],
  "dccmversion": "1.0.0"
}
```

| 字段 | 含义 |
|------|------|
| `name` | Mod 名称，必须与输出文件夹名一致 |
| `version` | Mod 版本号 |
| `type` | 类型，对应 `ModType` |
| `dependencies` | 依赖的其他 Mod 名称列表 |
| `dccmversion` | 构建时使用的 DCCM 版本 |

## 编写入口类

创建 `SimpleMod.cs`，编写 Mod 入口类。入口类需要：

1. 继承 `ModBase`
2. 重写 `Initialize()` 完成初始化
3. 实现生命周期事件接口以响应游戏事件

### 基本骨架

```csharp
using ModCore.Mods;

namespace SampleSimple
{
    public class SimpleMod(ModInfo info) : ModBase(info)
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }
    }
}
```

`ModBase` 的构造函数接收一个 `ModInfo` 参数，其中包含 `modinfo.json` 解析后的所有信息。`Info` 属性可在类的任何地方访问这些信息。

### 实现生命周期事件

DCCM 通过**接口**暴露游戏事件。实现对应接口即可在事件触发时收到回调：

```csharp
using ModCore.Events.Interfaces;
using ModCore.Events.Interfaces.Game;

public class SimpleMod(ModInfo info) : ModBase(info),
    IOnGameExit,          // 游戏退出前
    IOnGameEndInit,       // 游戏初始化完成后
    IOnAfterLoadingAssets // 资源加载完成后
{
    public override void Initialize()
    {
        Logger.Information("Hello, World!");
    }

    void IOnGameExit.OnGameExit()
    {
        Logger.Information("Game is exit");
    }

    void IOnGameEndInit.OnGameEndInit()
    {
        // 游戏初始化完成，可以安全访问游戏数据
    }

    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        // 资源加载完成，可以安全加载自定义资源
    }
}
```

:::tip 事件接口命名
事件接口遵循 `IOn<事件名>` 命名规则。接口方法使用**显式实现**（`void IOnGameExit.OnGameExit()`），避免污染类的公共接口。
:::

## 添加资源

Mod 的核心能力之一是加载自定义资源（贴图、数据、文本等）。DCCM 使用 `PackAssets` 声明资源文件，并在 `IOnAfterLoadingAssets` 中加载。

### 声明资源

在 csproj 的 `<ItemGroup>` 中添加：

```xml
<ItemGroup>
  <PackAssets Include="assets/**/*" RootInPak="sample_simple" />
</ItemGroup>
```

- `Include="assets/**/*"` — 将 `assets` 目录下的所有文件纳入打包范围
- `RootInPak="sample_simple"` — 这些文件在 pak 中的根路径为 `sample_simple`

在项目根目录创建 `assets` 文件夹，放入一个测试文件 `test1.txt`，内容随意。

### 加载资源

在 `IOnAfterLoadingAssets` 中加载 pak，在 `IOnGameEndInit` 中访问资源：

```csharp
using dc.hxd;
using ModCore.Utilities;

void IOnAfterLoadingAssets.OnAfterLoadingAssets()
{
    var res = Info.ModRoot!.GetFilePath("res.pak");
    FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
}

void IOnGameEndInit.OnGameEndInit()
{
    var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());
    Logger.Information("The content of test1.txt is {text}", test1.toText());
}
```

关键点：

- `Info.ModRoot!.GetFilePath("res.pak")` 获取 Mod 根目录下 `res.pak` 的绝对路径
- `FsPak.Instance.FileSystem.loadPak()` 将 pak 挂载到游戏文件系统
- `Res.Class.load()` 通过 pak 内的相对路径访问资源
- `AsHaxeString()` 扩展方法将 C# 字符串转换为 Haxe 字符串，这是 HashLink 互操作的必要步骤

## 完整代码

整合以上所有内容，`SimpleMod.cs` 的完整代码如下：

```csharp
using dc;
using dc.hxd;
using ModCore.Events.Interfaces;
using ModCore.Events.Interfaces.Game;
using ModCore.Mods;
using ModCore.Modules;
using ModCore.Utilities;

namespace SampleSimple
{
    public class SimpleMod(ModInfo info) : ModBase(info),
        IOnGameExit,
        IOnGameEndInit,
        IOnAfterLoadingAssets
    {
        public override void Initialize()
        {
            Logger.Information("Hello, World!");
        }

        void IOnAfterLoadingAssets.OnAfterLoadingAssets()
        {
            var res = Info.ModRoot!.GetFilePath("res.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
        }

        void IOnGameEndInit.OnGameEndInit()
        {
            var test1 = Res.Class.load("sample_simple/test1.txt".AsHaxeString());

            Logger.Information("The content of test1.txt is {text}", test1.toText());
        }

        void IOnGameExit.OnGameExit()
        {
            Logger.Information("Game is exit");
        }
    }
}
```

## 构建与测试

```bash
dotnet build
```

构建成功后，产物位于 `bin\Debug\net10.0\output`。如果启用了 `AutoInstallMod`，Mod 会被自动安装到游戏 Mod 目录。

通过 `DeadCellsModding.exe` 启动游戏，检查日志确认 Mod 加载：

```text
[13:47:52 INF][Simple] Hello, World!
[13:47:53 INF][Simple] The content of test1.txt is <你的文本内容>
```

## 常见问题

### 构建失败

- 确认已安装 [.NET 10 SDK](https://dotnet.microsoft.com/zh-cn/download/dotnet/10.0)
- 确认 MDK NuGet 源已配置（参考[安装 MDK](/docs/dev/tutorial/install-mdk)）
- 尝试清理后重新构建：

```bash
dotnet clean
dotnet build
```

### Mod 不加载

1. 检查 `modinfo.json` 中的 `name` 与输出文件夹名是否一致
2. 确认 `ModMain` 指定的类路径完全正确（命名空间 + 类名）
3. 查看日志中的错误信息

### 游戏崩溃

- 检查 Mod 代码是否抛出未处理的异常
- 尝试禁用其他 Mod 进行隔离测试
- 确认 `AsHaxeString()` 调用没有遗漏
