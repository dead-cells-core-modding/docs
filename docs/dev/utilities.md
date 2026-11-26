---
sidebar_position: 11
---

# Utilities

`ModCore.Utilities` contains public helper classes and extension methods used for C# ↔ Haxe interop, byte/array conversion, resource loading, and worker processes. Most helpers are extension methods, so a single `using ModCore.Utilities;` makes them available.

## StringUtils

Helpers for `dc.String` (the Haxe string type).

| API | Description |
| --- | --- |
| `string.AsHaxeString()` | Converts a C# string to `dc.String`. Short strings (≤ 1024 chars) are cached per thread. |
| `dc.String.Create(string)` | Static extension alias for `AsHaxeString()`. |
| `dc.String == dc.String` | Compares string content via `ToString()`. |
| `dc.String != dc.String` | Negated content comparison. |

```csharp
using dc;
using ModCore.Utilities;

dc.String a = "hello".AsHaxeString();
dc.String b = dc.String.Create("hello");

if (a == b)
{
    // Content comparison, not reference comparison
}
```

:::tip

Convert every C# string before passing it to a Haxe API. `AsHaxeString()` is the standard conversion used throughout the docs.

:::

## BytesUtils

Helpers for `dc.haxe.io.Bytes`.

| API | Description |
| --- | --- |
| `bytes.AsSpan()` | Returns a `Span<byte>` over the underlying Haxe buffer (writable) |
| `bytes.AsMemory()` | Returns a `Memory<byte>` over the buffer |
| `Bytes.Create(ReadOnlySpan<byte>)` | Allocates a Haxe `Bytes` and copies the data into it |

```csharp
using dc.haxe.io;
using ModCore.Utilities;

var bytes = Bytes.Create(data);
var span = bytes.AsSpan();
```

:::warning

`AsSpan()`/`AsMemory()` expose the existing Haxe buffer without copying. Do not keep the span after the `Bytes` object can be collected or freed.

:::

## ArrayUtils

Creates empty Haxe arrays with a shared, zeroed backing area. Useful when a Haxe API requires an array but you have no elements.

| API | Returns |
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

Adds `GetEnumerator` so Haxe arrays and iterators can be used with C# `foreach`.

| Overload | Works on |
| --- | --- |
| `GetEnumerator(this ArrayBase)` | Haxe `ArrayBase` |
| `GetEnumerator(this ArrayDyn)` | Haxe `ArrayDyn` |
| `GetEnumerator(this ArrayAccess)` | Dispatches to `ArrayBase`/`ArrayDyn`; throws `NotSupportedException` for unknown types |
| `GetEnumerator<T>(this virtual_hasNext_next_<HlFunc<T>>)` | Haxe iterators with `hasNext`/`next` |

```csharp
using ModCore.Utilities;

foreach (var item in haxeArray)
{
    Logger.Information("{item}", item);
}
```

Elements are returned as `dynamic`.

## ExtraDataUtils

Attaches arbitrary metadata to a Hashlink object through a dynamic `ExpandoObject`.

| API | Description |
| --- | --- |
| `HashlinkObj.Meta` | Dynamic per-object metadata |
| `HaxeProxyBase.Meta` | Same metadata, accessed through a proxy |

```csharp
using ModCore.Utilities;

someHaxeObject.Meta.Tag = "my value";
var tag = someHaxeObject.Meta.Tag;
```

:::warning

- The first access marks the Hashlink object as stateful.
- Metadata is in-memory only and is not saved or serialized.
- Access is not thread-safe; synchronize if multiple threads use the same object.

:::

## ResLoadUtils

Resource loading helpers for `dc.hxd.Res`.

| API | Description |
| --- | --- |
| `Res.LoadResEx(string path, string? ns = null)` | Loads `ns + "/" + path` when `path` is non-empty; loads `path` as-is when it is empty |
| `Res.TryLoadResEx(string path, out Any? res, string? ns = null)` | Same, but returns `false` instead of throwing |

```csharp
using dc.hxd;
using ModCore.Utilities;

if (Res.TryLoadResEx("data/config.txt", out var res, "my_mod"))
{
    Logger.Information("{text}", res!.toText());
}
```

If `path` is empty, `LoadResEx` loads it as-is. If `path` is non-empty and `ns` is `null`, the constructed name starts with `/`; pass an explicit namespace for normal resource paths.

## ColorUtils

| API | Description |
| --- | --- |
| `ColorUtils.PackColor(byte r, byte g, byte b)` | Packs RGB into a 32-bit integer (`0xRRGGBB`) |

```csharp
var color = ColorUtils.PackColor(0xFF, 0x80, 0x00); // 0xFF8000
```

:::info

The value is 24-bit RGB: red is in bits 16–23, green in bits 8–15, blue in bits 0–7. There is no alpha parameter; combine alpha separately if the target API needs it.

:::

## WorkerProcessUtils

Starts a separate host process that invokes a static method, which is useful for isolating expensive or crash-prone work.

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

Behavior:

- Launches `Environment.ProcessPath` with `--worker <type>` when no custom `startInfo.FileName` is provided.
- Sets `DCCM_CUSTOM_STARTUP_TYPE`, `DCCM_CUSTOM_STARTUP_METHOD`, `DCCM_LOAD_ADDITIONAL_ASSEMBLIES`, and (when `closeOnParentExit` is `true`) `DCCM_EXIT_WHEN_PROCESS_PID`.
- Clears `DCCM_SHOULD_WAIT_FOR_DEBUGGER` for the worker.

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

- The target type and static method must exist and be accessible in the worker process; pass the containing assembly in `loadAssemblies`.
- If `ContextConfig.Config.disableWorkerProcessUtils` is `true`, the method throws `InvalidOperationException`.
- If you supply a custom executable in `startInfo`, it must understand the DCCM worker environment variables.

:::

## Internal Utils

The framework also contains an internal `ModCore.Utils` class. It is **not** part of the public mod API, but framework contributors may encounter it:

| Member | Description |
| --- | --- |
| `Utils.MemCmp(a, b)` | `SequenceEqual` over two byte spans |
| `Utils.HashFile(path)` | SHA-256 hash of a file |
| `Utils.SafeGetAllTypes(assembly)` | `Assembly.GetTypes()` that tolerates `ReflectionTypeLoadException` |
| `Utils.HashlinkThrow(exception)` | Throws a .NET exception into the Hashlink VM |

Use `ModCore.Utilities` from mods; `ModCore.Utils` is intended for DCCM internals.
