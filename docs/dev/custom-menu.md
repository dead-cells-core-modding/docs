---
sidebar_position: 8
---

# Custom Menu

DCCM adds a **Core Modding** entry to the game's Options menu. Mods can add their own settings page under that entry and build it with the game's native `dc.ui.Options` widget API, so the page looks and behaves like a vanilla options screen.

## How a Mod Menu Is Discovered

`MenuModule` scans every loaded `ModBase` receiver when the Core Modding page is built. A mod can expose a menu in two ways:

- implement `ModCore.Menu.IModMenu` directly on the `ModBase` class, or
- implement `IModMenuProvider` and return an `IModMenu` instance from `GetModMenu()`.

No manual registration call is needed. The menu entry appears at **Options → Core Modding → (your menu name)**.

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

| Member | Purpose |
| --- | --- |
| `GetName()` | Entry label. DCCM passes it through `GetText.Instance.GetString`, so it can be a translation key. |
| `GetSubText()` | Optional description shown under the entry. May return `null`. |
| `BuildMenu(options)` | Builds the page. Called every time the section is opened; build it from scratch. |

## Minimal Example

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
            Ref<double>.In(1.0),   // step
            flow,
            Ref<bool>.In(false),   // show percent
            Ref<bool>.In(true),    // show raw value
            Ref<double>.In(0.1),   // min
            Ref<double>.In(10.0),  // max
            null,                  // button text
            Ref<int>.Null);        // padding

        options.addSeparator("".AsHaxeString(), flow);
    }
}
```

Key points:

- `options.createScroller(1)` creates the scroll view. Use `options.scrollerFlow` as the parent flow for all widgets.
- `options.title.set_text(...)` sets the page title.
- Haxe `ref` parameters are passed as `Ref<T>.In(value)` or `Ref<T>.Null`.
- Haxe strings use `dc.String`; convert C# strings with `AsHaxeString()` (from `ModCore.Utilities`).
- Persist settings with `Config<T>` and call `Save()` in the callback.
- `BuildMenu` can be called multiple times (for example after resizing or re-entering the page). Rebuild the widgets each time instead of caching them.

## Widget Helpers

All helpers are methods on `dc.ui.OptionsBase` (inherited by `dc.ui.Options`) and return an `OptionWidget`.

| Method | C# proxy signature |
| --- | --- |
| `addSimpleWidget` | `(dc.String title, dc.String? subText, HlAction onClick, Ref<int> icon, Flow parent)` |
| `addToggleWidget` | `(dc.String title, dc.String? subText, HlFunc<bool> onClick, Ref<bool> value, Flow parent)` |
| `addRadioWidget` | `(dc.String title, dc.String? subText, HlAction onClick, Ref<bool> selected, Flow parent)` |
| `addSliderWidget` | `(dc.String title, HlAction<double> onUpdate, double value, Ref<double> step, Flow parent, Ref<bool> showPercent, Ref<bool> showRawValue, Ref<double> min, Ref<double> max, dc.String? buttonText, Ref<int> padding)` |
| `addListWidget` | `(dc.String title, dc.String? subText, HlAction<int> onSelect, int selectedIndex, int count, ArrayObj texts, Ref<int> padding, Flow parent)` |
| `addSeparator` | `(dc.String label, Flow parent)` |

Notes:

- The `icon` parameter of `addSimpleWidget` is the icon index; `5` is commonly used and `0` means no icon.
- `addToggleWidget` expects the callback to toggle the value and return the **new** state.
- `addRadioWidget` is a single radio button; manage the group state yourself if you need mutually exclusive options.
- `addSliderWidget` has many optional `Ref` parameters; pass `Ref<bool>.In(false)` / `Ref<int>.Null` for the ones you do not need.
- More helpers exist (`addKeyboardWidget`, `addHSVColorWidget`, `addRadioLangWidget`, etc.); they follow the same pattern.

## Using IModMenuProvider

If you want to keep the menu in a separate class (for example because it also implements event interfaces), use `IModMenuProvider`:

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
        // Add widgets to options.scrollerFlow ...
    }
}
```

If the separate menu class also needs to receive events, make it implement `ModCore.Events.IEventReceiver` and call `EventSystem.AddReceiver(this)` in its constructor.

## Opening Another Menu

Use `MenuModule.Instance.SetSection(menu)` to switch to another `IModMenu`. A common pattern is a single menu class with a page/category field:

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

`MenuModule.Instance.DCCMMainMenu` returns the OptionsSection of the DCCM core main menu if you need to navigate back to the core page.

## Adding a Pause Menu Button

Implement `IOnAfterPauseMenuBuild` and call `MenuModule.Instance.AddCustomButtonToPause`. The text is passed through `GetText`, and `index` is the insertion offset relative to the existing bottom buttons.

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

Hiding `pause.root`, creating `new Options(pause, null, null)`, and setting `killOnBack = true` lets the custom page behave like a normal pause-menu sub-page and close on Back.

## Localization

- `GetName()` and `GetSubText()` are passed through `GetText.Instance.GetString`. Return a translation key and register it with `GetText.Instance.RegisterMod(...)` if you ship `.mo` files; see [Localization (GetText)](./gettext.md).
- Widget titles/subtexts are **not** translated automatically. Wrap them yourself:

```csharp
var label = GetText.Instance.GetString("Enable feature").AsHaxeString();
```

## API Reference

| API | Description |
| --- | --- |
| `IModMenu.GetName()` | Menu entry label |
| `IModMenu.GetSubText()` | Optional entry description |
| `IModMenu.BuildMenu(Options)` | Builds the menu page |
| `IModMenuProvider.GetModMenu()` | Supplies the menu when implemented on a `ModBase` |
| `MenuModule.Instance.SetSection(IModMenu)` | Switches to a custom menu section |
| `MenuModule.Instance.DCCMMainMenu` | DCCM core main-menu section |
| `MenuModule.Instance.AddCustomButtonToPause(text, onClick, pause, index)` | Adds a button to the pause menu |
| `IOnAfterPauseMenuBuild.OnAfterPauseMenuBuild(Pause)` | Pause menu build event |
| `Options.createScroller(double)` | Creates the scroll view used by the page |
| `Options.scrollerFlow` | Parent `Flow` for widgets |
| `Options.title` | Page title (`dc.ui.Text`) |
| `Options.onResize()` | Re-layouts the page after dynamic changes |

## Notes

- `BuildMenu` runs on the game main thread. Avoid long blocking work there.
- The menu is rebuilt when the section is entered; do not keep references to widgets across builds.
- `IModMenu` is discovered only from `ModBase` receivers. A standalone menu class must be returned by `IModMenuProvider`; just adding it to the event system is not enough.
- The Options widget API mirrors the game's own settings screen. If you are unsure about a widget's behavior, inspect the vanilla `Options` code or try it in a development build.
