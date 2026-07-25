---
sidebar_position: 7
---

# Localization (GetText)

The `GetText` module loads a mod's `.mo` translation files and merges them into the game's built-in `Lang.Class.t.get()` localization chain, enabling mod UI text, tooltips, and other strings to support multi-language switching.

## Overview

The game uses the GNU gettext `.mo` binary format to store translation text. DCCM intercepts the game's `.mo` loading process, and on each language switch, loads registered mods' translation files in priority order, merging them into the game's global `texts` table. Developers simply call `GetText.Instance.GetString("key")` to retrieve the translated string for the current language.

:::info
`GetText` is a CoreModule with priority `ModulePriorities.Game`. It completes its own initialization during the `IOnAdvancedModuleInitializing` stage and registers the `dccm-core` namespace.
:::

## `.mo` File Lookup Paths

When packaging `.mo` files, mod names must follow a fixed pattern. After calling `RegisterMod(name)`, DCCM attempts to load files in the following order:

| Priority | Path Pattern | Description |
| --- | --- | --- |
| 1 | `{name}/lang/main.{lang}.mo` | Namespaced, filename `main`, current language |
| 2 | `{name}/lang/{name}.{lang}.mo` | Namespaced, filename matches namespace, current language |
| 3 | `lang/{name}.{lang}.mo` | No namespace, current language |
| 4 | `{name}/lang/main.en.mo` | Namespaced, filename `main`, English (fallback) |
| 5 | `{name}/lang/{name}.en.mo` | Namespaced, filename matches namespace, English (fallback) |
| 6 | `lang/{name}.en.mo` | No namespace, English (fallback) |

Where `{lang}` is the current game language code (e.g. `zh`, `en`, `fr`), and `{name}` is the namespace passed to `RegisterMod()`.

The 6 paths are tried one by one, stopping at the first file found. The first 3 match the current language; the last 3 fall back to English. This means: if your mod only provides English translations, placing the file in any location matching paths 4 through 6 works. If you provide multiple languages, DCCM prioritizes loading the file for the current language.

## Registration and Loading

Call `RegisterMod()` during mod initialization to register the namespace. On each language switch (when `readMo` is invoked), DCCM automatically iterates over all registered names and loads the corresponding `.mo` files:

```csharp
using ModCore.Events.Interfaces;
using ModCore.Modules;
using ModCore.Utilities;

internal class MyMod(ModInfo info) : ModBase(info), IOnAfterLoadingAssets
{
    public override void Initialize()
    {
        // Register namespace - DCCM auto-loads corresponding .mo files on language switch
        GetText.Instance.RegisterMod("DeadCellsMultiplayerX");
    }

    void IOnAfterLoadingAssets.OnAfterLoadingAssets()
    {
        var res = Info.ModRoot!.GetFilePath("res.pak");
        FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
    }
}
```

After loading completes, DCCM broadcasts the `IOnLoadingLanguage` event with the language code string as the parameter. Mods can listen for this event to execute custom logic after a language switch (e.g. refreshing UI):

```csharp
public class MyMod(ModInfo info) : ModBase(info), IOnLoadingLanguage
{
    void IOnLoadingLanguage.OnLoadingLanguage(string lang)
    {
        // lang is a language code such as "zh", "en", etc.
        Logger.Information("Language switched to: {lang}", lang);
    }
}
```

## Retrieving Strings

The module provides two overloads:

```csharp
// Simple translation
string GetString(string key)

// Translation with parameter formatting
string GetString(string key, IDictionary<string, string> @params)
```

Projects commonly wrap a short helper method to simplify calls:

```csharp
private static string T(string key) => GetText.Instance.GetString(key);
```

Usage examples (from DeadCellsMultiplayerX's `ClientMain.cs`):

```csharp
using ModCore.Modules;

// Wrap helper method
private static string T(string key) => GetText.Instance.GetString(key);

// Use in UI construction
lobby!.BuildMenuChild("Online", () => lobby.Show(), color: color);

// With parameters:
var text = GetText.Instance.GetString("welcome_message",
    new Dictionary<string, string>
    {
        { "player", playerName },
        { "version", "1.0" }
    });
```

## Full Workflow

Summary of the complete steps from creating translations to runtime usage:

1. **Create `.mo` file**: Use `msgfmt` or a PO editing tool (e.g. Poedit) to compile `.po` into `.mo`, following the path conventions (e.g. `DeadCellsMultiplayerX/lang/main.zh.mo`)
2. **Pack**: Include `.mo` files in `res.pak` via `PackAssets` in `.csproj`. See [Asset Packaging](../resources.md#packaging-resources)
3. **Load**: Implement `IOnAfterLoadingAssets` and call `FsPak.Instance.FileSystem.loadPak()` to load `res.pak`
4. **Register**: Call `GetText.Instance.RegisterMod("namespace")` in `Initialize()`
5. **Use**: Retrieve translated strings via `GetText.Instance.GetString("key")`

## API Quick Reference

| API | Description |
| --- | --- |
| `GetText.Instance.RegisterMod(name)` | Registers a mod namespace; DCCM auto-loads `.mo` files under that namespace on language switch |
| `GetText.Instance.GetString(key)` | Retrieves the translated string for the current language (returns the key itself if not found) |
| `GetText.Instance.GetString(key, params)` | Retrieves the translated string and replaces placeholders; `params` is `IDictionary<string, string>` |
| `IOnLoadingLanguage.OnLoadingLanguage(lang)` | Language loaded/switched event; `lang` parameter is the language code |
