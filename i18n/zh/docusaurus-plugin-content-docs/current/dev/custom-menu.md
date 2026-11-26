---
sidebar_position: 8
---

# 自定义菜单

DCCM 会在游戏的设置（Options）菜单中加入一个 **Core Modding** 入口。模组可以在该入口下添加自己的设置页面，并使用游戏原生的 `dc.ui.Options` 控件 API 构建页面，因此外观和行为与游戏原版设置页一致。

## 菜单是如何被发现的

构建 Core Modding 页面时，`MenuModule` 会扫描所有已加载的 `ModBase` 接收者。模组可以通过两种方式提供菜单：

- 直接在 `ModBase` 类上实现 `ModCore.Menu.IModMenu`；
- 或者实现 `IModMenuProvider`，在 `GetModMenu()` 中返回一个 `IModMenu` 实例。

无需手动注册。菜单入口会出现在 **设置 → Core Modding →（你的菜单名）**。

## IModMenu

```csharp
namespace ModCore.Menu
{
    public interface IModMenu
    {
        string GetName();
        string? GetSubText() => null;
        void BuildMenu( dc.ui.Options options );
    }
}
```

| 成员 | 作用 |
| --- | --- |
| `GetName()` | 入口标签。DCCM 会把它传入 `GetText.Instance.GetString`，因此可以返回翻译键。 |
| `GetSubText()` | 入口下方的可选描述，可返回 `null`。 |
| `BuildMenu(options)` | 构建页面。每次打开该分节时都会调用，应从零开始构建。 |

## 最小示例

```csharp
using dc.ui;
using HaxeProxy.Runtime;
using ModCore.Menu;
using ModCore.Mods;
using ModCore.Modules;
using ModCore.Storage;

public class MyConfig
{
    public static Config<MyConfig> Instance { get; } = new("MyMod");

    public bool EnableFeature { get; set; } = true;
    public double SpawnRate { get; set; } = 1.0;
}

public class MyMod(ModInfo info) : ModBase(info), IModMenu
{
    string IModMenu.GetName() => "My Mod";
    string? IModMenu.GetSubText() => "Example settings";

    void IModMenu.BuildMenu( Options options )
    {
        options.title.set_text("MY MOD".AsHaxeString());
        options.createScroller(1);

        var flow = options.scrollerFlow;

        options.addSimpleWidget(
            "Log a message".AsHaxeString(),
            "Writes a line to the log".AsHaxeString(),
            () => Logger.Information("Hello from the menu"),
            Ref<int>.In(5),
            flow);

        options.addToggleWidget(
            "Enable feature".AsHaxeString(),
            "".AsHaxeString(),
            () =>
            {
                var cfg = MyConfig.Instance.Value;
                cfg.EnableFeature = !cfg.EnableFeature;
                MyConfig.Instance.Save();
                return cfg.EnableFeature;
            },
            Ref<bool>.In(MyConfig.Instance.Value.EnableFeature),
            flow);

        options.addSliderWidget(
            "Spawn rate".AsHaxeString(),
            new HlAction<double>(v =>
            {
                MyConfig.Instance.Value.SpawnRate = v;
                MyConfig.Instance.Save();
            }),
            MyConfig.Instance.Value.SpawnRate,
            Ref<double>.In(1.0),   // 步进
            flow,
            Ref<bool>.In(false),   // 显示百分比
            Ref<bool>.In(true),    // 显示原始数值
            Ref<double>.In(0.1),   // 最小值
            Ref<double>.In(10.0),  // 最大值
            null,                  // 按钮文本
            Ref<int>.Null);        // 内边距

        options.addSeparator("".AsHaxeString(), flow);
    }
}
```

要点：

- `options.createScroller(1)` 创建滚动视图。所有控件都应添加到父级 `options.scrollerFlow` 中。
- `options.title.set_text(...)` 设置页面标题。
- Haxe 的 `ref` 参数使用 `Ref<T>.In(value)` 或 `Ref<T>.Null` 传入。
- Haxe 字符串类型是 `dc.String`，C# 字符串可用 `AsHaxeString()`（位于 `ModCore.Utilities`）转换。
- 使用 `Config<T>` 持久化设置，并在回调中调用 `Save()`。
- `BuildMenu` 可能被多次调用（例如调整窗口大小或重新进入页面时）。每次都应重新构建控件，不要缓存控件实例。

## Widget 辅助方法

所有辅助方法都定义在 `dc.ui.OptionsBase` 上（`dc.ui.Options` 继承自它），并返回 `OptionWidget`。

| 方法 | C# 代理签名 |
| --- | --- |
| `addSimpleWidget` | `(dc.String title, dc.String? subText, HlAction onClick, Ref<int> icon, Flow parent)` |
| `addToggleWidget` | `(dc.String title, dc.String? subText, HlFunc<bool> onClick, Ref<bool> value, Flow parent)` |
| `addRadioWidget` | `(dc.String title, dc.String? subText, HlAction onClick, Ref<bool> selected, Flow parent)` |
| `addSliderWidget` | `(dc.String title, HlAction<double> onUpdate, double value, Ref<double> step, Flow parent, Ref<bool> showPercent, Ref<bool> showRawValue, Ref<double> min, Ref<double> max, dc.String? buttonText, Ref<int> padding)` |
| `addListWidget` | `(dc.String title, dc.String? subText, HlAction<int> onSelect, int selectedIndex, int count, ArrayObj texts, Ref<int> padding, Flow parent)` |
| `addSeparator` | `(dc.String label, Flow parent)` |

说明：

- `addSimpleWidget` 的 `icon` 是图标索引；常用 `5`，`0` 表示不显示图标。
- `addToggleWidget` 的回调需要切换值并返回**新状态**。
- `addRadioWidget` 表示单个单选按钮；如果需要互斥的一组选项，请自行管理组状态。
- `addSliderWidget` 的可选 `Ref` 参数较多；不需要的参数可传 `Ref<bool>.In(false)` / `Ref<int>.Null`。
- 还有其他辅助方法（`addKeyboardWidget`、`addHSVColorWidget`、`addRadioLangWidget` 等），使用方式类似。

## 使用 IModMenuProvider

如果希望把菜单放在单独的类中（例如该类还需要实现事件接口），可以使用 `IModMenuProvider`：

```csharp
using ModCore.Menu;
using ModCore.Mods;

public class MyMod( ModInfo info ) : ModBase(info), IModMenuProvider
{
    private readonly MyMenu menu = new();

    IModMenu IModMenuProvider.GetModMenu() => menu;
}

public class MyMenu : IModMenu
{
    string IModMenu.GetName() => "My Mod";
    string? IModMenu.GetSubText() => null;

    void IModMenu.BuildMenu( dc.ui.Options options )
    {
        options.title.set_text("MY MOD".AsHaxeString());
        options.createScroller(1);
        // 向 options.scrollerFlow 添加控件……
    }
}
```

如果单独的菜单类还需要接收事件，请让它实现 `ModCore.Events.IEventReceiver`，并在构造函数中调用 `EventSystem.AddReceiver(this)`。

## 打开另一个菜单

使用 `MenuModule.Instance.SetSection(menu)` 切换到另一个 `IModMenu`。常见做法是使用一个菜单类加一个页面/分类字段：

```csharp
public class MyMenu : IModMenu
{
    public MyPage Page { get; set; } = MyPage.Root;

    void IModMenu.BuildMenu( dc.ui.Options options )
    {
        if (Page == MyPage.Root)
        {
            options.title.set_text("MY MOD".AsHaxeString());
            options.createScroller(1);

            options.addSimpleWidget(
                "Open advanced page".AsHaxeString(),
                null,
                () =>
                {
                    Page = MyPage.Advanced;
                    MenuModule.Instance.SetSection(this);
                },
                Ref<int>.In(5),
                options.scrollerFlow);
        }
        else
        {
            options.title.set_text("ADVANCED".AsHaxeString());
            options.createScroller(1);
            // ...
        }
    }
}
```

如果需要返回 DCCM 核心主菜单，可以使用 `MenuModule.Instance.DCCMMainMenu` 获取核心主菜单的 `OptionsSection`。

## 添加暂停菜单按钮

实现 `IOnAfterPauseMenuBuild`，并调用 `MenuModule.Instance.AddCustomButtonToPause`。文本会经过 `GetText`，`index` 是相对于现有底部按钮的插入偏移。

```csharp
using dc.ui;
using ModCore.Events.Interfaces.Game.Menu;
using ModCore.Menu;
using ModCore.Mods;
using ModCore.Modules;

public class MyMod( ModInfo info ) : ModBase(info), IModMenuProvider, IOnAfterPauseMenuBuild
{
    private readonly MyMenu menu = new();

    IModMenu IModMenuProvider.GetModMenu() => menu;

    void IOnAfterPauseMenuBuild.OnAfterPauseMenuBuild( Pause pause )
    {
        MenuModule.Instance.AddCustomButtonToPause("My Menu", _ =>
        {
            pause.root.set_visible(false);
            var options = new Options(pause, null, null);
            MenuModule.Instance.SetSection(menu);
            options.killOnBack = true;
        }, pause, 2);
    }
}
```

隐藏 `pause.root`、创建 `new Options(pause, null, null)` 并设置 `killOnBack = true`，可以让自定义页面像普通暂停菜单子页面一样工作，并在返回时关闭。

## 本地化

- `GetName()` 和 `GetSubText()` 会经过 `GetText.Instance.GetString`。如果你提供 `.mo` 文件，可以返回翻译键并调用 `GetText.Instance.RegisterMod(...)` 注册；参见[本地化（GetText）](./gettext.md)。
- 控件的标题/描述**不会**自动翻译，需要自行包装：

```csharp
var label = GetText.Instance.GetString("Enable feature").AsHaxeString();
```

## API 参考

| API | 说明 |
| --- | --- |
| `IModMenu.GetName()` | 菜单入口标签 |
| `IModMenu.GetSubText()` | 可选的入口描述 |
| `IModMenu.BuildMenu(Options)` | 构建菜单页面 |
| `IModMenuProvider.GetModMenu()` | 在 `ModBase` 上实现时提供菜单实例 |
| `MenuModule.Instance.SetSection(IModMenu)` | 切换到自定义菜单分节 |
| `MenuModule.Instance.DCCMMainMenu` | DCCM 核心主菜单分节 |
| `MenuModule.Instance.AddCustomButtonToPause(text, onClick, pause, index)` | 向暂停菜单添加按钮 |
| `IOnAfterPauseMenuBuild.OnAfterPauseMenuBuild(Pause)` | 暂停菜单构建事件 |
| `Options.createScroller(double)` | 创建页面使用的滚动视图 |
| `Options.scrollerFlow` | 控件的父级 `Flow` |
| `Options.title` | 页面标题（`dc.ui.Text`） |
| `Options.onResize()` | 动态修改后重新布局页面 |

## 注意事项

- `BuildMenu` 在游戏主线程运行，避免在其中执行长时间阻塞操作。
- 进入分节时会重新构建菜单，不要跨构建缓存控件引用。
- `IModMenu` 只会从 `ModBase` 接收者中发现。独立菜单类必须通过 `IModMenuProvider` 返回；只把它加入事件系统并不会显示。
- Options 控件 API 与游戏自己的设置界面一致。如果不确定某个控件的行为，可以查看原版 `Options` 代码，或在开发构建中实际测试。
