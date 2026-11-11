# Getting Started

This guide covers system prerequisites, installation paths, and initial deployment of PermaFrost (Airlock) with `greetd`.

---

## Prerequisites

### Runtime Requirements

- **[greetd](https://git.sr.ht/~kennylevinsen/greetd)** daemon
- **[Quickshell](https://quickshell.outfoxxed.me/)** $\ge$ 0.3.0
- **Qt 6.6+** (Core, DBus, Qml, Quick, Quick3D)
- **M3Shapes QML module** (provided by Caelestia Shell or upstream m3shapes)
- **Wayland Compositor**: [Cage](https://github.com/cage-kiosk/cage) (recommended for minimal kiosk) or [Hyprland](https://github.com/hyprwm/hyprland)

### Optional Dependencies

- **[caelestia-cli](https://github.com/caelestia-dots/cli)** — Required for desktop scheme extraction via `--sync`.
- **[wlr-randr](https://gitlab.freedesktop.org/emersion/wlr-randr)** — Required when specifying custom monitor output flags (`--only`, `--output`, `--mode`).

---

## Installation

### Nix & NixOS Setup

Add PermaFrost to your system `flake.nix`:

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    permafrost.url = "github:SpanishSyntax/PermaFrost";
  };

  outputs = { self, nixpkgs, permafrost, ... }: {
    nixosConfigurations.workstation = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        permafrost.nixosModules.default
        {
          services.greetd.astraAirlock = {
            enable = true;
            compositor = "cage"; # "cage" or "hyprland"
          };
        }
      ];
    };
  };
}
```

The NixOS module automatically provisions the `greetd` service, configures the greeter command, and sets up permissions for `/var/cache/astra-airlock`.

---

### Arch Linux (AUR)

Install via your preferred AUR helper:

```bash
# Stable release
paru -S astra-airlock

# Latest development branch
paru -S astra-airlock-git
```

> [!NOTE]
> The AUR package post-install script automatically adds GNOME Keyring entries to `/etc/pam.d/greetd` and sets up the required system groups.

---

### Manual Compilation (CMake)

```bash
# Clone the repository
git clone https://github.com/SpanishSyntax/PermaFrost.git ~/PermaFrost
cd ~/PermaFrost

# Configure and compile launcher and C++ QML plugins
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build

# Install binaries and configuration
sudo cmake --install build
```

Installed artifacts:
- Binary: `/usr/bin/astra-airlock`
- QML Plugin: `/usr/lib/qt6/qml/Astra/Airlock/`
- Shell files: `/etc/xdg/quickshell/astra-airlock/`

---

## Quick Start

### 1. Initialize System Greeter User & Cache

If installing manually or on non-Nix distributions, ensure the `greeter` user and cache directory exist:

```bash
# Create greeter user if not provided by greetd
getent passwd greeter >/dev/null || \
    sudo useradd -r -M -s /usr/bin/nologin -d /var/cache/astra-airlock greeter

# Ensure video and input permissions
sudo usermod -aG video,input greeter

# Create shared state store
sudo install -d -m 0755 -o greeter -g greeter /var/cache/astra-airlock
```

### 2. Deploy greetd Kiosk Configuration

Use the built-in `--kiosk` command to generate your `/etc/greetd/config.toml`:

```bash
# Kiosk deployment using Cage
sudo astra-airlock --kiosk cage

# Or kiosk deployment using Hyprland
sudo astra-airlock --kiosk hyprland
```

### 3. Register User Avatar

Because `greetd` runs under the unprivileged `greeter` user, it cannot read files in user home directories (`~/.face`). Copy your avatar into the greeter cache:

```bash
sudo astra-airlock --set-pfp /path/to/avatar.png
```

### 4. Enable and Start greetd

```bash
# Disable existing display manager (e.g. SDDM or GDM)
sudo systemctl disable sddm.service

# Enable and start greetd
sudo systemctl enable --now greetd.service
```
