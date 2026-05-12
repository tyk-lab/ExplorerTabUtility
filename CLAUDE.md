# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Explorer Tab Utility** — Windows 11 system utility that automatically converts new File Explorer windows into tabs. Uses system-level hooks, COM interop with Windows Shell, and WPF for UI.

## Build Commands

```powershell
# Debug build
dotnet build ExplorerTabUtility/ExplorerTabUtility.csproj -c Debug

# Release build
dotnet build ExplorerTabUtility/ExplorerTabUtility.csproj -c Release

# Run (after build)
./ExplorerTabUtility/bin/Debug/net9.0-windows/ExplorerTabUtility.exe

# Publish for specific target (example: net9.0, x64)
msbuild ExplorerTabUtility/ExplorerTabUtility.csproj /restore /t:Publish `
  /p:Configuration=Release `
  /p:TargetFramework=net9.0-windows `
  /p:RuntimeIdentifier=win-x64 `
  /p:PublishDir=../publish/net9.0-windows/x64
```

**Targets:** `net9.0-windows` and `net481` × `win-x64 / win-x86 / win-arm64`

## Testing

No test project exists. All verification is manual (requires Windows 11 + Explorer). CI is GitHub Actions triggered by `v*` tags.

## Architecture

### Component Map

```
App.xaml.cs
└── MainWindow (WPF host)
    ├── HookManager          ← orchestrates all hooks; routes hotkey → action
    │   ├── ExplorerWatcher  ← window-level hook; COM tab manipulation
    │   ├── Keyboard         ← global keyboard hook (H.Hooks)
    │   └── Mouse            ← global mouse hook
    ├── ProfileManager       ← hotkey profiles CRUD, JSON persistence
    ├── SettingsManager      ← app settings → %APPDATA%\ExplorerTabUtility\settings.json
    └── UI/Views/            ← WPF views (SystemTrayIcon, TabSearchPopup, HotKeyProfileControl)
```

### Key Design Constraints

- **COM threading (STA):** All Shell32/SHDocVw COM calls must run on a `StaTaskScheduler`. Never call COM from a thread-pool thread directly.
- **Thread safety:** Window handle tracking uses `ConcurrentDictionary` and explicit locks (`_windowEntryDictLock`, `_closedWindowsLock`). Maintain this pattern when adding shared state to `ExplorerWatcher`.
- **Dual-framework:** Code must compile for both `net9.0-windows` and `net481`. Avoid APIs unavailable on .NET Framework 4.8.1 unless guarded by `#if NET`.

### Adding a New Hotkey Action

1. Add enum value to `Models/HotKeyAction.cs`
2. Implement handler branch in `HookManager.OnHotKeyProfileTriggered`
3. Expose UI option in `UI/Views/HotKeyProfileControl.xaml`

### Settings Persistence

`SettingsManager` (static class) loads/saves `AppSettings` as JSON on startup/change. Settings file: `%APPDATA%\ExplorerTabUtility\settings.json`. Hotkey profiles are serialized as part of `AppSettings` via `HotKeyActionJsonConverter`.

### COM Interop Entry Points

- `ExplorerWatcher` — uses `ShellWindows` COM collection to enumerate/monitor Explorer instances
- `Interop/` — raw COM interface definitions (`IShellBrowser`, `IShellFolder`, `IAccessible`)
- `WinAPI/WinApi.cs` — all P/Invoke declarations

## Known Limitations / 已知限制

### 重复打开已有 Tab 的文件夹时无法自动跳转

**场景：** 当某文件夹已作为 Explorer tab 打开时，再次通过以下方式打开同一文件夹，Explorer 只会"闪烁"激活窗口，但不会切换到对应 tab：
- 第三方桌面整理软件（如腾讯桌面整理）中双击文件夹图标
- 外部应用（如 Obsidian）右键"用资源管理器打开"

**根本原因：** Explorer 在检测到目标路径已在某 tab 中打开时，仅做 `SetForegroundWindow`，不触发任何可被外部观察的事件：
- `ShellWindows.WindowRegistered` — 不触发（无新窗口）
- `NavigateComplete2` — 不触发（无导航发生）
- 所有 WinEvent (`EVENT_OBJECT_STATECHANGE` / `EVENT_SYSTEM_FOREGROUND` 等) — 均不触发

通过 `SetWinEventHook` 诊断已确认：整个"已有路径激活"流程对外完全不可观察。

**原生 Windows 桌面双击有效：** 通过 `AccessibleObjectFromPoint`（IAccessible MSAA）可从点击坐标拿到图标名，进而解析路径并切换 tab。但此方案对第三方桌面整理软件无效（这类软件不暴露 IAccessible 子项，`accHitTest` 返回 0）。

**若要彻底解决通用情况（任意外部应用），唯一可行路线是：**
- API Hook（Detours 风格拦截 `ShellExecuteExW` / `SHOpenFolderAndSelectItems`）—— 需注入，触发杀毒
- 暂时没有非侵入性的通用方案

**当前可用替代方案：** TabSearch 热键（`HotKeyAction.TabSearch`）—— 用户手动搜索并切换 tab。

---

## Release Process

Tag `v*` → GitHub Actions builds 6 artifacts → SignPath signing → Inno Setup installer → GitHub Release. Also published to winget (`w4po.ExplorerTabUtility`) and Chocolatey (`explorertabutility`).
