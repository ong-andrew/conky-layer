# Conky NextGen Layer (GNOME Wayland Extension)

A lightweight GNOME Shell extension providing desktop pinning and pixel-perfect positioning for **Conky** under **Wayland**. Metadata currently declares GNOME Shell 45–51 support; the setup in [SETUP.md](SETUP.md) was verified on GNOME Shell 48.7.

---

## Why this extension?

Under GNOME Shell on Wayland, the compositor (Mutter) does not support the `wlr-layer-shell` protocol. As a result:
- Conky cannot position itself via standard `gap_x` / `gap_y` coordinates.
- Normal windows float, appear in Alt+Tab, and show up in the Activities Overview.
- Clicking on a widget raises it over normal application windows.

Modeled after the architecture of **Desktop Icons NG (DING)**, this extension bridges that gap from inside Mutter using internal GNOME Shell / Mutter C APIs:
1. Keeps the Conky Wayland toplevel at the bottom of Mutter's normal window layer, below regular applications. GNOME 48 does not expose a supported API to change an already-created Wayland window to `Meta.WindowType.DESKTOP`.
2. Fixes coordinates using Mutter's internal `move_frame(true, x, y)`.
3. Pins widgets across all virtual workspaces (`stick()`).
4. Filters widgets out of the **Alt+Tab** window list.
5. Automatically pushes widgets back down if clicked (`raised` signal handler).
6. Overrides GNOME's **Show Desktop** shortcut with an equivalent minimize/restore toggle that leaves Conky visible.

---

## Installation

For the full tested GNOME Wayland setup—including Conky window settings, logical-coordinate guidance, Show Desktop behavior, and AppImage autostart—see [SETUP.md](SETUP.md).

### Method 1: Debian / Ubuntu Package (.deb)

Install system-wide for all users:
```bash
sudo apt install ./pkg/gnome-shell-extension-conky-layer_*.deb
```

### Method 2: Direct Git Clone (User Directory)

```bash
git clone https://github.com/molnari811023/conky-layer.git ~/.local/share/gnome-shell/extensions/conky-layer@molnar.github.com
```

### Enable the Extension:

Log in to a new session (or restart GNOME Shell), then enable:

```bash
gnome-extensions enable conky-layer@molnar.github.com
```

Or toggle it in the **Extensions** app (`gnome-extensions-app`).

---

## Conky Configuration (`.conf`)

In your Conky `.conf` file, set the Wayland window parameters and encode the target coordinates into `own_window_title`.

### Essential Conky window settings:

```lua
conky.config = {
  -- Enable Wayland output (Conky 1.25+)
  output_backend = 'wayland',

  -- Window management
  own_window = true,
  own_window_type = 'normal',   -- Wayland xdg-toplevel surface
  own_window_class = 'conky',

  -- Desktop coordinate definition in window title (see syntax below):
  own_window_title = 'Conky_Left @!DESKTOP:x=10,y=40',

  -- Transparent background (Conky 1.25+)
  own_window_color = '#0000',

  -- Dimensions
  minimum_width = 360,
  minimum_height = 800,
}
```

---

## `own_window_title` Syntax & Examples

The extension identifies Conky windows by looking for the word **`conky`** (case-insensitive) in the title, and then extracts coordinates following the `@!` marker.

### Supported formats:

#### 1. Key-Value Syntax (Recommended)
You can specify `x` and `y` using `x=<number>` and `y=<number>`. Spaces around `=`, `,`, and `:` are fully tolerated:

```lua
-- Left panel pinned at X=10, Y=40
own_window_title = 'Conky_Left @!DESKTOP:x=10,y=40',

-- Right-hand weather widget with spaces for readability
own_window_title = 'Conky Weather @!DESKTOP: x=1540, y=40',

-- Top system monitor
own_window_title = 'Conky - Top Bar @!x=380, y=10',
```

#### 2. Short Coordinate Syntax
You can also specify coordinates simply as `X,Y` after the `@!` marker:

```lua
-- Short format: X=20, Y=50
own_window_title = 'Conky_Widget @!20,50',
own_window_title = 'Conky_Clock @!380,380',
```

### Format Summary:
- **Title Prefix:** Must contain `conky` (e.g. `Conky`, `conky_left`, `Conky - System`).
- **Tag Marker:** `@!` separates the widget name from coordinate instructions.
- **Coordinates:** `x=<val>, y=<val>` or `<x>,<y>`.
- **Dynamic Updates:** If the title changes at runtime (e.g. via Lua), the extension detects `notify::title` and automatically updates the window position without restarting.

---

## Technical Details

| Feature | Mutter C Mechanism | Result |
|---|---|---|
| **Layer Stacking** | `Meta.Window.lower()` + `raised` signal | Conky stays at the bottom of the normal layer, so applications cover it while it remains visible on the desktop. |
| **Show Desktop** | Custom handler for Mutter's `show-desktop` keybinding | Minimizes the active workspace's other windows and restores them on the next shortcut press, leaving Conky visible. |
| **Taskbar / Dock** | Conky's `skip_taskbar` window hint | Requests taskbar exclusion; handling depends on Mutter's Wayland integration. |
| **Alt+Tab** | JS filter wrapper on `global.display.get_tab_list` | Excludes the widget from the Alt+Tab window cycler. |
| **Positioning** | `meta_window_move_frame(win, true, x, y)` | Bypasses Wayland's client-side positioning prohibition directly from inside the compositor. |
| **Click Behavior** | `raised` GObject signal listener | Re-lowers the widget instantly if clicked (`window.lower()`), preventing it from covering applications. |

---

## Compatibility & Known Issues

- **Desktop Icons NG (DING):** ⚠️ **Known conflict with mouse interaction.**
  DING operates by stretching a massive, transparent "desktop" window over the background to handle icon rendering and drag-to-select boxes. Because DING's input region spans the entire screen, it intercepts and captures all mouse clicks intended for the desktop layer. If you use DING, **mouse interaction (clicks, hover effects) with Conky widgets will not work**. 
  - *Workaround:* If you require interactive Conky widgets (e.g., clicking to change music/weather), you must disable DING and use GNOME's clean desktop. If your widgets are display-only (no clicking required), they will run together perfectly.
