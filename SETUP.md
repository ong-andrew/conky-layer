# Reproducible Conky-on-GNOME-Wayland Setup

This guide installs the Conky Layer extension, configures a Conky Wayland window for it, and optionally starts a Conky AppImage at login. It documents the working setup tested with **GNOME Shell 48.7**, **Wayland**, and **Conky 1.25.4**. The extension metadata declares GNOME Shell 45–51 support, but Mutter/GNOME internal APIs can change between releases.

## What the extension does

GNOME/Mutter does not provide a layer-shell surface for Conky on Wayland. Conky therefore creates a normal `xdg-toplevel`; the extension recognizes it by its title, moves it to the configured logical-screen coordinates, sticks it across workspaces, and lowers it beneath application windows.

The extension also overrides Mutter's `show-desktop` keybinding. The first press minimizes other minimizable windows on the active workspace while leaving Conky visible; the next press restores the windows it minimized. This is a minimize/restore toggle rather than Mutter's internal desktop-window mode. It applies when the named `show-desktop` keybinding is used (Super+D by default).

## 1. Install and enable the extension

From a checkout of this repository:

```bash
uuid='conky-layer@molnar.github.com'
dest="$HOME/.local/share/gnome-shell/extensions/$uuid"
mkdir -p "$dest"
install -m 644 metadata.json extension.js "$dest/"
```

Log out and back in so GNOME Shell discovers the newly copied extension. Then enable and verify it:

```bash
gnome-extensions enable conky-layer@molnar.github.com
gnome-extensions info conky-layer@molnar.github.com
```

The status should show `Enabled: Yes` and `State: ACTIVE`. If the extension is already installed, copy the updated `extension.js` and log out/in to ensure the running Shell loads the new code.

## 2. Configure the Conky window

Merge these settings into your existing Conky configuration (commonly `~/.config/conky/conky.conf`). Do not replace the rest of a custom dashboard just to install the extension.

```lua
conky.config = {
    output_backend = 'wayland',
    own_window = true,
    own_window_type = 'normal',
    own_window_class = 'Conky',
    own_window_title = 'Conky_Aura @!DESKTOP:x=1224,y=28',
    own_window_color = '#14171C99',
    own_window_hints = 'undecorated,skip_taskbar,skip_pager',

    minimum_width = 284,
    maximum_width = 284,
    -- Keep the rest of your existing Conky settings here.
}
```

Important details:

- Keep `own_window_type = 'normal'`: this is the Wayland toplevel that the extension can manage. The extension cannot change an already-created Wayland window to Mutter's `DESKTOP` type.
- The title must contain `Conky` (case-insensitive) and coordinates. The recommended form is `Conky_Name @!DESKTOP:x=<x>,y=<y>`.
- Coordinates are in Mutter's **logical screen coordinates**, not necessarily the panel's physical pixel resolution. For a 1920×1200 display at 125% scale, the logical size is 1536×960. The example `x=1224` places a 284-logical-pixel-wide panel 28 logical pixels from the right edge: `1536 - 284 - 28`.
- Recalculate the title coordinates for your display size, scale, monitor layout, panel width, and desired margins. `gap_x`/`gap_y` alone do not position the Wayland toplevel; the extension uses the title coordinates.
- Keep only one `own_window_class` entry in the Lua table.

Start Conky with your AppImage to check the configuration:

```bash
"$HOME/AppImages/conky.appimage" --daemonize --pause=1
```

Use `-c "$HOME/.config/conky/conky.conf"` if your configuration is not in Conky's default config location.

## 3. Optional: start the AppImage at login

Create `~/.config/autostart/conky.desktop`. Replace the executable and icon paths with the paths on your system; desktop-entry `Exec` paths should be absolute and should not use `~`.

```ini
[Desktop Entry]
Type=Application
Name=Conky
Icon=/home/USER/AppImages/.icons/conky
TryExec=/home/USER/AppImages/conky.appimage
Exec=env DESKTOPINTEGRATION=1 /home/USER/AppImages/conky.appimage --daemonize --pause=1
Terminal=false
StartupNotify=false
X-GNOME-Autostart-enabled=true
```

If the AppImage was not installed by Gear Lever, `DESKTOPINTEGRATION=1` may be omitted. The `--pause=1` gives the session a moment to settle; the extension also handles Conky windows that are already open when it loads.

## 4. Verify the behavior

After logging in:

1. Confirm the extension is `ACTIVE` with `gnome-extensions info conky-layer@molnar.github.com`.
2. Confirm Conky is running with `pgrep -af conky.appimage`.
3. Check the Shell log for a pin message:

   ```bash
   journalctl --user -b --no-pager | grep 'Conky-NextGen'
   ```

4. Press **Super+D** once: other minimizable windows on the active workspace should hide while Conky remains visible. Press it again to restore the windows hidden by the first press.

## Notes and limitations

- The extension lowers Conky to the bottom of Mutter's **normal** window layer. Applications cover it; it remains visible on the desktop. This is not the same as a native `DESKTOP`-type window.
- The Show Desktop replacement only handles the Mutter keybinding named `show-desktop`; it does not change behavior for other tools that invoke Mutter's internal Show Desktop mode directly.
- Desktop Icons NG (DING) can intercept mouse input over desktop widgets; see the compatibility notes in [README.md](README.md).
- The sample AppImage path, panel width, and coordinates are examples. Keep personal Conky content and machine-specific values local to the user's config.
