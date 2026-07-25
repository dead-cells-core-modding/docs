---
sidebar_position: 3
---

# 配置与存档

DCCM 提供了两套数据持久化机制：`Config<T>` 用于模组自身的配置（JSON 文件），`SaveData<T>` 用于绑定游戏存档的模组数据。此外，`IHxbitSerializable<TData>` 接口让游戏对象能参与 Hxbit（游戏原生）序列化。

## Config\<T\>

`Config<T>` 是一个自动 JSON 序列化的配置系统。配置文件存储在 `{CORE_ROOT}/config/{name}.json`，读写对使用者完全透明。

### 声明

定义一个普通 C# 类作为配置模型，然后创建静态 Config 实例：

```csharp
using ModCore.Storage;

public class MyModConfig
{
    // 静态单例，构造参数为配置文件名（不含扩展名）
    public static Config<MyModConfig> Instance { get; } = new("MyMod");

    public double StartTimeout { get; set; } = 5;
    public bool EnableFeature { get; set; } = true;
}
```

`T` 必须有无参构造函数（`new()` 约束）。属性支持任意 JSON 可序列化的类型。

### 读写

通过 `Value` 属性读写配置：

```csharp
// 读取
double timeout = MyModConfig.Instance.Value.StartTimeout;

// 修改（会在合适的时机自动保存）
MyModConfig.Instance.Value.EnableFeature = false;
```

首次访问 `Value` 时自动从 JSON 文件加载；文件不存在则创建默认实例并保存，确保配置文件始终存在。

### 保存

`Config<T>` 实现了 `IOnSaveConfig` 接口，DCCM 会在合适时机（如游戏退出、模组卸载）自动触发保存。一般无需手动调用。

如需立即保存，可调用 `Save()`：

```csharp
MyModConfig.Instance.Save();
```

### 自定义序列化

修改 `SerializerOptions` 可调整 JSON 输出格式（默认开启缩进）：

```csharp
MyModConfig.Instance.SerializerOptions.Formatting = Formatting.None;
```

:::tip 外部配置库
DCCM 的 `Config<T>` 处理的是模组的 JSON 配置文件。如果你使用其他库的配置系统（如 `ConfigurationManager`），两者互不冲突，分别管理各自的文件。
:::

## SaveData\<T\>

`SaveData<T>` 将数据绑定到游戏存档。读取存档时自动恢复数据，保存游戏时自动写入。数据以 JSON 格式嵌入存档文件。

### 声明

```csharp
using ModCore.Storage;

public class MySaveData
{
    public int Coins { get; set; }
    public List<string> UnlockedItems { get; set; } = new();
}

// 注册实例，name 需要全局唯一
public static SaveData<MySaveData> Save { get; } = new("MyMod_SaveData");
```

与 `Config<T>` 不同：`Config<T>` 绑定 `new()` 约束，`SaveData<T>` 绑定 `class, new()`——因为存档数据需要通过 JSON 反序列化还原引用类型。

### 读写

同样通过 `Value` 属性操作：

```csharp
// 修改数据
MyMod.Save.Value.Coins += 100;
MyMod.Save.Value.UnlockedItems.Add("SomeWeapon");

// 读取数据
int coins = MyMod.Save.Value.Coins;
```

数据会在存档保存时自动写入，存档加载时自动恢复。开发者只需操作 `Value`，无需关心持久化细节。

### 与 Config 的区别

| | Config\<T\> | SaveData\<T\> |
|---|---|---|
| 存储位置 | `config/{name}.json` | 嵌入游戏存档文件 |
| 生命周期 | 模组级别，跨存档共享 | 存档级别，随存档独立 |
| 适用场景 | 模组设置（按键、开关） | 随存档变化的进度数据 |
| T 约束 | `new()` | `class, new()` |

## IHxbitSerializable\<TData\>

游戏内部使用 Hxbit 协议序列化运行态对象（武器、技能等）。当一个 C# 类继承自游戏原生类型并需要参与 Hxbit 序列化时，实现此接口。

### 接口定义

```csharp
namespace ModCore.Storage
{
    public interface IHxbitSerializable<TData>
    {
        TData GetData();       // 序列化时调用，返回要保存的数据
        void SetData(TData data); // 反序列化时调用，用数据恢复状态
    }
}
```

### 示例

下例中 `OtherDashSword` 继承自 `DashSword`（游戏原生武器类），实现 `IHxbitSerializable<object>` 以参与存档：

```csharp
using dc.en;
using ModCore.Storage;

public class OtherDashSword : DashSword, IHxbitSerializable<object>
{
    public OtherDashSword(Hero hero, InventItem item) : base(hero, item) { }

    object IHxbitSerializable<object>.GetData()
    {
        // 返回需要持久化的状态
        return new();
    }

    void IHxbitSerializable<object>.SetData(object data)
    {
        // 从存档恢复状态
    }
}
```

`TData` 通常为 `object`，实际类型由游戏侧的 Hxbit 引擎决定。如果你的类型无需额外状态持久化，两个方法留空即可——但实现接口本身是必需的，否则游戏加载时可能因找不到序列化处理器而报错。

## 关键 API

| API | 说明 |
|-----|------|
| `new Config<T>("name")` | 创建配置实例，自动注册到事件系统 |
| `Config<T>.Value` | 获取/设置配置值（首次访问自动加载） |
| `Config<T>.Save()` | 手动保存配置到 JSON 文件 |
| `Config<T>.SerializerOptions` | JSON 序列化设置（`Newtonsoft.Json.JsonSerializerSettings`） |
| `new SaveData<T>("name")` | 创建存档数据实例，自动注册到事件系统 |
| `SaveData<T>.Value` | 读取/写入存档数据 |
| `IHxbitSerializable<TData>.GetData()` | Hxbit 序列化时提取数据 |
| `IHxbitSerializable<TData>.SetData(data)` | Hxbit 反序列化时恢复状态 |
