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
#SingleInstance Force

; Re-launch as admin if not already elevated
if !A_IsAdmin {
    Run '*RunAs "' A_ScriptFullPath '"'
    ExitApp
}

; Load the DLL (put VirtualDesktopAccessor.dll next to this script)
hVDA := DllCall("LoadLibrary",
    "Str", A_ScriptDir "\VirtualDesktopAccessor.dll", "Ptr")

GoToDesktop(num) {
    DllCall(A_ScriptDir "\VirtualDesktopAccessor.dll\GoToDesktopNumber",
        "Int", num)
}

MoveWindow(num) {
    hwnd := WinGetID("A")
    DllCall(A_ScriptDir "\VirtualDesktopAccessor.dll\MoveWindowToDesktopNumber",
        "Ptr", hwnd, "Int", num)
}

; Win + Shift + 1..9 → move active window to desktop N
#+1::MoveWindow(0)
#+2::MoveWindow(1)
#+3::MoveWindow(2)
#+4::MoveWindow(3)
#+5::MoveWindow(4)
#+6::MoveWindow(5)
#+7::MoveWindow(6)
#+8::MoveWindow(7)
#+9::MoveWindow(8)

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

### 2. Auto-start on Boot (with Admin Privileges)

Using Task Scheduler ensures the script runs elevated on login, which is
needed for hotkeys to work across all windows (including those running
as admin).

#### Option A: GUI

1. Press `Win + R`, type `taskschd.msc`, hit Enter.
2. Click **Create Task** (not "Create Basic Task").
3. **General** tab:
   - Name: `Virtual Desktop Hotkeys`
   - Check **Run only when user is logged on**
   - Check **Run with highest privileges**
4. **Triggers** tab → **New…**:
   - Begin the task: **At log on**
   - Specific user: your account
5. **Actions** tab → **New…**:
   - Action: **Start a program**
   - Program/script: `C:\Path\To\AutoHotkey.exe`
   - Add arguments: `"C:\Path\To\virtual-desktops.ahk"`
6. **Conditions** tab:
   - Uncheck **Start the task only if the computer is on AC power**
7. **Settings** tab:
   - Uncheck **Stop the task if it runs longer than…**
   - Set **If the task is already running**: **Do not start a new instance**
8. Click **OK**.

#### Option B: One-liner (PowerShell as Admin)

```powershell
$ahk = "C:\Path\To\AutoHotkey.exe"
$script = "C:\Path\To\virtual-desktops.ahk"

$action = New-ScheduledTaskAction -Execute $ahk -Argument "`"$script`""
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -RunLevel Highest -LogonType Interactive
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0

Register-ScheduledTask -TaskName "Virtual Desktop Hotkeys" `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Settings $settings
```

> **Update the paths** to match where you placed AutoHotkey and the script.

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