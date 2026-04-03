# Linux-like Workspace Switching on Windows

Replicates the Linux desktop experience on Windows 10/11:

- **`Win + N`** → Switch to Nth virtual desktop (workspace)
- **`Win + Shift + Q`** → Close the active window

## Prerequisites

- [AutoHotkey v2](https://www.autohotkey.com/)
- [VirtualDesktopAccessor.dll](https://github.com/Ciantic/VirtualDesktopAccessor)
  — place the DLL in the same folder as the script

## Setup

### 1. AutoHotkey Script

Save as `virtual-desktops.ahk`:

```ahk
#Requires AutoHotkey v2.0

; Load the VirtualDesktopAccessor DLL
hVDA := DllCall("LoadLibrary",
    "Str", A_ScriptDir "\VirtualDesktopAccessor.dll", "Ptr")

GoToDesktop(num) {
    DllCall(A_ScriptDir "\VirtualDesktopAccessor.dll\GoToDesktopNumber",
        "Int", num)
}

; Win + Shift + Q → close active window
#+q::WinClose("A")

; Win + 1..9 → switch to desktop (0-indexed)
#1::GoToDesktop(0)
#2::GoToDesktop(1)
#3::GoToDesktop(2)
#4::GoToDesktop(3)
#5::GoToDesktop(4)
#6::GoToDesktop(5)
#7::GoToDesktop(6)
#8::GoToDesktop(7)
#9::GoToDesktop(8)
```

### 2. Auto-start on Boot

1. Press `Win + R`, type `shell:startup`, hit Enter.
2. Place a **shortcut** to `virtual-desktops.ahk` in that folder.

### 3. Fix Taskbar Animation Delay

When switching desktops, the taskbar can lag behind due to Windows
animations. Apply the registry tweaks below to make it instant.

#### Windows 10

Save as `disable-animations-w10.reg` and double-click to import:

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"TaskbarAnimations"=dword:00000000

[HKEY_CURRENT_USER\Control Panel\Desktop\WindowMetrics]
"MinAnimate"="0"

[HKEY_CURRENT_USER\Control Panel\Desktop]
"MenuShowDelay"="0"
```

#### Windows 11

Windows 11 uses a new XAML/Mica compositor, so the classic Win 10 keys
don't work reliably. Save as `disable-animations-w11.reg`:

```reg
Windows Registry Editor Version 5.00

; Disable animations via accessibility flag
[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects]
"VisualFXSetting"=dword:00000003

; Disable taskbar animations
[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
"TaskbarAnimations"=dword:00000000

; Disable DWM extras (Aero Peek)
[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\DWM]
"EnableAeroPeek"=dword:00000000

; Disable transparency / Mica (reduces taskbar redraw latency)
[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Themes\Personalize]
"EnableTransparency"=dword:00000000

; Group Policy: disable DWM window animations
[HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\DWM]
"DisallowAnimations"=dword:00000001
```

> **Note:** The `HKEY_LOCAL_MACHINE` key requires admin privileges.
> Right-click the `.reg` file → **Run as administrator**.

**Reboot** after importing for changes to take effect.

## File Structure

```text
.
├── README.md
├── VirtualDesktopAccessor.dll
├── virtual-desktops.ahk
├── disable-animations-w10.reg
└── disable-animations-w11.reg
```

## Reverting

To restore default animations:

- **Windows 10/11:** Go to **Settings → Accessibility → Visual effects**
  and toggle **Animation effects** back **On**.
- Delete the registry keys added above, or re-enable them manually via
  `regedit`.
- The `HKEY_LOCAL_MACHINE\...\DWM\DisallowAnimations` key can be deleted
  or set to `0`.

## License

Do whatever you want with this. No warranty.