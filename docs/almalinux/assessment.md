# AlmaLinux 10 Migration Assessment

## Overview

This document evaluates AlmaLinux 10 as a migration target from Debian 13 (Trixie) on a Framework 13 AMD 7840U laptop. It is a parallel assessment to the existing NixOS evaluation, intended to give a fair comparison for a developer who has prior positive experience with AlmaLinux.

AlmaLinux 10 is a 1:1 binary-compatible rebuild of Red Hat Enterprise Linux 10. It inherits RHEL 10's 10-year support lifecycle, enterprise stability focus, and the entire RHEL ecosystem of certifications and third-party vendor support.

## AlmaLinux 10 / RHEL 10 Release Status

**RHEL 10 GA:** June 10, 2025.

**AlmaLinux 10 GA:** June 2025. AlmaLinux historically tracks RHEL GA releases within days to weeks. AlmaLinux 10 is expected to be fully available and stable by mid-2025, and by March 2026 (current date) it has been GA for approximately 9 months.

**RHEL 10 key specs:**
- Kernel: 6.12 LTS
- GCC: 15
- Python: 3.12 (system), 3.13 available as module stream
- Node.js: 22 LTS as module stream
- GNOME: 47
- systemd: 257
- DNF5 (replaces DNF4 as default package manager)
- Wayland-only GNOME (X11 session dropped from GNOME in RHEL 10)

**Lifecycle:**
- Full support: 5 years (until ~2030)
- Maintenance support: 5 additional years (until ~2035)
- During maintenance phase, only critical security fixes; no new features or hardware enablement

**What this means for a developer:**
The base system packages are frozen at mid-2025 versions for the life of the release. Python 3.12 will remain the system Python for 10 years. Mesa, the kernel, and GNOME receive only backported security fixes after GA. This is the fundamental trade-off of RHEL: rock-solid stability at the cost of freshness. For a desktop developer, this means heavy reliance on EPEL, Flatpak, third-party repos, and toolchain-specific installers (rustup, nvm, etc.) to get current versions.

## Pros and Cons for This User Profile

### Profile Summary
- Developer (Node.js, Python, Rust, Java, Flutter, Go, containers)
- KDE Plasma preference (fractional scaling on Framework 13 display)
- Framework 13 AMD 7840U laptop
- Active tinkerer who worries about package creep
- Work-critical: Citrix Workspace, Zoom, ZoomVDI
- Prior positive experience with AlmaLinux

### Pros

**1. Native RPM support for work-critical tools.**
Citrix Workspace, Zoom, and ZoomVDI all provide official RPM packages tested against RHEL. This is the most friction-free path for these three applications. No wrappers, no FHS shims, no hoping someone packaged it in nixpkgs. Download the RPM, install it, done.

**2. Familiar and well-documented ecosystem.**
The RHEL/CentOS/Alma ecosystem has decades of documentation, Stack Overflow answers, and vendor support. When Citrix publishes installation instructions, they publish them for RHEL. When a vendor certifies their software, they certify it on RHEL. This removes an entire class of "will it work" questions.

**3. SELinux is mature and enforcing by default.**
RHEL's security story is arguably the strongest in the Linux world. SELinux policies are well-tested, and most RPM packages ship with proper SELinux contexts. This is superior to Debian's AppArmor and to NixOS's optional AppArmor in terms of out-of-the-box security posture.

**4. Podman is a first-class citizen.**
Podman is developed by Red Hat and deeply integrated into the RHEL ecosystem. Rootless containers, systemd integration via Quadlet, and `podman-compose` work out of the box. For container-heavy development, this is excellent.

**5. 10-year support lifecycle.**
No forced distribution upgrades every 2 years. Security patches keep flowing. If the system works after initial setup, it will keep working for years without major disruption.

**6. Modern kernel (6.12) at release.**
RHEL 10's kernel 6.12 includes full support for AMD 7840U, Radeon 780M, and Framework 13 hardware. This is not a concern like it was with RHEL 8/9 on newer hardware.

**7. DNF5 is a significant improvement.**
DNF5 is faster than DNF4, uses less memory, and has a cleaner CLI. It also has built-in support for modularity streams, which helps manage multiple versions of Node.js, Python, etc.

### Cons

**1. KDE is NOT in RHEL 10 base repos.**
This is the single biggest friction point. RHEL/AlmaLinux ships GNOME as the only desktop environment in base repos. Getting KDE requires:
- The Fedora KDE SIG packages for EPEL (availability for EL10 needs verification)
- Or EPEL 10 KDE packages (if they exist)
- Or building from source (impractical)

As of early 2026, the KDE SIG for EPEL 10 may or may not have full KDE Plasma 6 packages. This is a real risk that needs validation before committing to AlmaLinux. If KDE is not available, you are limited to GNOME 47 -- which is capable but lacks the fractional scaling flexibility you value.

**UPDATE NOTE:** Check https://sig.fedoraproject.org/kde and https://packages.fedoraproject.org/ for current EPEL 10 KDE package status before proceeding.

**2. Frozen base packages hurt developer workflows.**
The system Python is 3.12 for the life of the release. Mesa stays at whatever version shipped with RHEL 10. The system Node.js is 22 LTS. For a developer who works across multiple language versions and wants current tooling, you will be relying on:
- `rustup` for Rust (same as everywhere)
- NodeSource or `fnm`/`nvm` for Node.js
- `pyenv` or `uv` for Python versions
- Manual Flutter SDK installs
- Flatpak for GUI apps that need newer versions
- EPEL for supplementary packages

This is not necessarily worse than Debian -- Debian 13 also freezes packages -- but it is substantially worse than NixOS, where `nix flake update` gives you current versions of everything.

**3. RPM Fusion for EL10 may have gaps.**
RPM Fusion is critical for multimedia codecs, Steam, VLC, and some other packages. RPM Fusion historically lags behind new EL releases. As of 9 months post-RHEL 10 GA, RPM Fusion free and nonfree repos for EL10 should be available, but some packages may still be missing or untested. Check https://rpmfusion.org/ for current status.

**4. Package creep is the traditional RPM problem.**
This is the core concern that motivated the NixOS evaluation. On AlmaLinux:
- You will have 6-10 active repositories (base, CRB, EPEL, RPM Fusion free, RPM Fusion nonfree, plus 5+ third-party repos)
- `dnf5 autoremove` does not reliably clean up all orphaned dependencies
- There is no declarative package manifest -- the system state is the sum of every `dnf install` ever run
- Mixing RPM + Flatpak + manual installs + language-specific installers makes the full system state hard to audit
- No atomic rollback (Btrfs snapshots via Snapper can partially mitigate)

Compare this to NixOS where the entire system state is a single `flake.nix` file and `nix-collect-garbage` actually works.

**5. No atomic rollback.**
If a `dnf update` breaks something, you cannot `dnf rollback` to the previous state (DNF5 has experimental transaction history replay, but it is not reliable for complex upgrades). Your safety net is Btrfs snapshots, which must be set up and maintained separately. This works, but it is a manual discipline compared to NixOS's automatic generation management.

**6. EPEL 10 is still maturing.**
EPEL (Extra Packages for Enterprise Linux) for version 10 will have fewer packages than EPEL 9 initially. Some packages you expect may not be there yet. This improves over time but is a real concern for early adopters.

**7. Wayland-only GNOME in RHEL 10.**
RHEL 10 drops the GNOME X11 session. This is fine for most use cases (Wayland is mature in 2026), but some legacy applications or remote desktop tools may have issues. Citrix Workspace on Wayland needs verification.

## Package Ecosystem Map

AlmaLinux 10 packages come from multiple layers. Understanding which layer provides what is critical for managing the system.

```
Layer 0: AlmaLinux 10 Base (BaseOS + AppStream)
  |-- Core system: kernel, systemd, NetworkManager, GNOME 47, Mesa, Pipewire
  |-- System Python 3.12, basic dev tools (gcc, make, cmake)
  |-- Podman, Buildah, Skopeo (container tools)
  |-- LibreOffice, Firefox ESR, Thunderbird
  |-- CUPS, Bluetooth, fwupd, lm-sensors
  |
Layer 1: CRB (CodeReady Builder)
  |-- Development headers and libraries not in AppStream
  |-- Required dependency for many EPEL packages
  |
Layer 2: EPEL 10
  |-- Community packages: Syncthing, htop, neofetch, many CLI tools
  |-- Potentially KDE Plasma (via KDE SIG -- verify availability)
  |-- GIMP (if not in base), Inkscape, additional dev tools
  |
Layer 3: RPM Fusion (free + nonfree)
  |-- Multimedia codecs (ffmpeg, x264, x265)
  |-- Steam (nonfree)
  |-- VLC (free)
  |-- Additional Mesa drivers (mesa-va-drivers-freeworld)
  |
Layer 4: Third-Party Vendor RPM Repos
  |-- Brave Browser (brave-browser-rpm-release)
  |-- VS Code / VS Code Insiders (packages.microsoft.com)
  |-- Docker CE (download.docker.com)
  |-- Tailscale (pkgs.tailscale.com)
  |-- Google Cloud SDK (packages.cloud.google.com)
  |-- GitHub CLI (cli.github.com)
  |-- Azul Zulu JDK (repos.azul.com)
  |-- NodeSource (rpm.nodesource.com)
  |-- Citrix Workspace (direct RPM download)
  |-- Zoom + ZoomVDI (direct RPM download)
  |-- Insync (direct RPM download or yum.insync.io)
  |
Layer 5: Flatpak (Flathub)
  |-- Discord, Obsidian, Logseq, Bambu Studio
  |-- Betterbird, Speech Note, GeForce Now
  |-- Inkscape, Upscayl, KTailctl, Nicotine+
  |-- Any app where the RPM version is too old or unavailable
  |
Layer 6: Manual Installs (not managed by any package manager)
  |-- Zed editor (AppImage)
  |-- Bun (official install script to ~/.bun)
  |-- uv (official install script)
  |-- Flutter/Dart SDK (manual in ~/projects/flutter)
  |-- Rust/Cargo (rustup)
  |-- Ollama (official install script, installs systemd service)
```

**Total active repositories:** ~12-15 (base + CRB + EPEL + RPM Fusion x2 + ~8 third-party)

This is a significant number of repositories to manage. Each one is an independent update source with its own release cadence and potential for conflicts.

## Complete Package Mapping

### Work-Critical (Non-Negotiable)

| Application | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Risk Level |
|---|---|---|---|---|
| Citrix Workspace | .deb from Citrix | RPM from Citrix | `ICAClient-*.rpm` direct download | **Low** -- official RPM, tested on RHEL |
| Zoom | .deb from Zoom | RPM from Zoom | `zoom-*.rpm` direct download | **Low** -- official RPM |
| ZoomVDI Plugin | .deb from Zoom | RPM from Zoom | `zoomvdi-*.rpm` direct download | **Low** -- official RPM |

This is where AlmaLinux shines. All three work-critical applications have official RPMs built for RHEL. No wrappers, no hoping for nixpkgs packaging.

### Browsers

| Application | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Notes |
|---|---|---|---|---|
| Brave | brave-browser apt repo | Brave RPM repo | `brave-browser` | Official repo |
| Firefox | Debian base | AlmaLinux base | `firefox` | ESR version, frozen |
| Google Chrome | google-chrome apt repo | google-chrome RPM repo | `google-chrome-stable` | Official repo |

### Editors and IDEs

| Application | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Notes |
|---|---|---|---|---|
| VS Code | Microsoft apt repo | Microsoft RPM repo | `code` | Official repo |
| VS Code Insiders | Microsoft apt repo | Microsoft RPM repo | `code-insiders` | Official repo |
| Zed | AppImage / deb | AppImage | Download from zed.dev | Not in any RPM repo |

### Development Tools

| Application | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Notes |
|---|---|---|---|---|
| Docker CE | docker apt repo | Docker RPM repo | `docker-ce docker-ce-cli containerd.io` | Official repo |
| Podman | Debian base | AlmaLinux base | `podman` | Native, first-class |
| Node.js | NodeSource apt | NodeSource RPM or AppStream module | `nodejs:22` module or NodeSource | Module streams available |
| Python 3 | Debian base | AlmaLinux base | `python3` (3.12) | Frozen at 3.12; use `uv` for other versions |
| Rust/Cargo | rustup | rustup | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` | Same everywhere |
| Go | Debian base or manual | AlmaLinux base or EPEL | `golang` | May be slightly old; use go.dev install for latest |
| Java Zulu JDK 21 | Azul apt repo | Azul RPM repo | `zulu21-jdk` | Official Azul repo |
| Flutter/Dart | Manual ~/projects/flutter | Manual ~/projects/flutter | Manual install | Same everywhere |
| Bun | Official script | Official script | `curl -fsSL https://bun.sh/install \| bash` | Same everywhere |
| uv | Official script | Official script | `curl -LsSf https://astral.sh/uv/install.sh \| sh` | Same everywhere |
| Git | Debian base | AlmaLinux base | `git` | Available |
| GitHub CLI | gh apt repo | GitHub RPM repo | `gh` | Official repo |
| Google Cloud SDK | google-cloud-sdk apt repo | Google RPM repo | `google-cloud-cli` | Official repo |

### System Services

| Service | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Notes |
|---|---|---|---|---|
| Tailscale | Tailscale apt repo | Tailscale RPM repo | `tailscale` | Official repo |
| Syncthing | Debian base or Syncthing apt repo | EPEL 10 or Flatpak | `syncthing` | Check EPEL availability |
| Ollama | Official script | Official script | `curl -fsSL https://ollama.com/install.sh \| sh` | Same everywhere |
| CUPS | Debian base | AlmaLinux base | `cups` | Available |
| fwupd | Debian base | AlmaLinux base | `fwupd` | Available, Framework support |
| Bluetooth | Debian base | AlmaLinux base | `bluez` | Available |
| NetworkManager | Debian base | AlmaLinux base | `NetworkManager` | Default |
| Pipewire | Debian base | AlmaLinux base | `pipewire wireplumber` | Default audio stack |
| power-profiles-daemon | Debian base | AlmaLinux base | `power-profiles-daemon` | Available |
| lm-sensors | Debian base | AlmaLinux base | `lm_sensors` | Available |

### Desktop Applications

| Application | Debian 13 Source | AlmaLinux 10 Source | Package/Method | Notes |
|---|---|---|---|---|
| KDE Connect | Debian base | EPEL or KDE SIG | `kdeconnect` | **Depends on KDE availability** |
| Syncthing Tray (KDE) | Debian/Flatpak | EPEL or build from source | `syncthingtray` | **May not be in EPEL 10** |
| Steam | Steam apt repo | RPM Fusion nonfree | `steam` | **Depends on RPM Fusion EL10** |
| VLC | Debian base | RPM Fusion free | `vlc` | **Depends on RPM Fusion EL10** |
| GIMP | Debian base | AlmaLinux base or Flatpak | `gimp` or Flatpak | Base may have older version |
| LibreOffice | Debian base | AlmaLinux base | `libreoffice` | Available |
| Thunderbird | Debian base | AlmaLinux base | `thunderbird` | Available |
| Betterbird | Flatpak | Flatpak | `eu.betterbird.Betterbird` | Flatpak only |
| Discord | Flatpak | Flatpak | `com.discordapp.Discord` | Flatpak only |
| Obsidian | Flatpak | Flatpak | `md.obsidian.Obsidian` | Flatpak only |
| Logseq | Flatpak | Flatpak | `com.logseq.Logseq` | Flatpak only |
| Bambu Studio | Flatpak | Flatpak | `com.bambulab.BambuStudio` | Flatpak only |
| Inkscape | Debian base | Flatpak or EPEL | `org.inkscape.Inkscape` | Flatpak for latest |
| Upscayl | Flatpak | Flatpak | `org.upscayl.Upscayl` | Flatpak only |
| Speech Note | Flatpak | Flatpak | `net.mkiol.SpeechNote` | Flatpak only |
| GeForce Now | Flatpak | Flatpak | Flatpak or browser | Flatpak if available |
| KTailctl | Flatpak | Flatpak | `org.fkoehler.KTailctl` | Flatpak; **needs KDE** |
| Nicotine+ | Flatpak | Flatpak | `org.nicotine_plus.Nicotine` | Flatpak |
| Insync | Insync deb repo | Insync RPM repo | `insync` from yum.insync.io | **Verify EL10 support** |

### Packages with Availability Uncertainty on AlmaLinux 10

These need verification before committing to AlmaLinux 10:

| Package | Concern | Mitigation |
|---|---|---|
| KDE Plasma 6 | Not in RHEL base; KDE SIG/EPEL status for EL10 unknown | Use GNOME; add KDE if/when available |
| RPM Fusion packages (Steam, VLC, codecs) | RPM Fusion EL10 repos may be incomplete | Flatpak alternatives for Steam/VLC; codecs via alternative sources |
| Syncthing | May not be in EPEL 10 yet | Flatpak or official Syncthing repo |
| Syncthing Tray (KDE) | Unlikely in any EL10 repo | Build from source or go without |
| KDE Connect | Depends on KDE availability | GNOME alternative: GSConnect extension |
| KTailctl | KDE-specific, Flatpak only | Flatpak; needs KDE runtime |
| Insync | EL10 RPM may not exist yet | Use GNOME Online Accounts for Google/OneDrive; rclone as backup |

## KDE on AlmaLinux 10

This deserves its own section because it is the user's preferred desktop environment and the biggest risk factor for this migration.

### The Situation

RHEL (and by extension AlmaLinux) ships only GNOME as a desktop environment. KDE has never been part of the RHEL base repos. On RHEL 7/8/9, KDE was available via EPEL. For RHEL 10 / EL10:

**Option 1: KDE SIG packages via EPEL 10**
The Fedora KDE SIG maintains KDE packages for EPEL. Whether these are available for EL10 depends on the SIG's timeline. Check:
```bash
dnf5 search plasma-desktop
dnf5 search kde-plasma
# Or check EPEL 10 package list:
# https://packages.fedoraproject.org/
```

**Option 2: GNOME 47 (default)**
GNOME 47 on RHEL 10 is mature, supports Wayland, and has fractional scaling. GNOME Online Accounts provides native Google Drive and OneDrive integration -- which actually simplifies the Insync situation. The fractional scaling in GNOME 47 is functional but less flexible than KDE Plasma 6's per-monitor scaling.

**Option 3: Fedora as a Workstation Alternative**
If KDE is a hard requirement, Fedora KDE Spin is a better choice than AlmaLinux. But that changes the stability profile entirely (6-month release cycle, not 10-year support).

### Recommendation for KDE on AlmaLinux 10

Before committing to AlmaLinux 10, verify KDE Plasma availability:

```bash
# On a test VM or container:
dnf5 install -y epel-release
dnf5 search plasma-desktop
dnf5 search kf6-kwindowsystem
```

If KDE Plasma 6 is not available in EPEL 10, your options are:
1. Use GNOME 47 (it is good, especially with GNOME Online Accounts)
2. Wait for KDE SIG to catch up
3. Choose a different distribution

If KDE IS available:
```bash
# Install KDE Plasma alongside GNOME
dnf5 install @kde-desktop-environment
# Or if the group is named differently in EL10:
dnf5 group install "KDE Plasma Workspaces"

# Set SDDM as the display manager (KDE's default)
systemctl disable gdm
systemctl enable sddm
```

Both GNOME and KDE can coexist. SDDM or GDM can serve as the display manager with session selection at login.

### Fractional Scaling

On the Framework 13 (2256x1504, 13.5"):

**GNOME 47:** Fractional scaling via Settings > Displays. Supports 100%, 125%, 150%, 175%, 200%. Uses Wayland fractional-scale protocol. Works but some apps may be blurry (XWayland apps).

**KDE Plasma 6 (if available):** Per-monitor fractional scaling with finer granularity. Generally considered superior for mixed-DPI setups and non-standard scaling factors. The Framework 13 display works best at ~150% scale.

## Update Cadence Reality Check

### What Freezes

After RHEL 10 GA, these base system components receive only security backports, not version upgrades:

| Component | RHEL 10 Version | Frozen For |
|---|---|---|
| Kernel | 6.12 | 10 years (backported security/driver fixes) |
| Python | 3.12 | 10 years |
| Mesa | ~24.x | 10 years |
| GNOME | 47 | 10 years |
| GCC | 15 | 10 years |
| systemd | 257 | 10 years |
| Node.js | 22 LTS (module) | Until module EOL, then 24 LTS added |

### What Stays Current

| Component | Update Source | Cadence |
|---|---|---|
| Brave, Chrome | Their own RPM repos | Rolling (browser auto-updates) |
| VS Code | Microsoft RPM repo | Monthly |
| Docker CE | Docker RPM repo | Rolling |
| Tailscale | Tailscale RPM repo | Rolling |
| Flatpak apps | Flathub | Per-app rolling updates |
| Citrix Workspace | Citrix RPM | Per Citrix release |
| Zoom | Zoom RPM | Per Zoom release |
| Rust (via rustup) | rustup | Rolling |
| Node.js (via NodeSource) | NodeSource RPM | Rolling |
| Bun | bun upgrade | Rolling |
| uv | uv self-update | Rolling |

### Comparison to Debian 13

Debian 13 also freezes packages at release, but with a ~2-year release cycle instead of 10 years. This means Debian packages are at most ~2 years old before the next release. AlmaLinux packages can be up to 10 years old. In practice, both require the same third-party repo and Flatpak strategy for keeping developer tools current -- AlmaLinux just requires it more aggressively and for a longer period.

### Comparison to NixOS

NixOS stable freezes for ~6 months, and `nixos-unstable` provides near-rolling updates. With the stable + unstable overlay pattern described in the NixOS evaluation, you get 95% stable system with cherry-picked fresh packages. AlmaLinux cannot replicate this -- you either use the frozen base package or bypass it entirely with a third-party source.

## Package Creep Risk Assessment

### The Problem on AlmaLinux

Package creep is the gradual accumulation of unnecessary packages, orphaned dependencies, and forgotten manual installs that makes a system harder to understand, maintain, and reproduce over time.

On AlmaLinux 10, the package creep vectors are:

**1. Multiple repository sources with independent lifecycles.**
With 12-15 active repos, each `dnf5 install` pulls dependencies from whichever repo provides them. Over time, the dependency graph becomes complex and hard to trace. There is no single manifest that says "these are the packages I chose to install."

**2. DNF autoremove is incomplete.**
```bash
dnf5 autoremove  # Removes packages installed as dependencies that are no longer needed
```
This works for simple cases but misses:
- Packages you installed manually and forgot about
- Dependencies pulled from EPEL/RPM Fusion that are no longer needed by any user-installed package
- Config files left behind by removed packages (`dnf5 remove` does not remove config files in `/etc`)

**3. No atomic rollback of package state.**
If you install 20 packages over a month and want to "go back," you cannot. DNF5 has `dnf5 history undo <id>` for individual transactions, but undoing a complex series of installs/updates is unreliable.

**4. Mixing package installation methods.**
Your system will have packages from: DNF (base + EPEL + RPM Fusion + third-party), Flatpak, manual RPM installs (Citrix, Zoom), install scripts (Ollama, rustup, Bun, uv), and manual binary installs (Zed, Flutter). There is no single command that shows "everything installed on this system."

**5. No reproducibility.**
If your SSD dies, you cannot recreate the exact system state from a config file. You need to remember which repos to add, which packages to install, which scripts to run. The setup-runbook.md in this evaluation is the closest thing to a reproducible config, but it is a document you follow manually, not a machine-readable specification.

### Mitigation Strategies

**Strategy 1: Btrfs Snapshots (Primary Safety Net)**
```bash
# Install Snapper for automatic Btrfs snapshots
dnf5 install snapper

# Create a Snapper config for the root filesystem
snapper -c root create-config /

# Configure automatic snapshots
# Edit /etc/snapper/configs/root:
# TIMELINE_CREATE="yes"
# TIMELINE_CLEANUP="yes"
# TIMELINE_MIN_AGE="1800"
# TIMELINE_LIMIT_HOURLY="5"
# TIMELINE_LIMIT_DAILY="7"
# TIMELINE_LIMIT_WEEKLY="4"
# TIMELINE_LIMIT_MONTHLY="6"

# Create a pre/post snapshot around DNF operations
snapper create -d "before dnf update" -t pre
dnf5 update -y
snapper create -d "after dnf update" -t post

# Rollback if something breaks
snapper undochange <pre-id>..<post-id>
```

This gives you rollback capability, but it is filesystem-level (revert all files to a point in time), not package-level (revert specific package changes). It also requires Btrfs as the root filesystem, which means choosing Btrfs during install.

**Strategy 2: Keep a Package Manifest**
Maintain a text file listing every intentionally installed package:
```bash
# Save list of user-installed packages (not auto-installed dependencies)
dnf5 history userinstalled > ~/projects/nixos/docs/almalinux/installed-packages.txt

# Or use repoquery to list explicitly installed packages
dnf5 repoquery --userinstalled --qf '%{name}' > ~/installed-packages.txt
```

Review this list periodically and remove packages you no longer need.

**Strategy 3: Use Flatpak for GUI Apps**
Flatpak apps are sandboxed and self-contained. Installing or removing them does not affect the base system. Using Flatpak for all non-critical GUI apps (Discord, Obsidian, Logseq, etc.) keeps the RPM package count lower and reduces dependency interactions.

**Strategy 4: Use Containers for Dev Dependencies**
Instead of installing every language runtime and library on the host:
```bash
# Use Podman/Docker for project-specific dependencies
podman run -it --rm -v $(pwd):/app:Z node:22 npm install
podman run -it --rm -v $(pwd):/app:Z python:3.13 pip install -r requirements.txt
```

This is the container equivalent of NixOS's `devShells` -- isolating project dependencies from the host system.

**Strategy 5: Periodic Cleanup**
```bash
# Remove orphaned packages
dnf5 autoremove

# Clean DNF cache
dnf5 clean all

# Remove old kernels (keep current + 1 previous)
dnf5 remove --oldinstallonly --setopt installonly_limit=2

# Check for packages not from any enabled repo (manually installed RPMs)
dnf5 list extras

# Remove Flatpak unused runtimes
flatpak uninstall --unused
```

### Comparison to NixOS Package Creep Prevention

| Aspect | NixOS | AlmaLinux 10 |
|---|---|---|
| Package manifest | `flake.nix` -- single declarative file | No equivalent; closest is `dnf5 history userinstalled` |
| Adding a package | Edit config, rebuild, commit to git | `dnf5 install`, optionally note it somewhere |
| Removing a package | Remove from config, rebuild, GC | `dnf5 remove`, hope deps are cleaned up |
| Orphan cleanup | `nix-collect-garbage -d` removes everything not referenced by current config | `dnf5 autoremove` catches some orphans, misses others |
| Rollback | Boot to previous generation; instant, package-level | Btrfs snapshot rollback; filesystem-level, manual |
| Reproducing system | `nixos-rebuild switch --flake .#framework` on new hardware | Follow runbook manually; hope you documented everything |
| Dev environment isolation | `nix develop` / devShells per project | Podman/Docker containers; Toolbox |
| Config drift detection | `diff` against git history of flake.nix | No built-in mechanism |

**Verdict:** NixOS is categorically superior for package creep prevention. AlmaLinux can mitigate it with discipline and tooling, but it requires constant manual effort. NixOS makes it structural -- you cannot drift from your declared state because the system IS the declared state.

## DE Switching: KDE and GNOME Workflow

### If KDE is Available (via EPEL/KDE SIG)

```bash
# Install both DEs
dnf5 group install "KDE Plasma Workspaces"
# GNOME is already installed from base

# Option A: Use SDDM (KDE's display manager)
systemctl disable gdm
systemctl enable sddm

# Option B: Use GDM (GNOME's display manager)
# GDM can also present KDE sessions in the session picker

# Switching between DEs: log out, select other session at login screen
```

Both DEs coexist. Settings are stored in different directories (`~/.config/plasma*` vs `~/.config/gnome*`). XDG portals need configuration:

```bash
# Ensure both portal backends are installed
dnf5 install xdg-desktop-portal-kde xdg-desktop-portal-gnome
```

### If Only GNOME is Available

GNOME 47 on RHEL 10 is a strong desktop:
- Wayland native
- Fractional scaling (100%, 125%, 150%, 175%, 200%)
- GNOME Online Accounts for Google Drive, OneDrive, Gmail, Calendar
- Extensions for additional functionality (GSConnect for phone integration)
- Evolution for email/calendar/contacts with account sync

If KDE is unavailable, GNOME with the following extensions provides a similar workflow:
```bash
# Install GNOME extensions support
dnf5 install gnome-extensions-app gnome-shell-extension-appindicator

# Install GSConnect (KDE Connect equivalent for GNOME)
dnf5 install gnome-shell-extension-gsconnect
# Or install from extensions.gnome.org
```

## Framework 13 AMD 7840U Hardware Support

### Kernel

RHEL 10 ships kernel 6.12, which includes:
- Full AMD Ryzen 7 7840U support (Zen 4 architecture)
- AMD Radeon 780M (RDNA 3) iGPU support via `amdgpu` driver
- Framework 13 specific: ambient light sensor, fingerprint reader support
- Power management: AMD P-State driver (active mode)

```bash
# Verify AMD P-State is active
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
# Expected output: amd-pstate-epp

# Check GPU driver
lspci -k | grep -A3 VGA
# Expected: Kernel driver in use: amdgpu
```

### GPU / Display

```bash
# Mesa and Vulkan are in base repos
dnf5 install mesa-dri-drivers mesa-vulkan-drivers

# For hardware video acceleration (VA-API)
dnf5 install mesa-va-drivers
# Or from RPM Fusion for full codec support:
dnf5 install mesa-va-drivers-freeworld  # if RPM Fusion EL10 available

# Verify Vulkan
vulkaninfo --summary

# Verify VA-API
vainfo
```

Fractional scaling on the Framework 13 display (2256x1504, 13.5"):
```bash
# GNOME: Settings > Displays > Scale: 150% (recommended)
# KDE (if available): System Settings > Display > Scale: 150%

# Wayland fractional scaling should work out of the box with GNOME 47
# GNOME uses the wp_fractional_scale_v1 Wayland protocol
```

### Firmware Updates

```bash
# fwupd is in base repos
dnf5 install fwupd

# Framework laptops are supported by fwupd/LVFS
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update
```

### Power Management

```bash
# power-profiles-daemon is in base repos
dnf5 install power-profiles-daemon
systemctl enable --now power-profiles-daemon

# Check available profiles
powerprofilesctl list
# Expected: performance, balanced, power-saver

# Set balanced (default, good for battery life)
powerprofilesctl set balanced

# For additional power tweaks (TLP alternative)
# TLP may be in EPEL:
dnf5 install tlp
# Note: Do NOT use both power-profiles-daemon and TLP simultaneously
```

### Thermal and Sensor Monitoring

```bash
dnf5 install lm_sensors
sensors-detect  # Accept defaults
sensors          # View temperatures

# For Framework-specific battery/charge thresholds:
# Framework EC (Embedded Controller) support is in kernel 6.12
# Check if framework_laptop module is loaded:
lsmod | grep framework
# If available, charge limit can be set:
echo 80 > /sys/class/power_supply/BAT1/charge_control_end_threshold
```

## Security Model

### SELinux (Default on RHEL/AlmaLinux)

RHEL 10 uses SELinux in enforcing mode by default. This is a fundamentally different security model from Debian's AppArmor or NixOS's optional AppArmor.

**What SELinux does:**
- Mandatory Access Control (MAC) on top of standard Unix permissions
- Every file, process, and port has a security context (label)
- Policy rules define which contexts can access which resources
- "Targeted" policy (default): only specific system services are confined; user processes run in `unconfined_t` domain

**For this user's workflow:**
- SELinux works transparently for 99% of desktop use
- Docker CE with SELinux: works, but requires `--security-opt label=disable` or proper SELinux contexts for bind mounts. Podman handles this better with the `:Z` volume flag.
- Third-party RPMs (Citrix, Zoom): may need custom SELinux policies if they do unusual things. Most vendor RPMs include proper contexts.
- Manual installs (Ollama, Zed AppImage): may trigger SELinux denials. Fix with:

```bash
# Check for SELinux denials
ausearch -m AVC -ts recent

# Generate and install a custom policy module for a specific denial
audit2allow -a -M my-custom-policy
semodule -i my-custom-policy.pp

# Or temporarily set permissive mode for debugging (NOT recommended long-term)
setenforce 0  # Permissive (logs but does not block)
setenforce 1  # Enforcing (default)

# Check current mode
getenforce
```

**SELinux vs AppArmor:**
- SELinux is more comprehensive but harder to debug
- AppArmor (Debian/NixOS) is simpler, path-based profiles, easier to write custom policies
- For a desktop user, SELinux's targeted policy rarely causes issues
- For a developer running containers, SELinux adds complexity that Podman handles well but Docker handles less gracefully

**Can you use AppArmor instead?**
Technically possible but strongly discouraged on RHEL. The entire RHEL ecosystem assumes SELinux. Switching to AppArmor means losing all shipped SELinux policies and most RHEL documentation. Keep SELinux in enforcing mode and learn to work with it.

## Podman vs Docker on AlmaLinux 10

### Podman (Native)

Podman is Red Hat's container runtime and is deeply integrated into RHEL 10:

```bash
# Already installed or easily installed from base repos
dnf5 install podman podman-compose

# Rootless containers (no daemon, no root)
podman run -it --rm docker.io/library/ubuntu:latest bash

# Podman Compose (Docker Compose compatible)
podman-compose up -d

# Quadlet: systemd-native container management
# Place .container files in ~/.config/containers/systemd/
# systemd auto-generates service units for them
```

Podman advantages on RHEL:
- Rootless by default (no daemon running as root)
- SELinux-aware volume mounts with `:Z` flag
- Quadlet for systemd integration
- OCI-compliant, compatible with Docker images
- No separate daemon process

### Docker CE (Third-Party)

Docker CE is available via Docker's official RPM repo:

```bash
# Add Docker repo
dnf5 config-manager addrepo --from-repofile=https://download.docker.com/linux/rhel/docker-ce.repo

# Install
dnf5 install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start
systemctl enable --now docker

# Add user to docker group
usermod -aG docker zervin
```

Docker considerations on RHEL 10:
- Works fine but is not the "blessed" container runtime
- Docker daemon runs as root (security concern vs Podman's rootless)
- SELinux interactions can be more complex than with Podman
- `docker compose` works the same as everywhere
- Some RHEL-specific documentation assumes Podman; Docker is treated as third-party

### Recommendation

Use both:
- **Podman** for production-like workflows, systemd-managed services, and when SELinux compliance matters
- **Docker CE** for development workflows where `docker compose` and Docker-specific tooling is needed
- They can coexist on the same system without conflict

```bash
# Both installed
dnf5 install podman podman-compose
# Docker from docker repo (see setup-runbook.md)

# Use Podman as a Docker CLI drop-in (optional)
dnf5 install podman-docker  # Provides /usr/bin/docker -> podman alias
# Note: This conflicts with Docker CE. Choose one or the other for the `docker` command.
```

## Migration Path from Debian 13

### Pre-Migration Checklist

```bash
# On current Debian 13 system:

# 1. Document installed packages
dpkg --get-selections | grep -v deinstall > ~/debian-packages.txt
flatpak list --app --columns=application > ~/debian-flatpaks.txt
systemctl list-unit-files --state=enabled > ~/debian-services.txt

# 2. Backup home directory
rsync -avhP --delete \
  --exclude='.cache' \
  --exclude='.local/share/Trash' \
  --exclude='.local/share/Steam/steamapps' \
  /home/zervin/ /mnt/backup/home-zervin/

# 3. Export critical configs
gpg --export-secret-keys --armor > /mnt/backup/gpg-secret-keys.asc
gpg --export --armor > /mnt/backup/gpg-public-keys.asc
cp -r ~/.ssh /mnt/backup/ssh-keys/
cp -r ~/.config/syncthing /mnt/backup/syncthing-config/

# 4. Document Tailscale device name and network
tailscale status > /mnt/backup/tailscale-status.txt

# 5. Note LUKS passphrase (you will need it for the new LUKS volume)
```

### Installation Path

1. Download AlmaLinux 10 Workstation ISO (includes GNOME desktop)
2. Boot from USB, select installation
3. Choose LUKS encryption for root partition
4. Install with GNOME desktop (add KDE later if available)
5. Follow the setup-runbook.md for post-install configuration
6. Restore home directory data from backup
7. Re-authenticate services (Tailscale, Syncthing, GNOME Online Accounts, etc.)

### What Transfers Cleanly

- SSH keys (copy `~/.ssh/`)
- GPG keys (import from backup)
- Git config (in `~/.gitconfig`)
- Project files (in `~/projects/`)
- Syncthing data (restore config, reconnect devices)
- Browser data (sync via browser account, or copy profile)
- VS Code settings (Settings Sync via Microsoft account)
- Most dotfiles (`.bashrc`, `.bash_aliases`, etc.)

### What Needs Reconfiguration

- All system services (Docker, Tailscale, Syncthing, Ollama, CUPS)
- Desktop environment settings (wallpaper, panels, shortcuts)
- GNOME Online Accounts (re-add Google/Microsoft accounts)
- Flatpak apps (reinstall from Flathub)
- Citrix Workspace configuration (profiles, server URLs)
- Zoom settings (re-login)
- Printer setup

## Summary Assessment

### AlmaLinux 10 is a Strong Choice When:
- Work-critical applications need native RPM support (Citrix, Zoom, ZoomVDI) -- this is its strongest advantage
- You value long-term stability over package freshness
- You are comfortable with SELinux and the RHEL ecosystem
- GNOME is acceptable as your primary desktop (with KDE as a "nice to have")
- You want Podman as a first-class container runtime
- You want a distro with extensive vendor certification and documentation

### AlmaLinux 10 is a Weaker Choice When:
- KDE is a hard requirement (availability on EL10 is uncertain)
- Package creep prevention is your primary concern (NixOS is categorically better)
- You want reproducible system configurations (NixOS is categorically better)
- You need the latest versions of Mesa, Python, or GNOME
- You want atomic rollback at the package level (NixOS generations vs Btrfs snapshots)

### Key Risks

1. **KDE availability** -- MEDIUM-HIGH. Must verify before committing.
2. **RPM Fusion EL10 maturity** -- MEDIUM. Some packages may be missing.
3. **EPEL 10 coverage** -- LOW-MEDIUM. Improving but still less than EPEL 9.
4. **Package creep over time** -- HIGH. Requires manual discipline that NixOS makes structural.
5. **Insync EL10 support** -- LOW. Use GNOME Online Accounts as primary alternative.
