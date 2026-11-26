---
sidebar_position: 11
---

# 实用工具（Utilities）

`ModCore.Utilities` 包含用于 C# ↔ Haxe 互操作、字节/数组转换、资源加载与工作进程的公共辅助类与扩展方法。大多数辅助方法都是扩展方法，只需 `using ModCore.Utilities;` 即可使用。

## StringUtils

用于 `dc.String`（Haxe 字符串类型）的辅助方法。

| API | 说明 |
| --- | --- |
| `string.AsHaxeString()` | 将 C# 字符串转换为 `dc.String`。短字符串（≤ 1024 字符）会按线程缓存。 |
| `dc.String.Create(string)` | `AsHaxeString()` 的静态扩展别名。 |
| `dc.String == dc.String` | 通过 `ToString()` 比较字符串内容。 |
| `dc.String != dc.String` | 内容比较取反。 |

```csharp
using dc;
using ModCore.Utilities;

dc.String a = "hello".AsHaxeString();
dc.String b = dc.String.Create("hello");

if (a == b)
{
    // 内容比较，而不是引用比较
}
```

:::tip

把 C# 字符串传给 Haxe API 前都要先转换。`AsHaxeString()` 是文档中通用的转换方式。

:::

## BytesUtils

用于 `dc.haxe.io.Bytes` 的辅助方法。

| API | 说明 |
| --- | --- |
| `bytes.AsSpan()` | 返回底层 Haxe 缓冲区的 `Span<byte>`（可写） |
| `bytes.AsMemory()` | 返回缓冲区的 `Memory<byte>` |
| `Bytes.Create(ReadOnlySpan<byte>)` | 分配一个 Haxe `Bytes` 并复制数据 |

```csharp
using dc.haxe.io;
using ModCore.Utilities;

var bytes = Bytes.Create(data);
var span = bytes.AsSpan();
```

:::warning

`AsSpan()`/`AsMemory()` 直接暴露现有 Haxe 缓冲区，不会复制。`Bytes` 对象可能被回收或释放后，不要继续保留该 span。

:::

## ArrayUtils

创建共享零填充内存的空 Haxe 数组。适用于 Haxe API 需要数组但你没有元素的情况。

| API | 返回 |
| --- | --- |
| `ArrayUtils.CreateUI16()` | `ArrayBytes_hl_UI16` |
| `ArrayUtils.CreateInt()` | `ArrayBytes_Int` |
| `ArrayUtils.CreateSingle()` | `ArrayBytes_Single` |
| `ArrayUtils.CreateFloat()` | `ArrayBytes_Float` |
| `ArrayUtils.CreateDyn()` | `ArrayDyn` |

```csharp
using ModCore.Utilities;

var emptyInts = ArrayUtils.CreateInt();
```

## EnumerableUtils

添加 `GetEnumerator`，使 Haxe 数组与迭代器可用于 C# `foreach`。

| 重载 | 适用对象 |
| --- | --- |
| `GetEnumerator(this ArrayBase)` | Haxe `ArrayBase` |
| `GetEnumerator(this ArrayDyn)` | Haxe `ArrayDyn` |
| `GetEnumerator(this ArrayAccess)` | 分发到 `ArrayBase`/`ArrayDyn`；未知类型抛出 `NotSupportedException` |
| `GetEnumerator<T>(this virtual_hasNext_next_<HlFunc<T>>)` | 带 `hasNext`/`next` 的 Haxe 迭代器 |

```csharp
using ModCore.Utilities;

foreach (var item in haxeArray)
{
    Logger.Information("{item}", item);
}
```

元素以 `dynamic` 返回。

## ExtraDataUtils

通过动态 `ExpandoObject` 给 Hashlink 对象附加任意元数据。

| API | 说明 |
| --- | --- |
| `HashlinkObj.Meta` | 动态的逐对象元数据 |
| `HaxeProxyBase.Meta` | 通过代理访问同一份元数据 |

```csharp
using ModCore.Utilities;

someHaxeObject.Meta.Tag = "my value";
var tag = someHaxeObject.Meta.Tag;
```

:::warning

- 首次访问会把该 Hashlink 对象标记为 stateful。
- 元数据只存在于内存中，不会被保存或序列化。
- 访问不是线程安全的；多线程使用同一对象时请自行同步。

:::

## ResLoadUtils

`dc.hxd.Res` 的资源加载辅助方法。

| API | 说明 |
| --- | --- |
| `Res.LoadResEx(string path, string? ns = null)` | `path` 非空时加载 `ns + "/" + path`；`path` 为空时按原样加载 |
| `Res.TryLoadResEx(string path, out Any? res, string? ns = null)` | 同上，但失败时返回 `false` 而不是抛异常 |

```csharp
using dc.hxd;
using ModCore.Utilities;

if (Res.TryLoadResEx("data/config.txt", out var res, "my_mod"))
{
    Logger.Information("{text}", res!.toText());
}
```

如果 `path` 为空，`LoadResEx` 会按原样加载。如果 `path` 非空而 `ns` 为 `null`，构造出的名称会以 `/` 开头；正常资源路径请传入明确的命名空间。

## ColorUtils

| API | 说明 |
| --- | --- |
| `ColorUtils.PackColor(byte r, byte g, byte b)` | 将 RGB 打包为 32 位整数（`0xRRGGBB`） |

```csharp
var color = ColorUtils.PackColor(0xFF, 0x80, 0x00); // 0xFF8000
```

:::info

返回值是 24 位 RGB：红色位于第 16–23 位，绿色位于第 8–15 位，蓝色位于第 0–7 位。没有 alpha 参数；目标 API 需要 alpha 时请另行组合。

:::

## WorkerProcessUtils

启动一个独立的宿主进程来调用静态方法，适合隔离昂贵或容易崩溃的工作。

```csharp
public static Process StartWorkerProcess(
    string typeFullName,
    string methodName,
    ProcessStartInfo? startInfo,
    params ReadOnlySpan<string> loadAssemblies)

public static Process StartWorkerProcess(
    string typeFullName,
    string methodName,
    ProcessStartInfo? startInfo,
    bool closeOnParentExit,
    params ReadOnlySpan<string> loadAssemblies)
```

行为：

- 未提供自定义 `startInfo.FileName` 时，以 `--worker <type>` 启动 `Environment.ProcessPath`。
- 设置 `DCCM_CUSTOM_STARTUP_TYPE`、`DCCM_CUSTOM_STARTUP_METHOD`、`DCCM_LOAD_ADDITIONAL_ASSEMBLIES`，并在 `closeOnParentExit` 为 `true` 时设置 `DCCM_EXIT_WHEN_PROCESS_PID`。
- 清除工作进程的 `DCCM_SHOULD_WAIT_FOR_DEBUGGER`。

```csharp
using ModCore.Utilities;

var worker = WorkerProcessUtils.StartWorkerProcess(
    "MyMod.Worker",
    "Run",
    null,
    typeof(MyMod).Assembly.Location);

worker.WaitForExit();
```

:::warning

- 目标类型与静态方法必须存在，并且在 worker 进程中可访问；通过 `loadAssemblies` 传入所在程序集。
- 若 `ContextConfig.Config.disableWorkerProcessUtils` 为 `true`，该方法会抛出 `InvalidOperationException`。
- 如果在 `startInfo` 中指定了自定义可执行文件，它必须能识别 DCCM 的 worker 环境变量。

:::

## 内部 Utils

框架内部还有一个 `ModCore.Utils` 类。它**不属于**公共模组 API，但框架贡献者可能会遇到：

| 成员 | 说明 |
| --- | --- |
| `Utils.MemCmp(a, b)` | 对两个字节 span 做 `SequenceEqual` |
| `Utils.HashFile(path)` | 计算文件的 SHA-256 |
| `Utils.SafeGetAllTypes(assembly)` | 能容忍 `ReflectionTypeLoadException` 的 `Assembly.GetTypes()` |
| `Utils.HashlinkThrow(exception)` | 将 .NET 异常抛入 Hashlink 虚拟机 |

模组请使用 `ModCore.Utilities`；`ModCore.Utils` 仅供 DCCM 内部使用。
