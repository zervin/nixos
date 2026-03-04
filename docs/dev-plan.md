# NixOS Migration Plan

## Executive Summary

This plan covers migrating a Framework 13 AMD 7840U laptop from Debian 13 (Trixie) with KDE Plasma 6 to NixOS with an equivalent (or improved) setup. The migration is structured in 6 phases, designed so the user can bail out at any phase boundary without data loss.

**Key findings:**
- Every component of the current Debian setup has a NixOS equivalent
- 3-4 applications require Flatpak (Betterbird, Speech Note, possibly GeForce Now, optionally Insync for KDE sessions)
- GNOME Online Accounts replaces Insync when using GNOME sessions (Google Drive + OneDrive native)
- Framework 13 AMD has first-class support via nixos-hardware
- A "one swoop" migration is feasible: boot NixOS installer, deploy config, restore data
- Citrix Workspace, Zoom, and ZoomVDI are work-critical and accounted for (Citrix via unstable overlay)

**Channel strategy:** Stable base (nixos-24.11, upgrading to 25.05 when released) with selective unstable overlay for packages with known stable-channel issues (Citrix Workspace).

## Feasibility Assessment

### Question 1: What does replicating this setup look like in NixOS?

It looks like a single Git repository containing ~10-15 Nix files that declaratively describe the entire system. Every package, service, configuration option, and desktop environment setting is expressed in code. The configuration is roughly 600-1000 lines of Nix across all files (slightly more than the previous estimate because of dual DE support and the overlay pattern).

Specifically:
- **Hardware**: One import line pulls in the nixos-hardware Framework 13 AMD module
- **Desktop**: Both KDE Plasma 6 and GNOME enabled simultaneously (~10 lines each)
- **Services**: Each service (Docker, Ollama, Tailscale, etc.) is 1-5 lines
- **Packages**: A list of ~50-60 package names in a `home.packages` block, with Citrix from unstable overlay
- **Flatpaks**: A list of ~4-5 Flatpak app IDs for apps not in nixpkgs
- **Cloud storage**: GNOME Online Accounts for Google/Microsoft in GNOME sessions; Insync Flatpak or rclone for KDE sessions

The result is a system that can be reproduced from scratch on any machine by running one command.

### Question 2: Is there a "one swoop" migration path?

Yes, with preparation. The process is:

1. Build and test the NixOS configuration in a VM (1-2 days of tinkering)
2. Back up all data from current Debian install (~/projects, ~/.ssh, ~/.gnupg, browser profiles, etc.)
3. Boot NixOS installer USB
4. Partition, encrypt, install using the pre-built configuration
5. Restore user data from backup
6. Re-authenticate services (Tailscale, browser logins, Syncthing pairing, GNOME Online Accounts, etc.)

Total downtime: 2-4 hours for the actual install. Total project time including VM testing: 1-2 weeks of evening sessions.

What you keep automatically (via declarative config):
- Every installed package
- Every running service and its configuration
- Both desktop environments, display manager, audio stack
- Shell configuration, git config, editor setup
- Garbage collection and store optimization settings

What you re-do manually:
- Log into accounts (browsers, Discord, Tailscale, etc.)
- Re-pair Syncthing devices
- Re-pull Ollama models
- Copy SSH/GPG keys from backup
- Add Google/Microsoft accounts in GNOME Online Accounts
- KDE desktop customizations (wallpaper, panel tweaks)
- Citrix/Zoom/Teams authentication

### Question 3: Is NixOS more sustainable long-term for semi-rolling updates?

**Yes, significantly.** Here is why:

| Factor | Debian Testing/Unstable | NixOS Stable + Unstable Overlay |
|---|---|---|
| Base system updates | Release-based (~2 years) | Release-based (~6 months) with unstable escape hatch |
| Critical app freshness | Whatever Debian packages | Unstable overlay for specific packages (Citrix, etc.) |
| Breakage recovery | Manual (apt, dpkg rescue) | Atomic rollback (boot menu or one command) |
| Update confidence | "Will this break X?" | "I can always roll back" |
| Config drift | Accumulates over time | Impossible (config is the source of truth) |
| Package creep | `apt autoremove` misses things | Declarative: if not in config, not on system |
| Fresh install | Hours of manual setup | Minutes (deploy existing config) |
| Reproducibility | Difficult | Exact (flake.lock) |

NixOS stable releases every 6 months (May and November) compared to Debian's ~2-year cycle. The unstable overlay ensures work-critical packages stay fresh without destabilizing the whole system. Combined with atomic rollback, it is genuinely safer than Debian.

**nixpkgs is the largest package repository in existence** (over 100,000 packages). Package availability is rarely an issue.

### Question 4: Can you switch DEs without reinstalling?

**Yes, trivially.** This is one of NixOS's strongest features, and the recommended config enables both KDE and GNOME simultaneously.

Both enabled at once (session picker at login):
```nix
services.desktopManager.plasma6.enable = true;
services.xserver.desktopManager.gnome.enable = true;
```

At the SDDM login screen, a dropdown offers "Plasma (Wayland)", "GNOME", or "GNOME on Xorg". Select one, log in. Your files are shared, your settings are per-DE.

To disable one entirely: set its `.enable = false`, run `sudo nixos-rebuild switch`, done. Previous generation with both DEs is still in the boot menu if you change your mind.

This works for any DE: GNOME, KDE, Hyprland, Sway, XFCE, Cinnamon, i3, etc. Mix and match freely.

## Package Creep Prevention

This is where NixOS provides its strongest advantage over Debian for long-term sustainability. On Debian, `apt install` leaves traces everywhere -- configs in `/etc`, libraries in `/usr/lib`, man pages, `.desktop` files -- and `apt autoremove` only catches direct dependency orphans, not the accumulated cruft from years of installs and removes.

NixOS eliminates this problem structurally.

### The Core Principle: Declarative = No Hidden Cruft

If a package is not declared in `configuration.nix` or `home.nix`, it does not exist after a rebuild. There is no "install once, forget, and it lingers forever." Removing a package means deleting one line and rebuilding.

### The Three Contexts for Installing Software

**1. System packages** (`configuration.nix` -- requires sudo to rebuild):
```nix
# For system-wide tools, services, drivers
environment.systemPackages = with pkgs; [ vim git htop ];
```
Use for: system utilities, anything all users need, anything requiring system-level integration (CUPS, Docker, etc.)

**2. User packages** (`home.nix` via Home Manager):
```nix
# For user applications and dev tools
home.packages = with pkgs; [ vscode brave obsidian ];
```
Use for: applications you personally use, dev tools, editors, browsers.

**3. Project packages** (`flake.nix` in each project directory -- the anti-creep superpower):
```nix
# In ~/projects/my-flutter-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }: {
    devShells.x86_64-linux.default = nixpkgs.legacyPackages.x86_64-linux.mkShell {
      packages = with nixpkgs.legacyPackages.x86_64-linux; [
        flutter
        dart
        android-studio
        jdk21
      ];
    };
  };
}
```
Use for: project-specific toolchains. `cd` into the project, run `nix develop`, get exactly the tools that project needs. Exit the shell, they are gone. **No global pollution.**

### The Golden Rule: Never Use `nix-env -i`

`nix-env -i <package>` is the NixOS equivalent of `apt install` -- it is imperative and creates the same creep problem. Packages installed via `nix-env` are:
- Not tracked in any config file
- Not visible to `nixos-rebuild`
- A source of "works on my machine but not on a fresh install" bugs

**Always edit a `.nix` file instead.** The only acceptable imperative install command is `nix shell nixpkgs#<package>` for ephemeral one-off use (see below).

### Ephemeral Package Use (Zero Creep)

Need a package for 5 minutes? Do not install it:

```bash
# Run a command with a package, then it is gone:
nix run nixpkgs#cowsay -- "Hello NixOS"

# Drop into a shell with multiple packages:
nix shell nixpkgs#imagemagick nixpkgs#ffmpeg
# Use them, then exit the shell. Nothing persists.
```

### Garbage Collection

Even with declarative config, the Nix store accumulates old builds. This is by design (it enables rollback), but needs periodic cleanup.

**Automatic GC (configured in system):**
```nix
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 30d";
};
```

**Manual GC commands:**
```bash
# See how big the store is:
du -sh /nix/store

# Count store paths:
nix path-info --all | wc -l

# Delete generations older than 30 days:
sudo nix-collect-garbage --delete-older-than 30d

# Delete ALL old generations (keeps only current):
sudo nix-collect-garbage -d

# Optimize store (hardlink identical files):
nix store optimise
```

**Boot menu generation limit:**
```nix
boot.loader.systemd-boot.configurationLimit = 5;
```
This caps boot menu entries at 5 most recent generations. Old generations are still in the store until GC removes them, but the boot menu stays clean.

### The Contract: Big Store, Full Control

The Nix store is large. 20-50GB is normal for a fully configured desktop. This is because:
- Each `nixos-rebuild` creates a new "generation" that references store paths
- Multiple generations share paths (deduplication via hardlinks)
- Old generations keep their paths alive until GC

But unlike Debian's `/usr/lib` and `/etc` sprawl:
- Every byte in `/nix/store` is referenced by a known derivation
- `nix-collect-garbage` removes ALL unreferenced paths -- no orphans, no dangling configs
- `nix store optimise` hardlinks identical content across paths, reclaiming significant space
- You can audit exactly what is in your system: `nix flake show`, `nix flake check`, `nixos-rebuild build --dry-run`

### devShells: The Single Biggest Anti-Creep Tool

For a developer who works with Flutter, Node.js, Python, Rust, Go, and Java -- devShells are transformative. Instead of installing every toolchain globally:

```nix
# ~/projects/my-node-app/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  outputs = { self, nixpkgs }:
    let pkgs = nixpkgs.legacyPackages.x86_64-linux;
    in {
      devShells.x86_64-linux.default = pkgs.mkShell {
        packages = with pkgs; [
          nodejs_22
          bun
          nodePackages.typescript
        ];
        shellHook = ''
          echo "Node $(node --version) | Bun $(bun --version)"
        '';
      };
    };
}
```

```bash
cd ~/projects/my-node-app
nix develop
# Now you have node 22, bun, and typescript.
# Your system has NONE of these installed globally.
```

Different projects can use different Node versions, different Python versions, different everything -- without conflict, without pollution, without `nvm` or `pyenv` or `asdf`.

**direnv integration** (optional but recommended): With `nix-direnv`, entering a project directory automatically activates its devShell. No manual `nix develop` needed.

## Migration Phases

### Phase 0: Preparation (Before touching hardware)

**Duration:** 1-2 evenings
**Risk:** None (current system untouched)

Tasks:
1. Learn Nix language basics (1-2 hours)
   - Recommended: https://nix.dev/tutorials/nix-language
   - Key concepts: attribute sets, let bindings, imports, `with`, `rec`
2. Set up the flake repository structure (this repo)
3. Write the initial `flake.nix` with dual nixpkgs inputs (stable + unstable)
4. Write module files for hardware, desktop (KDE + GNOME), services, packages
5. Read nixos-hardware Framework 13 AMD documentation
6. Understand `nixos-rebuild switch` vs `nixos-rebuild test` vs `nixos-rebuild boot`

**Deliverable:** Complete NixOS configuration files in this Git repo

### Phase 1: VM Validation

**Duration:** 2-3 evenings
**Risk:** None (current system untouched)

Tasks:
1. Download NixOS minimal ISO (24.11 stable)
2. Create a VM (QEMU/virt-manager or VirtualBox)
3. Install NixOS in VM using the configuration from this repo
4. Verify:
   - KDE Plasma 6 boots and works
   - GNOME session is available at SDDM login
   - GNOME Online Accounts can add a Google account
   - All declared packages are installed
   - Citrix Workspace installs from unstable overlay
   - Zoom installs and launches
   - **ZoomVDI plugin -- critical to test here.** Check if `zoom-us` includes VDI or if a separate package is needed.
   - Services start correctly (Docker, Ollama, Tailscale stubs)
   - Flatpak integration works
   - Home Manager applies user config
5. Iterate on config until VM matches expectations
6. Test `nixos-rebuild switch` workflow
7. Test DE switching (log into KDE, log out, log into GNOME, verify both)
8. Test garbage collection: run a few rebuilds, then `nix-collect-garbage -d`

**Deliverable:** Validated configuration that builds and boots in a VM, with ZoomVDI status confirmed

### Phase 2: Data Backup

**Duration:** 1 evening
**Risk:** Low (just making backups)

Tasks:
1. Full backup of home directory to external drive or Syncthing target:
   ```bash
   # Critical directories
   ~/projects/          # All code and projects
   ~/.ssh/              # SSH keys
   ~/.gnupg/            # GPG keys
   ~/.config/           # Application configs (KDE, editors, etc.)
   ~/.local/share/      # Application data
   ~/Documents/         # Personal files
   ~/Pictures/          # Media
   ~/Downloads/         # Downloads
   ```
2. Export browser bookmarks (Firefox, Brave, Chrome)
3. Note down Syncthing device IDs (or screenshot Syncthing web UI)
4. Export KDE settings if desired (`~/.config/k*`, `~/.config/plasma*`)
5. List of Flatpak app data to preserve: `~/.var/app/`
6. Verify backup integrity (spot check key files)
7. Note Citrix Workspace configuration (server URLs, store config)

**Deliverable:** Complete backup on external storage, verified

### Phase 3: Installation

**Duration:** 2-4 hours
**Risk:** Medium (wiping current OS, mitigated by backup)

Tasks:
1. Boot NixOS minimal ISO from USB
2. Partition the NVMe (LUKS2 + btrfs, see runbook for exact commands)
3. Generate hardware config with `nixos-generate-config`
4. Clone this config repo and install with `nixos-install --flake .#framework`
5. Reboot into NixOS

**Deliverable:** Booting NixOS system with KDE Plasma + GNOME available at login

### Phase 4: Data Restoration and Service Setup

**Duration:** 1-2 evenings
**Risk:** Low

Tasks:
1. Restore home directory data from backup
2. Authenticate services:
   - `sudo tailscale up` -- authenticate with Tailscale
   - Log into browsers (Firefox, Brave, Chrome) -- sync restores bookmarks/passwords
   - Log into Discord, Zoom, Teams, Citrix
   - Configure Syncthing device pairing
   - **GNOME Online Accounts**: Add Google and Microsoft accounts via GNOME Settings > Online Accounts
3. Install remaining Flatpak apps (if not handled by nix-flatpak)
4. Pull Ollama models
5. Verify all services are running
6. Test key workflows (see runbook checklist)

**Deliverable:** Fully functional system matching previous Debian setup

### Phase 5: Polish and Optimize

**Duration:** Ongoing over first week
**Risk:** None

Tasks:
1. KDE customization (wallpaper, panel, shortcuts, themes)
2. GNOME customization (extensions, appearance)
3. Fine-tune power management for battery life
4. Configure garbage collection (already in config, verify it runs)
5. Set up devShells for active projects:
   - Create `flake.nix` in each project directory
   - Test `nix develop` workflow
   - Optionally set up `direnv` + `nix-direnv` for automatic activation
6. Consider plasma-manager for declarative KDE settings
7. Commit final working config to Git
8. Move project-specific toolchains out of `home.packages` into devShells

**Deliverable:** Optimized, polished NixOS installation with config committed to Git

## Timeline Estimate

| Phase | Duration | Cumulative |
|---|---|---|
| Phase 0: Preparation | 1-2 evenings | 1-2 evenings |
| Phase 1: VM Validation | 2-3 evenings | 3-5 evenings |
| Phase 2: Data Backup | 1 evening | 4-6 evenings |
| Phase 3: Installation | 2-4 hours (weekend) | ~1 week elapsed |
| Phase 4: Restoration | 1-2 evenings | ~1.5 weeks elapsed |
| Phase 5: Polish | Ongoing | ~2 weeks to "done" |

## Rollback Plan

If NixOS is not working out after installation:
1. The Debian backup is intact on external storage
2. Boot a Debian installer USB
3. Re-partition and reinstall Debian
4. Restore data from backup

This is the nuclear option. Within NixOS, you have much easier rollback:
- `sudo nixos-rebuild switch --rollback` undoes the last rebuild
- Boot menu shows previous system generations (up to `configurationLimit`)
- `git revert` on your config and rebuild restores any previous state

## Known Gotchas and Mitigations

### 1. Insync / Cloud Storage
**Problem:** Insync is proprietary and not in nixpkgs. Used for Google Drive + OneDrive.
**Mitigation:** In GNOME sessions, GNOME Online Accounts provides native Google Drive and OneDrive integration directly in the file manager and calendar. No Insync needed. In KDE sessions, use Flatpak Insync or rclone with a systemd user service for cloud mounts. This is not a migration blocker.

### 2. Citrix Workspace Version Lag
**Problem:** Citrix Workspace on stable nixpkgs has historically had libwebkitgtk2 dependency issues and version lag.
**Mitigation:** Pull `citrix_workspace` from the unstable overlay (`pkgs-unstable.citrix_workspace`). This ensures the latest version with correct dependencies. Requires `allowUnfree = true` and license acceptance. This is work-critical -- test thoroughly in Phase 1 VM.

### 3. ZoomVDI Plugin
**Problem:** The current system has `zoomvdi-universal-plugin`. This may or may not be packaged in nixpkgs.
**Mitigation:** Investigate during Phase 1 VM testing. Check `nix search nixpkgs#zoomvdi`. If not available, options are: (a) check if `zoom-us` already includes VDI support, (b) wrap the upstream `.deb`/`.tar` in `buildFHSUserEnv`, (c) Flatpak. This is a validation item for Phase 1.

### 4. Flutter Development
**Problem:** Flutter expects FHS-compliant directory structure. NixOS is not FHS-compliant.
**Mitigation:** Use a devShell with `flutter` package from nixpkgs, which wraps Flutter in an FHS-compatible environment. See the devShell example in the configuration skeleton. Do NOT install Flutter globally -- use per-project devShells.

### 5. VS Code Insiders
**Problem:** VS Code Insiders is not a standard nixpkgs package.
**Mitigation:** Use the `vscode` package. If Insiders-specific features are needed, install via the `.tar.gz` release with a custom derivation or use the Flatpak. Most users do not need both.

### 6. Ollama Models
**Problem:** Ollama models are not part of the system config; they are runtime data.
**Mitigation:** After install, re-pull models with `ollama pull <model>`. These are cloud-proxied, so this is fast. Models persist in `/var/lib/ollama` across rebuilds.

### 7. LUKS Password Entry at Boot
**Problem:** NixOS needs to know about the LUKS device to prompt for password at boot.
**Mitigation:** `hardware-configuration.nix` (generated by `nixos-generate-config`) automatically detects LUKS and adds the correct `boot.initrd.luks.devices` entry. Verify this is present after generation.

### 8. Steam and Gaming
**Problem:** Steam requires FHS environment and 32-bit libraries.
**Mitigation:** NixOS has excellent Steam support via `programs.steam.enable = true`, which handles all the FHS sandboxing and 32-bit Mesa/Vulkan automatically.

### 9. Nvidia Services in Current System
**Problem:** The current system has nvidia-hibernate/resume/suspend services, but the Framework 13 AMD has no Nvidia GPU.
**Mitigation:** These are leftover from a previous system or eGPU. Do not port to NixOS. The AMD Radeon 780M is handled entirely by `amdgpu` and Mesa.

### 10. KDE PIM (Akonadi, Kontact, Merkuro)
**Problem:** Akonadi can be finicky on any distro.
**Mitigation:** Consider using Evolution in GNOME sessions instead (better integration with Online Accounts). If KDE PIM is needed, install via `kdePackages.kontact`, `kdePackages.merkuro`. Email accounts need manual re-setup.

### 11. Dual DE XDG Portal Conflicts
**Problem:** KDE and GNOME each want their own XDG desktop portal implementation. Having both can cause confusion about which portal handles file dialogs, screen sharing, etc.
**Mitigation:** Configure `xdg.portal.config` explicitly (see architecture.md). Each DE session should prefer its own portal. In practice this works well because the portal implementations check `$XDG_CURRENT_DESKTOP`.

### 12. Package Creep from nix-env
**Problem:** New NixOS users often reach for `nix-env -i` out of habit, creating the same imperative package sprawl as `apt install`.
**Mitigation:** Never use `nix-env -i`. Use `nix shell` for ephemeral needs, devShells for projects, and declarative config for everything persistent. See the "Package Creep Prevention" section above for the full discipline.
