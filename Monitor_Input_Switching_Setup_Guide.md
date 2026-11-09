# Software Monitor Input Switching: Arch Linux + Windows 10 on a BenQ RD320U KVM

A record of the setup used to switch the monitor's video input between two computers with a single keyboard shortcut, without touching the monitor's OSD (on-screen display) menu. The monitor's built-in KVM (keyboard, video and mouse) switch moves the USB keyboard and mouse along with the video.

## Hardware and Context

| Item | Detail |
|---|---|
| Monitor | BenQ RD320U, with built-in KVM (USB peripherals follow the active video input) |
| Machine 1 | Arch Linux PC, Hyprland desktop |
| Machine 2 | Windows 10 laptop, configured to keep running with the lid closed, with the BenQ as primary display; also usable with the lid open |
| Connections, Arch PC | Video: HDMI, into the monitor's HDMI-2 input. USB (for the KVM): from a USB-A port on the PC into the monitor's USB-B upstream port (the larger, square port) |
| Connections, Windows laptop | A single USB-C cable into the monitor's USB-C input, which carries video (DisplayPort Alt Mode), USB data and power |

Both machines are connected to the monitor. Pressing the shortcut on whichever machine is currently active switches the monitor to the other machine's input, and the KVM hands over the keyboard and mouse at the same time.

### Tested With

| Component | Version |
|---|---|
| Arch Linux | Rolling release, tested November 2026; kernel 7.2.14-arch1-1 |
| Hyprland | 0.57.1 (Arch package hyprland 0.57.1-1) |
| ddcutil | 3.0.2 |
| Windows 10 | 22H2 (OS build 19045.6466) |
| ControlMyMonitor | 1.43 |
| AutoHotkey | 2.0.31 |

## How It Works

The monitor's input source is a setting that the computer can change over DDC/CI (the Display Data Channel Command Interface): VCP (Virtual Control Panel) code 60 (named "Input Source" by ddcutil and "Input Select" by ControlMyMonitor). Writing a value to this setting switches the input exactly as if it had been chosen in the monitor's own menu. Since the RD320U's KVM follows the active input, switching VCP 60 also moves the USB devices.

The tooling differs per platform, but both sides send the same DDC/CI command:

| Platform | Tool | Command |
|---|---|---|
| Arch Linux | ddcutil | `ddcutil setvcp 60 <value>` |
| Windows 10 | NirSoft ControlMyMonitor | `ControlMyMonitor.exe /SetValue <monitor> 60 <value>` |

### Input Values on This Monitor

The mapping below was established for this particular RD320U by testing:

| VCP 60 value | ddcutil name | Input |
|---|---|---|
| 18 (0x12) | HDMI-2 | Arch Linux PC (the target when pressing the shortcut on Windows) |
| 19 (0x13) | Unrecognized value | Windows 10 laptop, USB-C (the target when pressing the shortcut on Linux) |
| 15 (0x0f) | DisplayPort-1 | Also selected the Windows laptop in testing; not used |
| 17 (0x11) | HDMI-1 | No visible effect in testing; not used |

## Linux Side (Arch Linux + Hyprland)

### Software

- `ddcutil` (installed from the Arch repositories: `pacman -S ddcutil`). Communicates with the monitor over DDC/CI.

### Access to the I2C Devices

`ddcutil` talks to the monitor over the graphics card's I2C bus, through the `/dev/i2c-*` device files. The ddcutil package sets up everything this needs, so neither `sudo` nor any group membership is required:

- it loads the `i2c-dev` kernel module at boot (`/usr/lib/modules-load.d/ddcutil.conf`);
- its udev rule (`/usr/lib/udev/rules.d/60-ddcutil-i2c.rules`) gives the logged-in user read-write access to the display's I2C devices, in the graphical session and on a text console alike.

After `pacman -S ddcutil`, a reboot puts both into effect. `ls -l /dev/i2c-*` shows a `+` after the permissions on the devices the user can access.

### Testing in the Terminal

Before binding anything, values were verified from the Linux side:

```bash
# Identify the monitor and which I2C bus it sits on
ddcutil detect

# Ask the monitor which VCP features it supports, and which input values it accepts
ddcutil capabilities

# Read the current input source (VCP 60); the value is reported in hex as sl=0x..
ddcutil getvcp 60

# Test switching by writing candidate values (decimal; x0f, x11, x12, x13 are equivalent)
ddcutil setvcp 60 15
ddcutil setvcp 60 17
ddcutil setvcp 60 18
ddcutil setvcp 60 19
```

Discovering the candidate values: `ddcutil capabilities` reports the monitor's supported features, including feature 60 (Input Source) and the input values the monitor itself advertises. ddcutil lists these values in hexadecimal; for this unit they are `0f` (DisplayPort-1), `11` (HDMI-1), `12` (HDMI-2) and `13` (listed as "Unrecognized value"), which correspond to decimal 15, 17, 18 and 19. `setvcp` accepts a decimal value by default, or a hexadecimal value prefixed with `x` (for example, `ddcutil setvcp 60 x13`).

The values were then mapped to physical inputs from the Linux side, with the Windows laptop connected to the monitor and switched on:

1. With the monitor showing the Linux desktop, writing 18 (0x12, HDMI-2) produced no visible change, as it is the input already in use by the Arch PC. (`ddcutil getvcp 60` also reports the active input directly.)
2. Writing 19 (0x13) brought up the Windows desktop, identifying 19 as the laptop's input.
3. Writing 15 (0x0f, DisplayPort-1) unexpectedly brought up the Windows desktop as well, although nothing is connected to the DisplayPort input. The reason was not established; 19 is used in this setup.
4. Writing 17 (0x11, HDMI-1) produced no visible change.
5. Whenever a test moved the display (and with it the keyboard) away from the Linux PC, the Linux input was selected again from the monitor's own menu.

The values were later cross-checked on Windows, where ControlMyMonitor listed the same values in decimal.

### Hyprland Binding

The command was bound to Super+Ctrl+W (Super being the Windows key) in the Hyprland configuration file, `~/.config/hypr/hyprland.lua`, which uses the Lua configuration format of current Hyprland versions (`mainMod` is defined earlier in the file as `"SUPER"`):

```lua
hl.bind(mainMod .. " + CTRL + W", hl.dsp.exec_cmd("ddcutil setvcp 60 19"))
```

(In the older hyprlang configuration format, `hyprland.conf`, the equivalent is `bind = SUPER CTRL, W, exec, ddcutil setvcp 60 19`.)

Result: pressing Super+Ctrl+W on the Linux PC switches the monitor and KVM to the Windows laptop.

### Console (Non-GUI) Binding

The Hyprland binding only works inside the graphical session. To make Super+Ctrl+W work on a text console (a virtual terminal, or tty, with no graphical session), a kernel keymap macro is used instead.

This work does not require Hyprland to be closed. A text console can be opened alongside the running graphical session by pressing Ctrl+Alt+F2 (or F3 to F6), logging in there, and carrying out the steps below. The graphical session is resumed by switching back to the text console on which Hyprland runs (Ctrl+Alt+F1 when Hyprland was started from tty1).

Create/edit the keymap file:

```bash
sudo vim /usr/local/share/kbd/keymaps/kvm-switch.map
```

Contents:

```
# 1. Define the macro string (F100 is a safe custom string identifier)
string F100 = "/usr/bin/ddcutil setvcp 60 19\n"

# 2. Make the left Super (Windows) key act as a console modifier
#    (keycode 125 is the left Super key; ShiftL is a modifier no other key uses by default)
keycode 125 = ShiftL

# 3. Bind Super + Ctrl + W to execute that macro string
shiftl control keycode 17 = F100
```

How it works: the kernel's keyboard driver can bind a spare function-key action code (F100, which no key normally uses) to an arbitrary string. The console keymap has no Super modifier of its own, so the left Super key (keycode 125) is assigned to the spare `ShiftL` modifier, and the binding is made on the `shiftl control` combination. When Super+Ctrl+W is pressed on the console, the kernel types the string `/usr/bin/ddcutil setvcp 60 19` followed by Enter into the active text console, as if it had been typed by hand. Keycode 17 is the W key.

Loading and persistence:

```bash
# Load for the current session only
sudo loadkeys /usr/local/share/kbd/keymaps/kvm-switch.map
```

To make the binding load automatically at boot, specify the full path in the `KEYMAP` variable in `/etc/vconsole.conf` (systemd reads this file at startup and runs loadkeys internally):

```
KEYMAP=/usr/local/share/kbd/keymaps/kvm-switch.map
```

`loadkeys` adds the file's entries to the keymap already loaded; it does not replace it. Where an earlier version of the file has been loaded in the running session (for example, an earlier Ctrl+Alt+W binding), the old bindings therefore remain active. In that case the default keymap is to be restored first, and the file loaded afterwards:

```bash
sudo loadkeys -d
sudo loadkeys /usr/local/share/kbd/keymaps/kvm-switch.map
```

`loadkeys -d` loads the default keymap, which replaces any other console layout loaded earlier in the session. The step is not needed after a reboot, since the file is then loaded through `vconsole.conf` and no earlier version is in place.

Notes:

- The keymap change affects all text consoles, and does not interfere with the Hyprland binding — the GUI session and the console each handle their own key events.
- The full path `/usr/bin/ddcutil` is used so that the command works whatever the shell's search path.
- Because the macro works by typing, the command line is visibly echoed on the console before it runs. For the same reason, it is typed into whatever has the focus on the console, so it should be used only at a shell prompt.

## Windows Side (Windows 10 Laptop)

### Software Installed

- **ControlMyMonitor** (NirSoft, free, single executable, no installer): https://www.nirsoft.net/utils/control_my_monitor.html — provides GUI inspection of VCP codes and a command-line mode for setting them.
- **AutoHotkey v2**: https://www.autohotkey.com/ — provides the global keyboard shortcut. The script syntax below is AutoHotkey v2.

ControlMyMonitor was placed at `C:\Tools\ControlMyMonitor\ControlMyMonitor.exe`.

### Determining the Correct Values

1. Launched the ControlMyMonitor GUI with the monitor showing the Windows desktop.
2. Noted the current value of VCP code 60 (Input Select): **19**, confirming 19 is the laptop's input. The GUI also listed the possible values: 15, 17, 18 and 19.
3. Tested each candidate value to find the Linux input: **18**.

### Identifying the Monitor

Addressing the monitor correctly turned out to be the main pitfall:

- `Primary` is not reliable on a laptop: with the lid open, "primary" may be the internal panel, and a VCP 60 write to the internal display silently does nothing.
- The `\\.\DISPLAYn\Monitor0` string describes a position in Windows' display enumeration, which can shift when the internal panel activates (lid open vs closed).

The robust solution: address the monitor by its **serial number**, shown in the ControlMyMonitor GUI. It is written below as `<MONITOR_SERIAL>`, to be replaced with the monitor's actual serial number. The serial stays constant regardless of display topology (lid state, docks, ports).

### The Script

Created `switch-input.ahk` in the Startup folder, which is opened by typing `shell:startup` in the Run dialog (Win+R).

Final content:

```autohotkey
^#w::Run '"C:\Tools\ControlMyMonitor\ControlMyMonitor.exe" /SetValue "<MONITOR_SERIAL>" 60 18',, "Hide"
```

- `^#w` = Ctrl+Win+W (`^` is Ctrl and `#` is the Windows key; the same combination as the Linux Super+Ctrl+W binding)
- The serial number targets the BenQ specifically
- Value 18 switches to the Arch Linux PC
- `"Hide"` suppresses any window flash

The Startup folder means the script loads automatically at login. After creating or editing the file mid-session, run it manually (double-click) or reload it via the green "H" AutoHotkey tray icon.

## Troubleshooting Notes

Issues encountered and how they were resolved during setup:

1. **Hotkey "did nothing"** — Diagnosed by testing command and hotkey separately. Run the ControlMyMonitor command directly in a Command Prompt first; if it works there but not via the hotkey, the problem is the AutoHotkey layer (script not running, or AutoHotkey v1 installed where v2 syntax was used).
2. **"Primary" addressing the wrong display** — With the laptop lid open, `Primary` could resolve to the internal panel. Fixed by using the monitor's display string, then ultimately its serial number.
3. **Display string instability** — `\\.\DISPLAY1\Monitor0` may change to `\\.\DISPLAY2\Monitor0` when the internal panel activates. Fixed permanently by addressing the monitor by serial number; verified working with the lid both open and closed.
4. **Script not active after reboot** — Startup items load at login only; check for the green AutoHotkey tray icon.
5. **Console binding did nothing after the keymap was changed** — An earlier version of the keymap (a Ctrl+Alt+W binding) was still loaded, since `loadkeys` adds to the current keymap rather than replacing it. Resolved with `sudo loadkeys -d`, followed by loading the revised file again.

## Quick Reference

| Action | Where | How |
|---|---|---|
| Switch to Windows (from Linux, Hyprland) | Arch PC | Super+Ctrl+W → `ddcutil setvcp 60 19` via Hyprland bind |
| Switch to Windows (from Linux, text console) | Arch PC | Super+Ctrl+W → keymap macro in `/usr/local/share/kbd/keymaps/kvm-switch.map` (loaded by `loadkeys` / `vconsole.conf`) |
| Switch to Linux (from Windows) | Windows laptop | Ctrl+Win+W → `ControlMyMonitor.exe /SetValue "<MONITOR_SERIAL>" 60 18` |
| Read current input value (Linux) | Terminal | `ddcutil getvcp 60` (reported in hex, e.g. `sl=0x12` = 18) |
| Read current input value (Windows) | Command Prompt | `ControlMyMonitor.exe /GetValue "<MONITOR_SERIAL>" 60` |
| Reload console keymap after editing | Arch PC, text console | `sudo loadkeys -d && sudo loadkeys /usr/local/share/kbd/keymaps/kvm-switch.map` |

## Sources

- NirSoft ControlMyMonitor and its command-line usage: https://www.nirsoft.net/utils/control_my_monitor.html and https://www.nirsoft.net/articles/set_monitor_input_source_command_line.html
- AutoHotkey: https://www.autohotkey.com/
- ddcutil: https://www.ddcutil.com/
- Hyprland configuration (Lua binding syntax): https://wiki.hypr.land/configuring/
- Console keymap syntax: the `keymaps(5)` manual page of the kbd package (`man 5 keymaps`)
