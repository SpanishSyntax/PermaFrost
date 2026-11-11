# Configuration Reference

This guide details configuration parameters across the NixOS module, the `greetd` daemon, compositor kiosks, and PAM auto-unlocking.

---

## NixOS Module Options

PermaFrost exports a standard NixOS service module under `services.greetd.astraAirlock`:

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `services.greetd.astraAirlock.enable` | boolean | `false` | Enables PermaFrost (Airlock) as the default display manager frontend for greetd. |
| `services.greetd.astraAirlock.package` | package | `self.packages.${system}.default` | The compiled `astra-airlock` package derivation to install and execute. |
| `services.greetd.astraAirlock.compositor` | enum: `["cage", "hyprland"]` | `"cage"` | Wayland compositor used to host the greeter interface. |

### Flake Configuration Example

```nix
{
  services.greetd.astraAirlock = {
    enable = true;
    compositor = "cage";
  };
}
```

The NixOS module automatically performs:
1. Enabling `services.greetd.enable = true;`
2. Setting `default_session` to invoke `cage -s -- astra-airlock` (or `Hyprland`) with standard output silenced to avoid TTY flickering.
3. Registering systemd tmpfiles rule: `d /var/cache/astra-airlock 0755 greeter greeter -`.

---

## greetd Configuration

The core daemon is configured in `/etc/greetd/config.toml`.

### Cage Kiosk (Minimal Footprint)

```toml
[terminal]
vt = 1

[default_session]
command = "cage -s -- astra-airlock >/dev/null 2>&1"
user = "greeter"
```

> [!NOTE]
> Redirecting stdout and stderr to `/dev/null` suppresses compositor initialization messages on the virtual terminal, preventing brief flashes of console text prior to UI render.

### Hyprland Kiosk

```toml
[terminal]
vt = 1

[default_session]
command = "start-hyprland -- -c /etc/greetd/hyprland.lua >/dev/null 2>&1"
user = "greeter"
```

---

## Hyprland Kiosk Configuration

When running inside Hyprland, the greeter's window environment is controlled by `/etc/greetd/hyprland.lua`:

```lua
-- Output configuration
hl.monitor({ output = "", mode = "preferred", position = "auto", scale = 1 })

-- Environmental options
hl.config({
  animations = { enabled = false },
  decoration = {
    blur = { enabled = false },
    shadow = { enabled = false },
  },
  input = {
    kb_layout = "us",
    numlock_by_default = false,
    repeat_delay = 250,
    repeat_rate = 35,
    touchpad = {
      natural_scroll = true,
      disable_while_typing = true,
      scroll_factor = 0.3,
    },
  },
  misc = {
    disable_autoreload = true,
    disable_hyprland_logo = true,
    force_default_wallpaper = 0,
  },
})

-- Cursor theme definitions (must exist in /usr/share/icons)
local cursor_theme = "Bibata-Modern-Classic"
local cursor_size = "14"

hl.env("HYPRCURSOR_THEME", cursor_theme)
hl.env("HYPRCURSOR_SIZE", cursor_size)
hl.env("XCURSOR_THEME", cursor_theme)
hl.env("XCURSOR_SIZE", cursor_size)

-- Launch greeter on startup, then exit kiosk once login completes
hl.on("hyprland.start", function()
  hl.exec_cmd("astra-airlock; hyprctl dispatch exit")
end)
```

> [!IMPORTANT]
> Because greetd runs under the system `greeter` user, custom cursor themes must reside in `/usr/share/icons/` with global read permissions (`chmod -R 755`), rather than inside `~/.icons/`.

---

## PAM Configuration & Keyring Auto-Unlock

To ensure that GNOME Keyring unlocks automatically on login without requesting a secondary password prompt, verify `/etc/pam.d/greetd`:

```text
# /etc/pam.d/greetd - PAM configuration for greetd
#%PAM-1.0

auth       required     pam_securetty.so
auth       requisite    pam_nologin.so
auth       include      system-local-login
auth       optional     pam_gnome_keyring.so

account    include      system-local-login

password   include      system-local-login

session    include      system-local-login
session    optional     pam_gnome_keyring.so auto_start
```

- `auth optional pam_gnome_keyring.so`: Captures the entered plaintext password during authentication.
- `session optional pam_gnome_keyring.so auto_start`: Initializes the login keyring daemon and passes the captured password to unlock it.
