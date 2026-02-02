# UniWindowController (UniWinC) - Complete Reference

> **Package:** `com.kirurobo.uniwinc` v0.9.4
> **License:** MIT
> **Namespace:** `Kirurobo`
> **Platforms:** Windows 10/11 (x86, x64), macOS (Standalone, Editor)
> **Unity:** 2020.3+ (.NET 4.x runtime)
> **Native Plugin:** `LibUniWinC.dll` (Windows), `LibUniWinC.bundle` (macOS/Swift)

---

## What This Package Does

UniWinC is a Unity library that controls the application's own OS window. It enables:

- Making the window **transparent** (non-rectangular appearance)
- Making the window **borderless**
- Setting the window as **always-on-top** or **always-on-bottom**
- **Click-through** on transparent pixels (mouse events pass to windows behind)
- **Drag-to-move** the window via UI elements
- **File/folder drag-and-drop** acceptance
- **Native file open/save dialogs** (cross-platform)
- **Window position, size, and maximize** control
- **Multi-monitor** support (detect, query, fit-to-monitor)

Transparency only works in **standalone builds**, not the Unity Editor.

---

## Repository Structure

```
UniWinC/Assets/Kirurobo/UniWindowController/
  Runtime/
    Scripts/
      UniWindowController.cs        # Main MonoBehaviour controller (singleton)
      UniWindowMoveHandle.cs         # Drag-to-move UI component
      LowLevel/
        UniWinCore.cs                # P/Invoke wrapper for native plugin (internal)
        FilePanel.cs                 # Native file dialog API
    Prefabs/
      UniWindowController.prefab     # Drop-in controller prefab
      DragMoveCanvas.prefab          # Full-screen drag-move overlay
    Plugins/
      Windows/x64/LibUniWinC.dll
      Windows/x86/LibUniWinC.dll
      MacOS/LibUniWinC.bundle
  Editor/
    Scripts/
      UniWindowControllerEditor.cs   # Custom inspector, Player Settings validation
      UniWindowControllerBatch.cs    # Build automation helper
  Samples/
    00_Menu/       # Sample scene loader and manager
    01_SimpleSample/
    02_UiSample/   # Full-featured demo with all controls
    03_Fullscreen/
    04_FileDialog/
    Common/        # Shared utilities (AutoRotator, ModelController)

VisualStudio/      # Windows DLL source (C++)
Xcode/             # macOS bundle source (Swift)
```

### Assembly Definitions

| Assembly | Scope | Platforms |
|:---------|:------|:----------|
| `Unity.UniWindowController` | Runtime | Editor, macOSStandalone, WindowsStandalone32, WindowsStandalone64 |
| `Unity.UniWindowController.Editor` | Editor only | Editor |

---

## Installation

**Via UPM (Unity Package Manager):**
1. Window > Package Manager > "+" > Add package from git URL
2. Enter: `https://github.com/kirurobo/UniWindowController.git#upm`

**Via .unitypackage:**
1. Download from [Releases](https://github.com/kirurobo/UniWindowController/releases)
2. Import into Unity

---

## Basic Setup

1. Add the `UniWindowController` prefab to your scene
2. Fix Player Settings via the inspector's validation button, or manually set:
   - **Run in Background:** true (required for click-through toggle)
   - **Resizable Window:** true
   - **Fullscreen Mode:** Windowed
   - **Allow Fullscreen Switch:** false
   - **Use DXGI Flip Mode Swapchain:** false (Windows, required for transparency)
3. Optionally add the `DragMoveCanvas` prefab for drag-to-move
   - Requires an `EventSystem` in the scene
4. Build as PC/Mac Standalone

---

## API Reference

### UniWindowController (MonoBehaviour, Singleton)

Access the singleton instance via:
```csharp
UniWindowController uwc = UniWindowController.current;
```

#### Properties

| Property | Type | Default | Description |
|:---------|:-----|:--------|:------------|
| `isTransparent` | `bool` | `false` | Toggle transparent (non-rectangular) window. Only works in builds. |
| `alphaValue` | `float` | `1.0` | Window opacity (0.0 fully transparent to 1.0 fully opaque). |
| `isTopmost` | `bool` | `false` | Always-on-top. Mutually exclusive with `isBottommost`. |
| `isBottommost` | `bool` | `false` | Always-on-bottom. Mutually exclusive with `isTopmost`. |
| `isZoomed` | `bool` | `false` | Maximize/restore the window (called "zoomed" on macOS). |
| `isClickThrough` | `bool` | `false` | Pass mouse events through to windows behind. |
| `isHitTestEnabled` | `bool` | `true` | Enable automatic click-through based on `hitTestType`. When false, manually control `isClickThrough`. |
| `hitTestType` | `HitTestType` | `Opacity` | Automatic click-through detection method. |
| `opacityThreshold` | `float` | `0.1` | Alpha threshold for opacity-based hit test (0.0-1.0). |
| `allowDropFiles` | `bool` | `false` | Accept file/folder drag-and-drop onto window. |
| `shouldFitMonitor` | `bool` | `false` | Fit (maximize) the window to a specific monitor. |
| `monitorToFit` | `int` | `0` | Target monitor index when `shouldFitMonitor` is true. |
| `windowPosition` | `Vector2` | — | Get/set window position. Origin is lower-left of main monitor, Y-up. |
| `windowSize` | `Vector2` | — | Get/set window size (including frame). |
| `clientSize` | `Vector2` | — | Get client area size (read-only). |
| `cursorPosition` | `Vector2` | — | Get/set mouse cursor position in screen coordinates. |
| `pickedColor` | `Color` | — | Color under cursor when opacity hit test is active (read-only). |
| `transparentType` | `TransparentType` | `Alpha` | Transparency method (Windows only). |
| `keyColor` | `Color32` | `(1,0,1,0)` | Key color for ColorKey transparency (Windows only). |
| `autoSwitchCameraBackground` | `bool` | `true` | Auto-switch camera to transparent background when transparent. |
| `forceWindowed` | `bool` | `false` | Force windowed mode on startup if launched fullscreen. |
| `currentCamera` | `Camera` | `null` | Camera to manage. Uses `Camera.main` if null. |

#### Methods

| Method | Returns | Description |
|:-------|:--------|:------------|
| `SetTransparentType(TransparentType type)` | `void` | Change transparency method at runtime (Windows only). |
| `SetCamera(Camera newCamera)` | `void` | Change the managed camera, restoring previous camera's background. |
| `GetMonitorCount()` | `int` | Static. Get number of connected monitors. |
| `GetMonitorRect(int index)` | `Rect` | Static. Get monitor position and size. |
| `GetCursorPosition()` | `Vector2` | Static. Get mouse cursor screen position. |
| `SetCursorPosition(Vector2 pos)` | `void` | Static. Set mouse cursor screen position. |

#### Enums

**`TransparentType`** (Windows only)
| Value | Description |
|:------|:------------|
| `None` | No transparency |
| `Alpha` | Standard alpha-based transparency (default) |
| `ColorKey` | Single-color transparency via layered window. No semi-transparency but better touch support. |

**`HitTestType`**
| Value | Description |
|:------|:------------|
| `None` | No automatic hit test. Manually control `isClickThrough`. |
| `Opacity` | Check pixel alpha under cursor. Natural but higher CPU cost. |
| `Raycast` | Check colliders and UI raycasts. Lower CPU cost, requires colliders. Respects "Ignore Raycast" layer. |

**`WindowStateEventType`** (Flags enum)
| Value | Description |
|:------|:------------|
| `StyleChanged` | Window style was modified |
| `Resized` | Window size changed |
| `TopMostEnabled` / `TopMostDisabled` | Topmost state changed |
| `BottomMostEnabled` / `BottomMostDisabled` | Bottommost state changed |
| `WallpaperModeEnabled` / `WallpaperModeDisabled` | Wallpaper mode toggled |

#### Events

| Event | Signature | Description |
|:------|:----------|:------------|
| `OnStateChanged` | `OnStateChangedDelegate(WindowStateEventType type)` | Window style, maximize, topmost, etc. changed. |
| `OnDropFiles` | `FilesDelegate(string[] files)` | Files or folders were dropped on the window. |
| `OnMonitorChanged` | `OnMonitorChangedDelegate()` | Monitor configuration or resolution changed. |

---

### UniWindowMoveHandle (MonoBehaviour)

Attach to any UI element (must be a Raycast Target) to enable drag-to-move of the window.

| Property | Type | Default | Description |
|:---------|:-----|:--------|:------------|
| `disableOnZoomed` | `bool` | `true` | Disable drag-move when the window is maximized or fit-to-monitor. |
| `IsDragging` | `bool` | — | Whether a drag is in progress (read-only). |

**Behavior:**
- Only responds to left mouse button drag
- Ignores drag when Shift, Ctrl, or Alt is held
- Automatically disables hit test during drag and restores it after
- On macOS uses native cursor position; on Windows uses `eventData.position` for touch compatibility
- Implements `IDragHandler`, `IBeginDragHandler`, `IEndDragHandler`, `IPointerUpHandler`

**DragMoveCanvas prefab pattern:**
- A full-screen Panel on the "Ignore Raycast" layer with `UniWindowMoveHandle` attached
- Since it's on "Ignore Raycast", the automatic hit test ignores it
- Set a lower Sort Order so other UI elements receive events first

---

### FilePanel (Static Class)

Native cross-platform file open/save dialogs.

#### Opening Files

```csharp
FilePanel.Settings settings = new FilePanel.Settings
{
    title = "Open Image",
    filters = new FilePanel.Filter[]
    {
        new FilePanel.Filter("Image files", "png", "jpg", "jpeg"),
        new FilePanel.Filter("All files", "*"),
    },
    flags = FilePanel.Flag.AllowMultipleSelection,
    initialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyPictures),
    initialFile = "photo.png",
};

FilePanel.OpenFilePanel(settings, (string[] files) =>
{
    foreach (var file in files)
        Debug.Log("Selected: " + file);
});
```

#### Saving Files

```csharp
FilePanel.Settings settings = new FilePanel.Settings
{
    title = "Save File",
    filters = new FilePanel.Filter[]
    {
        new FilePanel.Filter("Text files", "txt", "log"),
        new FilePanel.Filter("All files", "*"),
    },
    initialFile = "output.txt",
    initialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
};

FilePanel.SaveFilePanel(settings, (string[] files) =>
{
    Debug.Log("Save to: " + files[0]);
});
```

#### FilePanel.Settings

| Field | Type | Description |
|:------|:-----|:------------|
| `title` | `string` | Dialog title |
| `filters` | `Filter[]` | File type filters. `Filter("Label", "ext1", "ext2", ...)` |
| `initialDirectory` | `string` | Starting directory |
| `initialFile` | `string` | Default filename |
| `defaultExtension` | `string` | Not implemented yet |
| `flags` | `Flag` | Bitfield flags (see below) |

#### FilePanel.Flag

| Flag | Value | Description |
|:-----|:------|:------------|
| `None` | 0 | Default |
| `FileMustExist` | 1 | Windows only |
| `FolderMustExist` | 2 | Windows only |
| `AllowMultipleSelection` | 4 | Allow selecting multiple files |
| `CanCreateDirectories` | 16 | Allow creating new directories |
| `OverwritePrompt` | 256 | Warn on overwrite (always enabled on macOS) |
| `CreatePrompt` | 512 | Confirm file creation (always enabled on macOS) |
| `ShowHiddenFiles` | 4096 | Show hidden files |
| `RetrieveLink` | 8192 | Resolve shortcuts/symlinks |

---

### Custom Attributes

| Attribute | Usage | Description |
|:----------|:------|:------------|
| `[EditableProperty]` | Field | Makes serialized field editable at runtime in Inspector via reflection to its property. Field name must start with `_` and a matching public property must exist. |
| `[ReadOnly]` | Field | Displays the field as read-only in the Inspector. |

---

## Scripting Examples

### Basic Transparent Window

```csharp
using Kirurobo;

void Start()
{
    var uwc = UniWindowController.current;
    uwc.isTransparent = true;
    uwc.isTopmost = true;
}
```

### Toggle Transparency with Keyboard

```csharp
void Update()
{
    var uwc = UniWindowController.current;

    if (Input.GetKeyUp(KeyCode.T))
        uwc.isTransparent = !uwc.isTransparent;

    if (Input.GetKeyUp(KeyCode.F))
        uwc.isTopmost = !uwc.isTopmost;

    if (Input.GetKeyUp(KeyCode.B))
        uwc.isBottommost = !uwc.isBottommost;

    if (Input.GetKeyUp(KeyCode.Z))
        uwc.isZoomed = !uwc.isZoomed;
}
```

### Listen for Dropped Files

```csharp
void Start()
{
    var uwc = UniWindowController.current;
    uwc.allowDropFiles = true;
    uwc.OnDropFiles += (files) =>
    {
        foreach (var path in files)
            Debug.Log("Dropped: " + path);
    };
}
```

### Resize Window Programmatically

```csharp
var uwc = UniWindowController.current;
uwc.windowSize += new Vector2(100, 0);       // widen by 100px
uwc.windowPosition = new Vector2(100, 200);  // move window
```

### Fit to Monitor

```csharp
var uwc = UniWindowController.current;
int monitorCount = UniWindowController.GetMonitorCount();
if (monitorCount > 1)
{
    uwc.monitorToFit = 1;          // target second monitor
    uwc.shouldFitMonitor = true;   // triggers move + maximize
}
```

### Listen for State and Monitor Changes

```csharp
var uwc = UniWindowController.current;
uwc.OnStateChanged += (type) =>
{
    Debug.Log("Window state changed: " + type);
};
uwc.OnMonitorChanged += () =>
{
    Debug.Log("Monitor configuration changed. Count: " + UniWindowController.GetMonitorCount());
};
```

### Change Camera at Runtime

```csharp
var uwc = UniWindowController.current;
uwc.SetCamera(myNewCamera);  // restores old camera background, applies settings to new one
```

---

## Input System Compatibility

The package supports both input systems via preprocessor directives:
- `ENABLE_INPUT_SYSTEM` - New Input System (UnityEngine.InputSystem)
- `ENABLE_LEGACY_INPUT_MANAGER` - Legacy Input Manager

These are auto-defined by Unity based on the project's Active Input Handling setting (Player Settings > Other Settings).

---

## Platform-Specific Notes

### Windows
- Two transparency types: `Alpha` (default, standard) and `ColorKey` (better for touch, no semi-transparency)
- `ColorKey` mode uses a layered window; OS handles hit testing, so `hitTestType` setting is ignored
- `SetAlphaValue` does not work in the Windows Editor (display stops updating)
- DXGI Flip Mode Swapchain must be disabled for transparency to work

### macOS
- Only `Alpha` transparency type is available
- Window uses `NSWindow.Level.popUpMenu` for topmost (above menu bar)
- Save dialog does not support file type dropdown (causes crashes)
- Retina display causes coordinate system scaling differences; `UniWindowMoveHandle` compensates
- `.bundle` is built with Swift via Xcode

### Unity Editor
- Transparency does **not** work in the Editor
- Click-through is disabled in the Editor (would make it unoperable)
- Topmost, window position/size, file drop, and file dialogs do work
- The Editor re-acquires the game view window on focus change; avoid closing/re-docking game view during play

---

## Required Player Settings

| Setting | Value | Reason |
|:--------|:------|:-------|
| Run in Background | true | Needed for click-through state to update |
| Resizable Window | true | Window frame removal changes size |
| Fullscreen Mode | Windowed | Transparency requires windowed mode |
| Allow Fullscreen Switch | false | Prevents accidental fullscreen |
| Use DXGI Flip Mode Swapchain | false | Windows only; required for transparency |

The custom Inspector provides a one-click "Fix all" button and per-setting "Fix" buttons.

---

## Limitations

- Transparency only works in standalone builds, not the Unity Editor
- Single window only (no multi-window support)
- Touch operation with transparency can feel unnatural (cannot see pixel under finger)
  - Workaround on Windows: use `ColorKey` transparency type (loses semi-transparency)
- `ColorKey` transparency has no semi-transparency and reduced visual quality
- macOS fullscreen edge cases may exist
- `defaultExtension` in `FilePanel.Settings` is not yet implemented
- API is not finalized and may change in future versions

---

## Changelog Highlights

| Version | Date | Changes |
|:--------|:-----|:--------|
| v0.9.4 | 2025-02-06 | New Input System support; camera clear flags fix; macOS save dialog fix; macOS borderless crash fix |
| v0.9.3 | 2024-05-06 | macOS bundle rewritten in Swift |
| v0.9.2 | 2023-09-18 | Apple Silicon Editor fix; client size display in UI sample |
| v0.9.1 | 2023-05-03 | macOS file type position fix; macOS GetClientSize fix |
| v0.9.0 | 2023-04-22 | Dev environment updated to Unity 2020.3.43; Windows frame size fix |
| v0.8.6 | 2022-06-18 | macOS window shadow fix |
| v0.8.4 | 2021-11-27 | Singleton pattern; UPM samples bundled; macOS file type selection |
| v0.8.2 | 2021-10-15 | FilePanel.OpenFilePanel/SaveFilePanel added; M1 support (untested) |
| v0.8.0 | 2020-12-27 | Fullscreen demo; bottommost support (experimental) |
| v0.7.0 | 2020-12-07 | UPM support |
| v0.6.0 | 2020-12-06 | File dropping; fit-to-monitor; "Maximized" renamed to "Zoomed" |
