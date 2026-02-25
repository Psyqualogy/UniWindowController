# Desktop Pet: Bottommost + Minimize Prevention + Taskbar Hiding

> **Implementation guide** for a UniWinC desktop pet that:
> 1. Stays below all windows (desktop level) using **Bottommost Mode**
> 2. Survives Win+D / Win+M using **WM_SYSCOMMAND minimize prevention**
> 3. Hides from taskbar and Alt+Tab using **WS_EX_TOOLWINDOW**
>
> **Platform:** Windows 10/11 only
> **Requires:** Rebuilding `LibUniWinC.dll`

---

## Table of Contents

1. [Step 1: Bottommost Mode (Already Working)](#step-1-bottommost-mode)
2. [Step 2: Minimize Prevention (DLL Modification)](#step-2-minimize-prevention)
3. [Step 3: Hide from Taskbar/Alt+Tab (DLL Modification)](#step-3-hide-from-taskbaralt-tab)
4. [Step 4: C# Bindings](#step-4-c-bindings)
5. [Step 5: Unity Usage](#step-5-unity-usage)
6. [Step 6: Build & Test](#step-6-build--test)
7. [Edge Cases & Troubleshooting](#edge-cases--troubleshooting)

---

## Step 1: Bottommost Mode

**No changes needed.** This is fully implemented and exposed.

### C# Usage

```csharp
var uwc = UniWindowController.current;
uwc.isTransparent = true;
uwc.isBottommost = true;
```

### How It Works

The native DLL does two things:

1. **Initial placement** — `SetBottommost()` (`libuniwinc.cpp:856-876`) calls `SetWindowPos` with `HWND_BOTTOM`:

   ```cpp
   SetWindowPos(
       hTargetWnd_,
       HWND_BOTTOM,
       0, 0, 0, 0,
       SWP_NOSIZE | SWP_NOMOVE | SWP_NOOWNERZORDER | SWP_NOACTIVATE
   );
   ```

2. **Continuous enforcement** — The subclassed WndProc (`libuniwinc.cpp:1423-1428`) intercepts every `WM_WINDOWPOSCHANGING` and forces `HWND_BOTTOM`:

   ```cpp
   case WM_WINDOWPOSCHANGING:
       if (bIsBottommost_) {
           ((WINDOWPOS*)lParam)->hwndInsertAfter = HWND_BOTTOM;
       }
       break;
   ```

This means any attempt to raise the window (focus, click, another app activating) is overridden. The pet stays at the bottom.

### What This Doesn't Solve

- Win+D ("Show Desktop") minimizes the window — pet disappears
- Win+M ("Minimize All") minimizes the window — pet disappears
- The pet appears in the taskbar and Alt+Tab

Steps 2 and 3 solve these.

---

## Step 2: Minimize Prevention

Intercept `WM_SYSCOMMAND` with `SC_MINIMIZE` in the DLL's WndProc to block all minimize attempts.

### 2a. Add the Global State Variable

**File:** `VisualStudio/LibUniWinC/libuniwinc.cpp`

Add near the other state variables (around line 22, after `static BOOL bIsBottommost_`):

```cpp
static BOOL bPreventMinimize_ = FALSE;
```

### 2b. Add the WM_SYSCOMMAND Handler

**File:** `VisualStudio/LibUniWinC/libuniwinc.cpp`

In `customWindowProcedure()` (line 1400), add a new case **before** the `default:` case (line 1451).

**Important:** This case must `return 0` directly — not `break` — to prevent the message from reaching `CallWindowProc`. If you `break`, the minimize still happens.

```cpp
    case WM_WINDOWPOSCHANGING:
        // 常に最背面
        if (bIsBottommost_) {
            ((WINDOWPOS*)lParam)->hwndInsertAfter = HWND_BOTTOM;
        }
        break;

    // ---- ADD THIS BLOCK ----
    case WM_SYSCOMMAND:
        // Block minimize when prevention is enabled
        if (bPreventMinimize_ && (wParam & 0xFFF0) == SC_MINIMIZE) {
            return 0;
        }
        break;
    // ---- END ADDED BLOCK ----

    case WM_STYLECHANGED:
```

**Why `(wParam & 0xFFF0)`?** Windows reserves the low 4 bits of `wParam` for system use in `WM_SYSCOMMAND`. The actual command is in the upper bits. Masking with `0xFFF0` extracts the command ID.

### 2c. Handle WM_SIZE as a Safety Net

Win+D may bypass `WM_SYSCOMMAND` on some Windows versions and minimize the window directly. Add a restore-on-minimize fallback in the existing `WM_SIZE` handler:

```cpp
    case WM_SIZE:
        switch (wParam)
        {
        case SIZE_RESTORED:
        case SIZE_MAXIMIZED:
        case SIZE_MINIMIZED:
            // Auto-restore if minimize prevention is active
            if (bPreventMinimize_ && wParam == SIZE_MINIMIZED) {
                // Post a restore message (don't call ShowWindow directly from WndProc)
                PostMessage(hWnd, WM_SYSCOMMAND, SC_RESTORE, 0);
            }

            // Run callback
            if (hWindowStyleChangedHandler_ != nullptr) {
                hWindowStyleChangedHandler_((INT32)WindowStateEventType::Resized);
            }
            break;
        }
        break;
```

`PostMessage` is used instead of `ShowWindow` because calling `ShowWindow` from within `WndProc` can cause re-entrancy issues. `PostMessage` queues the restore for after the current message is processed.

### 2d. Add Exported Functions

**File:** `VisualStudio/LibUniWinC/libuniwinc.cpp`

Add near the other setter functions (after `SetBottommost`, around line 876):

```cpp
/// <summary>
/// Enable or disable minimize prevention
/// </summary>
void UNIWINC_API SetPreventMinimize(const BOOL bEnabled) {
    bPreventMinimize_ = bEnabled;
}

/// <summary>
/// Check if minimize prevention is enabled
/// </summary>
BOOL UNIWINC_API IsPreventMinimize() {
    return bPreventMinimize_;
}
```

### 2e. Add Header Declarations

**File:** `VisualStudio/LibUniWinC/libuniwinc.h`

Add near the other window state functions (after `IsMinimized`, around line 88):

```cpp
UNIWINC_EXPORT BOOL UNIWINC_API IsPreventMinimize();
UNIWINC_EXPORT void UNIWINC_API SetPreventMinimize(const BOOL bEnabled);
```

---

## Step 3: Hide from Taskbar/Alt+Tab

Add `WS_EX_TOOLWINDOW` extended style to remove the window from the taskbar and Alt+Tab switcher.

### 3a. Add Exported Functions

**File:** `VisualStudio/LibUniWinC/libuniwinc.cpp`

Add near the other setter functions:

```cpp
/// <summary>
/// Set or unset the tool window style (hides from taskbar and Alt+Tab)
/// </summary>
void UNIWINC_API SetToolWindow(const BOOL bEnabled) {
    if (!hTargetWnd_) return;

    LONG_PTR exStyle = GetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE);

    if (bEnabled) {
        exStyle |= WS_EX_TOOLWINDOW;
        exStyle &= ~WS_EX_APPWINDOW;
    } else {
        exStyle &= ~WS_EX_TOOLWINDOW;
        exStyle |= WS_EX_APPWINDOW;
    }

    SetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE, exStyle);

    // Windows requires a hide+show cycle to update the taskbar
    ShowWindow(hTargetWnd_, SW_HIDE);
    ShowWindow(hTargetWnd_, SW_SHOW);

    // Re-apply bottommost after the show cycle (SW_SHOW can raise the window)
    if (bIsBottommost_) {
        SetWindowPos(
            hTargetWnd_,
            HWND_BOTTOM,
            0, 0, 0, 0,
            SWP_NOSIZE | SWP_NOMOVE | SWP_NOOWNERZORDER | SWP_NOACTIVATE
        );
    }
}

/// <summary>
/// Check if the tool window style is set
/// </summary>
BOOL UNIWINC_API IsToolWindow() {
    if (!hTargetWnd_) return FALSE;
    LONG_PTR exStyle = GetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE);
    return (exStyle & WS_EX_TOOLWINDOW) != 0;
}
```

**Key detail:** After `ShowWindow(SW_SHOW)`, the window may jump to a higher z-order. The `SetWindowPos(HWND_BOTTOM)` call at the end re-applies bottommost positioning. Without this, the pet would briefly appear above other windows.

### 3b. Add Header Declarations

**File:** `VisualStudio/LibUniWinC/libuniwinc.h`

```cpp
UNIWINC_EXPORT void UNIWINC_API SetToolWindow(const BOOL bEnabled);
UNIWINC_EXPORT BOOL UNIWINC_API IsToolWindow();
```

### 3c. Flicker Considerations

The `ShowWindow(SW_HIDE)` / `ShowWindow(SW_SHOW)` cycle causes a **single-frame flicker**. Two strategies to minimize this:

**Option A: Call once at startup, before the window is visible.**
If your pet is always taskbar-hidden, call `SetToolWindow(true)` immediately after `AttachMyWindow()` in the native init sequence — before the first frame renders. No flicker.

**Option B: Call at runtime, accept the flash.**
If you toggle it at runtime, the flicker is brief (~16ms at 60fps). Users will see a single flash. This is acceptable for an infrequent toggle.

---

## Step 4: C# Bindings

### 4a. Native Interop Declarations

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/LowLevel/UniWinCore.cs`

Add inside the `LibUniWinC` class (after `SetBottommost` at line 127):

```csharp
[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
public static extern void SetPreventMinimize([MarshalAs(UnmanagedType.U1)] bool bEnabled);

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
[return: MarshalAs(UnmanagedType.Bool)]
public static extern bool IsPreventMinimize();

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
public static extern void SetToolWindow([MarshalAs(UnmanagedType.U1)] bool bEnabled);

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
[return: MarshalAs(UnmanagedType.Bool)]
public static extern bool IsToolWindow();
```

### 4b. UniWinCore Wrapper Methods

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/LowLevel/UniWinCore.cs`

Add in the `#region for Windows only` section (after `SetKeyColor`, around line 810):

```csharp
/// <summary>
/// Enable or disable minimize prevention (Windows only)
/// </summary>
public void SetPreventMinimize(bool enabled)
{
    LibUniWinC.SetPreventMinimize(enabled);
}

/// <summary>
/// Check if minimize prevention is enabled
/// </summary>
public bool GetPreventMinimize()
{
    return LibUniWinC.IsPreventMinimize();
}

/// <summary>
/// Set tool window style to hide from taskbar and Alt+Tab (Windows only)
/// </summary>
public void SetToolWindow(bool enabled)
{
    LibUniWinC.SetToolWindow(enabled);
}

/// <summary>
/// Check if tool window style is active
/// </summary>
public bool GetToolWindow()
{
    return LibUniWinC.IsToolWindow();
}
```

### 4c. UniWindowController Properties (Optional)

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/UniWindowController.cs`

If you want these accessible via the `UniWindowController.current` singleton, add properties following the existing pattern (near `isBottommost` around line 169):

```csharp
/// <summary>
/// Prevent window from being minimized (Win+D, Win+M, taskbar click)
/// </summary>
public bool preventMinimize
{
    get { return _uniWinCore != null && _uniWinCore.GetPreventMinimize(); }
    set { _uniWinCore?.SetPreventMinimize(value); }
}

/// <summary>
/// Hide window from taskbar and Alt+Tab (WS_EX_TOOLWINDOW)
/// </summary>
public bool isToolWindow
{
    get { return _uniWinCore != null && _uniWinCore.GetToolWindow(); }
    set { _uniWinCore?.SetToolWindow(value); }
}
```

---

## Step 5: Unity Usage

### Basic Setup

```csharp
using Kirurobo;
using UnityEngine;

public class DesktopPetInit : MonoBehaviour
{
    void Start()
    {
        var uwc = UniWindowController.current;
        if (uwc == null) return;

        // Core desktop pet setup
        uwc.isTransparent = true;
        uwc.isBottommost = true;

        // Prevent minimize (survives Win+D)
        uwc.preventMinimize = true;

        // Hide from taskbar and Alt+Tab
        uwc.isToolWindow = true;

        // Optional: enable file drop
        uwc.allowDropFiles = true;
        uwc.OnDropFiles += files => Debug.Log($"Dropped: {files[0]}");
    }
}
```

### Toggling Desktop Mode

If you want the pet to switch between "normal window" and "desktop pet" modes:

```csharp
public void SetDesktopMode(bool enabled)
{
    var uwc = UniWindowController.current;

    uwc.isBottommost = enabled;
    uwc.preventMinimize = enabled;
    uwc.isToolWindow = enabled;

    if (!enabled)
    {
        // Return to normal window behavior
        uwc.isTopmost = false;
    }
}
```

### Closing the Pet (No Taskbar = No Close Button)

Since the pet has no taskbar entry, provide a way to quit:

```csharp
// Right-click context menu, keyboard shortcut, or UI button
public void QuitPet()
{
    Application.Quit();
}
```

A system tray icon is the proper solution for this, but that's a separate implementation. For now, a right-click context menu on the pet or a keyboard shortcut (e.g., Ctrl+Q) works.

---

## Step 6: Build & Test

### Rebuilding the DLL

1. Open `VisualStudio/LibUniWinC.sln` in Visual Studio
2. Build the `LibUniWinC` project (Release, x64)
3. Copy the output `LibUniWinC.dll` to replace the one in the Unity project:
   - Package location: `Packages/com.kirurobo.uniwinc/Runtime/Plugins/x86_64/LibUniWinC.dll`
   - Or if using the Assets version: `Assets/Kirurobo/UniWindowController/Plugins/x86_64/LibUniWinC.dll`

### Testing Checklist

| Test | Expected Result |
|:-----|:----------------|
| Launch build | Pet appears on desktop, below all windows |
| Click on desktop behind pet | Pet stays at bottom, doesn't come to front |
| Open/close other windows | Pet stays at bottom |
| Press Win+D | Pet does NOT minimize, stays visible |
| Press Win+M | Pet does NOT minimize, stays visible |
| Check taskbar | Pet is NOT shown in taskbar |
| Press Alt+Tab | Pet is NOT in the Alt+Tab list |
| Right-click pet → Quit (or your quit method) | Pet closes |
| Click on pet (transparent area) | Click passes through to desktop |
| Click on pet (opaque area) | Pet receives the click |
| Drag pet via handle | Pet moves, stays at bottom after drag |

### Testing Notes

- **Transparency only works in standalone builds**, not in the Unity Editor
- Build with Player Settings: Run in Background = true, DXGI Flip Mode Swapchain = false
- Test on both single and multi-monitor setups if possible
- Test with and without desktop icons to verify the pet renders behind them

---

## Edge Cases & Troubleshooting

### Win+D Still Minimizes the Pet

Win+D uses a shell-level mechanism that may bypass `WM_SYSCOMMAND` on some Windows versions. The `WM_SIZE` / `SIZE_MINIMIZED` fallback in Step 2c handles this — the window gets minimized but immediately posts `SC_RESTORE` to bring itself back.

If this still doesn't work, the shell may be using `ShowWindow(SW_MINIMIZE)` directly. Add another fallback in the `WM_SHOWWINDOW` handler:

```cpp
case WM_SHOWWINDOW:
    if (bPreventMinimize_ && !wParam) {
        // Window is being hidden — cancel it
        return 0;
    }
    break;
```

### Pet Flashes When SetToolWindow Is Called

This is the `SW_HIDE` / `SW_SHOW` cycle. If calling at startup, move the `SetToolWindow(true)` call as early as possible — ideally right after `AttachMyWindow()` succeeds, before the first `Update()` frame.

### Pet Appears Above Other Windows After Drag

`UniWindowMoveHandle` calls `SetWindowPos` during drag. The bottommost enforcement in `WM_WINDOWPOSCHANGING` should catch this, but if it doesn't, verify that `isBottommost` is still `true` after the drag ends:

```csharp
void OnEndDrag()
{
    // Re-enforce after drag completes
    var uwc = UniWindowController.current;
    uwc.isBottommost = true;
}
```

### User Can't Find or Close the Pet

Without a taskbar entry, the only ways to interact are:
- Clicking the pet directly
- A global keyboard shortcut (you'd need to implement this)
- Task Manager (the process is still visible there)

For now, add a visible UI element (right-click menu or always-visible close button) so the pet can be quit. A tray icon can be added later.

### Multiple Monitors

Bottommost mode works across all monitors — the pet window can be dragged to any screen and stays at the bottom everywhere. No special handling needed.

### Focus Stealing

With `WS_EX_TOOLWINDOW` and bottommost, the pet should never steal focus. If it does, add `WS_EX_NOACTIVATE`:

```cpp
// In SetToolWindow, additionally:
exStyle |= WS_EX_NOACTIVATE;
```

This prevents the window from being activated when clicked. Note: this also means the window won't receive keyboard input, which is fine for a desktop pet.
