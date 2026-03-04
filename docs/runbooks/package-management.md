# Package Management Runbook

This is a practical guide to managing packages on NixOS without creating the cruft and dependency sprawl that accumulates on traditional distributions like Debian.

## The Three Contexts for Installing Software

NixOS has three distinct scopes for packages. Using the right scope is the core discipline that prevents package creep.

### 1. System Packages (`configuration.nix`)

**What goes here:** System-wide tools, services, drivers, anything that needs root-level integration.

**File:** `hosts/framework/configuration.nix` (or modules it imports)

```nix
environment.systemPackages = with pkgs; [
  vim
  git
  htop
  lm_sensors
  btrfs-progs
  pkgs-unstable.citrix_workspace  # Work-critical, from unstable
  zoom-us                         # Work-critical
];
```

**Rebuild required:** `sudo nixos-rebuild switch --flake ~/projects/nixos#framework`

**When to use:** CUPS, Docker, filesystem tools, hardware tools, display manager config, work-critical apps that need system integration (Citrix, Zoom).

### 2. User Packages (`home.nix` via Home Manager)

**What goes here:** Applications you personally use, editors, browsers, dev tools.

**File:** `home/zervin/default.nix` (or modules it imports)

```nix
home.packages = with pkgs; [
  brave
  vscode
  obsidian
  gimp
  nodejs_22
  rustup
];
```

**Rebuild required:** Same `sudo nixos-rebuild switch` command (Home Manager is integrated as a NixOS module in our flake).

**When to use:** Browsers, editors, creative tools, CLI utilities you use daily, language runtimes you use across many projects.

### 3. Project Packages (devShells via `flake.nix` per project)

**What goes here:** Toolchains and dependencies specific to a single project.

**File:** `flake.nix` in the project's root directory (e.g., `~/projects/my-app/flake.nix`)

```nix
# ~/projects/my-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ nodejs_22 bun nodePackages.typescript ];
        shellHook = ''echo "Node $(node --version) dev environment"'';
      };
    };
}
```

**Activation:** `cd ~/projects/my-app && nix develop`

Or with direnv (recommended), add `.envrc` containing `use flake` and the shell activates automatically on `cd`.

**When to use:** Flutter + Android SDK, specific Node.js version for a project, Python with ML dependencies, any toolchain that should not pollute your global environment.

**Why this matters:** On Debian, installing Flutter globally means `flutter`, `dart`, `adb`, `android-sdk`, and dozens of transitive dependencies live in your system permanently. On NixOS with devShells, they exist only while you are working on that project.

## The Golden Rule: Never Use `nix-env -i`

`nix-env -i <package>` installs a package imperatively into your user profile. This is the NixOS equivalent of `apt install` and creates the exact same problems:

- The package is not tracked in any `.nix` file
- `nixos-rebuild` does not know about it
- It survives rebuilds silently
- Your system cannot be reproduced from config alone

**If you catch yourself typing `nix-env -i`, stop.** Instead:

| What you want | What to do |
|---|---|
| Install permanently | Add to `home.packages` in `home.nix`, rebuild |
| Try for 5 minutes | `nix shell nixpkgs#package-name` |
| Run one command | `nix run nixpkgs#package-name -- args` |
| Use in one project | Add to project's `flake.nix` devShell |

## Ephemeral Package Use (Zero Creep)

These commands install nothing permanently. When you exit, the packages are gone (they remain in `/nix/store` until GC, but are not on your PATH):

```bash
# Run a single command with a package:
nix run nixpkgs#cowsay -- "Hello NixOS"
nix run nixpkgs#python313 -- -c "print('hello')"

# Drop into a shell with one or more packages:
nix shell nixpkgs#imagemagick nixpkgs#ffmpeg
convert input.png -resize 50% output.png
ffmpeg -i video.mp4 audio.mp3
exit  # Packages are gone from PATH

# Run a package from unstable:
nix shell github:NixOS/nixpkgs/nixos-unstable#some-bleeding-edge-tool
```

## Adding a Package from the Unstable Overlay

When a stable package has a regression or is too old:

1. Identify the problem with the stable version
2. Test the unstable version: `nix shell github:NixOS/nixpkgs/nixos-unstable#package-name`
3. If it fixes the issue, add it to the config with `pkgs-unstable`:

```nix
# In configuration.nix or home.nix:
environment.systemPackages = [
  # Citrix Workspace: stable has libwebkitgtk2 regression (Nov 2024).
  # Using unstable until nixos-25.05 ships with the fix.
  pkgs-unstable.citrix_workspace
];
```

4. Always add a comment explaining WHY the package is from unstable
5. On the next stable release, check if the issue is resolved and switch back:

```nix
  # Fixed in 25.05, moved back to stable:
  pkgs.citrix_workspace
```

## Finding Packages

```bash
# Search nixpkgs by name (searches attribute name and description):
nix search nixpkgs#firefox
nix search nixpkgs#citrix

# Search unstable:
nix search github:NixOS/nixpkgs/nixos-unstable#zoomvdi

# Web search (better for browsing):
# https://search.nixos.org/packages

# Show package metadata:
nix eval nixpkgs#firefox.meta.description
nix eval nixpkgs#firefox.version

# List all available versions of a package:
nix search nixpkgs#nodejs  # Shows nodejs_18, nodejs_20, nodejs_22, etc.
```

## Garbage Collection

### Understanding the Nix Store

Every `nixos-rebuild switch` creates a new "generation." Each generation references a set of store paths in `/nix/store`. Old generations keep their store paths alive until you explicitly delete them.

```
Generation 1 (3 days ago)  --> references 15,000 store paths
Generation 2 (2 days ago)  --> references 15,200 store paths (shares most with gen 1)
Generation 3 (current)     --> references 15,100 store paths (removed a package)
```

Generations share most of their store paths. Deleting generation 1 only frees paths that gen 2 and 3 do not reference.

### Checking Store Size

```bash
# Total store size:
du -sh /nix/store
# Typical: 20-50GB for a full desktop

# Number of store paths:
nix path-info --all | wc -l
# Typical: 10,000-30,000

# Size of a specific package's closure (everything it depends on):
nix path-info -rSh nixpkgs#firefox
# Shows total size including all dependencies

# What is taking the most space:
nix path-info --all -Sh | sort -t$'\t' -k2 -h | tail -20
# Shows the 20 largest store paths

# Btrfs compression ratio (if using btrfs with zstd):
sudo compsize /nix/store
# Nix store compresses well -- expect 1.5-2x ratio
```

### Manual Garbage Collection

```bash
# List system generations:
sudo nix-env --list-generations --profile /nix/var/nix/profiles/system
# Output:
#   1   2026-03-01   (current)
#   2   2026-03-04

# Delete generations older than 30 days:
sudo nix-collect-garbage --delete-older-than 30d

# Delete ALL old generations (only current survives):
sudo nix-collect-garbage -d
# WARNING: This removes your ability to rollback to any previous generation.

# Just run the store GC without touching generations:
nix store gc

# Optimize store (hardlink identical files to save space):
nix store optimise
# This is also done automatically if nix.settings.auto-optimise-store = true
```

### Automatic Garbage Collection (Configured in System)

The system config includes:

```nix
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 30d";
};
```

This creates a systemd timer that runs weekly and removes generations older than 30 days. Verify it is active:

```bash
systemctl list-timers nix-gc.timer
# Should show next scheduled run
```

### Boot Menu Generation Limit

```nix
boot.loader.systemd-boot.configurationLimit = 5;
```

This caps the systemd-boot menu at 5 entries. Old generations beyond this limit are not shown in the boot menu but still exist in the store until GC removes them.

## Generation Management

### Listing Generations

```bash
# System generations:
sudo nix-env --list-generations --profile /nix/var/nix/profiles/system

# Home Manager generations:
home-manager generations
```

### Rolling Back

```bash
# Roll back system to previous generation:
sudo nixos-rebuild switch --rollback

# Roll back to a specific generation (boot menu):
# Reboot and select the desired generation from systemd-boot menu.

# Roll back via config:
# Just `git revert` the last commit and rebuild:
cd ~/projects/nixos
git revert HEAD
sudo nixos-rebuild switch --flake .#framework
```

### Comparing Generations

```bash
# See what changed between current and previous generation:
nix store diff-closures \
  /nix/var/nix/profiles/system-$(( $(readlink /nix/var/nix/profiles/system | grep -o '[0-9]*') - 1 ))-link \
  /nix/var/nix/profiles/system

# This shows which packages were added, removed, or updated.
```

## devShell Recipes

### Flutter / Dart Project

```nix
# ~/projects/my-flutter-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let
      pkgs = import nixpkgs {
        system = "x86_64-linux";
        config.allowUnfree = true;
        config.android_sdk.accept_license = true;
      };
      androidComposition = pkgs.androidenv.composeAndroidPackages {
        platformVersions = [ "34" ];
        buildToolsVersions = [ "34.0.0" ];
        includeEmulator = false;
      };
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [
          flutter dart jdk21
          androidComposition.androidsdk
        ];
        ANDROID_HOME = "${androidComposition.androidsdk}/libexec/android-sdk";
        JAVA_HOME = "${pkgs.jdk21}";
        shellHook = ''flutter doctor'';
      };
    };
}
```

```bash
cd ~/projects/my-flutter-app
nix develop   # Flutter, Dart, Android SDK available
flutter run   # Build and run
exit          # All gone from PATH
```

### Node.js / Bun Project

```nix
# ~/projects/my-node-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ nodejs_22 bun nodePackages.typescript ];
      };
    };
}
```

### Python / uv Project

```nix
# ~/projects/my-ml-project/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ python313 uv ];
        shellHook = ''
          if [ ! -d .venv ]; then
            echo "Run 'uv sync' to create virtualenv"
          else
            source .venv/bin/activate
          fi
        '';
      };
    };
}
```

uv manages the Python virtualenv and PyPI dependencies. Nix just provides `python3` and `uv`. This is clean -- Nix handles the system-level toolchain, uv handles the Python ecosystem.

### Rust Project

```nix
# ~/projects/my-rust-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ rustup gcc pkg-config openssl ];
        # rustup manages toolchains; Nix provides system deps
        shellHook = ''
          rustup default stable 2>/dev/null
          echo "Rust $(rustc --version)"
        '';
      };
    };
}
```

### Go Project

```nix
# ~/projects/my-go-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ go gopls golangci-lint ];
        shellHook = ''echo "Go $(go version)"'';
      };
    };
}
```

### Java / Gradle Project

```nix
# ~/projects/my-java-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [ jdk21 gradle ];
        JAVA_HOME = "${pkgs.jdk21}";
        shellHook = ''echo "Java $(java --version 2>&1 | head -1)"'';
      };
    };
}
```

## direnv Integration (Automatic devShell Activation)

With `programs.direnv.enable = true` in Home Manager config:

```bash
# In any project with a flake.nix:
echo "use flake" > ~/projects/my-app/.envrc
direnv allow ~/projects/my-app

# Now, every time you `cd ~/projects/my-app`, the devShell activates.
# When you `cd` out, it deactivates. No manual `nix develop` needed.
```

nix-direnv caches the devShell, so subsequent activations are instant (no re-evaluation).

## Quick Reference

| Task | Command |
|---|---|
| Install system package | Edit `configuration.nix`, `sudo nixos-rebuild switch --flake .#framework` |
| Install user package | Edit `home.nix`, `sudo nixos-rebuild switch --flake .#framework` |
| Try package temporarily | `nix shell nixpkgs#package-name` |
| Run one-off command | `nix run nixpkgs#package-name -- args` |
| Project-specific tools | `nix develop` in project with `flake.nix` |
| Search for packages | `nix search nixpkgs#name` or search.nixos.org |
| Check store size | `du -sh /nix/store` |
| Garbage collect | `sudo nix-collect-garbage --delete-older-than 30d` |
| Rollback last change | `sudo nixos-rebuild switch --rollback` |
| List generations | `sudo nix-env --list-generations --profile /nix/var/nix/profiles/system` |
| Update all inputs | `nix flake update` |
| Update stable only | `nix flake update nixpkgs` |
| Update unstable only | `nix flake update nixpkgs-unstable` |
| Optimize store | `nix store optimise` |
