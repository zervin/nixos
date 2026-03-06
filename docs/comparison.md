# Migration Options Comparison: Debian 13 vs AlmaLinux 10 vs NixOS

## Overview

This document compares the three options for the Framework 13 AMD 7840U laptop:

1. **Stay on Debian 13 (Trixie)** -- the current system
2. **Migrate to AlmaLinux 10** -- RHEL 10 clone, traditional RPM-based distro
3. **Migrate to NixOS** -- declarative, reproducible system configuration

The comparison is written for a specific user profile: a developer working with Node.js, Python, Rust, Java, Flutter, Go, and containers, who prefers KDE Plasma for fractional scaling, uses Citrix/Zoom/ZoomVDI for work, and is concerned about package creep from constant tinkering.

---

## Comparison Matrix

### Package Freshness

| Aspect | Debian 13 | AlmaLinux 10 | NixOS (stable + unstable) |
|---|---|---|---|
| Base package age at release | Frozen at freeze date (~2025) | Frozen at RHEL 10 GA (mid-2025) | Frozen per stable release (~6 months) |
| Time between releases | ~2 years | ~10 years (minor point releases) | ~6 months |
| Kernel | 6.12 (at release) | 6.12 (at release) | 6.x (tracks latest stable within weeks) |
| Python | 3.13 | 3.12 | Current (unstable: days behind upstream) |
| Mesa / GPU drivers | Release version, security-only | Release version, security-only | Current (stable: 6 months, unstable: weeks) |
| GNOME / KDE | Release version | GNOME 47 frozen; KDE via EPEL (version depends on EPEL) | Both available, version depends on channel |
| Escape hatches | Backports, Flatpak, third-party repos | EPEL, RPM Fusion, Flatpak, third-party repos | Unstable overlay, Flatpak |

**Verdict:** NixOS wins. Its 6-month stable cycle plus instant access to unstable packages gives the most flexibility. Debian and AlmaLinux are comparable at release, but Debian's 2-year cycle means packages stay reasonably fresh, while AlmaLinux's 10-year cycle means base packages become genuinely old. Both rely on third-party sources for freshness, but AlmaLinux relies on them more heavily and for longer.

### Package Creep Risk

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Package manifest | None (imperative `apt install`) | None (imperative `dnf5 install`) | `flake.nix` (declarative, git-tracked) |
| Orphan cleanup | `apt autoremove` (decent) | `dnf5 autoremove` (decent) | `nix-collect-garbage -d` (comprehensive) |
| Config drift detection | None built-in | None built-in | `diff` against git history |
| Reproducibility | Low (reinstall = follow notes) | Low (reinstall = follow notes) | High (rebuild from config) |
| Mixed install methods | apt + Flatpak + manual | dnf + Flatpak + manual + 10+ repos | Nix + Flatpak (declarative) |
| Dev environment isolation | venv, nvm, containers | venv, nvm, containers, Toolbox | devShells (nix develop) + containers |
| System state auditability | `dpkg -l` + manual tracking | `rpm -qa` + manual tracking | The config IS the system |

**Verdict:** NixOS wins by a wide margin. Package creep was the user's primary concern, and NixOS is the only option that makes creep prevention structural rather than aspirational. Debian and AlmaLinux are roughly equivalent -- both require manual discipline. AlmaLinux is slightly worse because it needs more third-party repos (more sources of untracked state).

### Reproducibility

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Can you rebuild from scratch? | Only if you documented everything | Only if you documented everything | Yes: `nixos-rebuild switch --flake .#framework` |
| Is the config version-controlled? | Your dotfiles, maybe | Your dotfiles, maybe | Entire system config in git |
| Identical builds across machines? | No | No | Yes (same flake.lock = same system) |
| Share config with others? | Not meaningfully | Not meaningfully | Fork the flake repo |

**Verdict:** NixOS wins. This is its core design advantage. Neither Debian nor AlmaLinux can match it. For a developer who worries about their system becoming an unreproducible snowflake, NixOS is the answer.

### Desktop Environment Flexibility

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| KDE Plasma 6 | Available in base repos | **NOT in base repos; needs EPEL/KDE SIG (verify)** | Available in nixpkgs |
| GNOME | Available in base repos | Default, GNOME 47 | Available in nixpkgs |
| Dual DE (KDE + GNOME) | Install both, session picker at login | Possible IF KDE is available | Declarative: enable both in config |
| Fractional scaling (KDE) | Good (Wayland, per-monitor) | Depends on KDE version from EPEL | Good (Wayland, per-monitor) |
| Fractional scaling (GNOME) | Good (GNOME 46+) | Good (GNOME 47) | Good |
| GNOME Online Accounts | Available | Available (native in RHEL) | Available |
| DE switching without reinstall | Yes | Yes (if KDE installable) | Yes (edit config, rebuild) |

**Verdict:** Debian and NixOS tie. Both have KDE and GNOME readily available. AlmaLinux is the weakest here because KDE availability is uncertain and depends on community repos. If KDE is a hard requirement, AlmaLinux is a risk.

### Rollback Capability

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Package-level rollback | `apt` can downgrade individual packages | `dnf5 history undo` for individual transactions | Boot to any previous generation |
| System-level rollback | Btrfs snapshots (manual setup) | Btrfs snapshots (manual setup, Snapper) | Automatic: every rebuild creates a generation |
| Rollback at boot | Possible with Btrfs + grub-btrfs | Possible with Btrfs + Snapper | Built-in: GRUB menu lists generations |
| Tested rollback path | Third-party tooling | Third-party tooling (Snapper) | First-class, built into the system |

**Verdict:** NixOS wins. Rollback is a core feature, not an afterthought. Debian and AlmaLinux can approximate it with Btrfs snapshots, but that is a filesystem-level sledgehammer, not a package-level scalpel. NixOS lets you atomically switch between exact system states.

### Developer Experience

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Container support | Docker + Podman (via third-party repos) | Docker + Podman (Podman is native and excellent) | Docker + Podman (via nixpkgs) |
| Language runtimes | apt or manual install | dnf + module streams or manual install | nixpkgs (all major languages) |
| Per-project environments | venv, nvm, containers | venv, nvm, containers, Toolbox | devShells (nix develop) + containers |
| CLI tools availability | Extensive (Debian has huge repos) | Moderate (base + EPEL) | Extensive (nixpkgs is the largest repo) |
| IDEs / Editors | All available | All available (VS Code via Microsoft repo) | All available |
| Build toolchains | Excellent (Debian is a build reference) | Good (RHEL is a build target) | Good (but some quirks with FHS assumptions) |
| FHS compliance | Yes | Yes | **No** (Nix store paths break FHS-assuming scripts) |

**Verdict:** Tie between Debian and AlmaLinux for traditional workflows. NixOS has the best per-project isolation (devShells) but the worst compatibility with scripts and build systems that assume FHS paths. For this user's mix of languages (Node, Python, Rust, Java, Flutter, Go), all three work. AlmaLinux has a slight edge for Podman-heavy workflows. NixOS has a slight edge for reproducible dev environments.

### Work Tool Compatibility (Citrix, Zoom, ZoomVDI)

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Citrix Workspace | .deb from Citrix | **RPM from Citrix (native, tested on RHEL)** | nixpkgs (may need unstable overlay) |
| Zoom | .deb from Zoom | **RPM from Zoom (native)** | nixpkgs (`zoom-us`) |
| ZoomVDI | .deb from Zoom | **RPM from Zoom (native)** | **Unknown; may need FHS wrapper** |
| Vendor support / testing | Citrix tests on Ubuntu/Debian | **Citrix tests on RHEL** | Citrix does not test on NixOS |
| Installation friction | Low (download .deb, install) | **Lowest (download .rpm, install)** | Medium-High (may need wrappers, overlay) |

**Verdict:** AlmaLinux wins clearly. RHEL is the platform Citrix and Zoom test against. RPMs install cleanly, SELinux contexts are correct, and if something breaks, vendor support has RHEL in their test matrix. NixOS is the weakest here -- ZoomVDI is a genuine risk that requires Phase 1 VM validation.

### Maintenance Burden

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Update frequency | `apt update && apt upgrade` weekly | `dnf5 upgrade` weekly | `nix flake update && nixos-rebuild switch` weekly |
| Major upgrades | Every ~2 years (Debian 14) | Rare (point releases, no major upgrade needed for 10 years) | Every ~6 months (change channel in flake.nix) |
| Breaking updates | Occasional (major upgrades) | Rare (RHEL is very conservative) | Occasional (unstable overlay can break) |
| Repo management | ~3-5 repos (manageable) | **~12-15 repos (complex)** | 1 flake.nix (simple) |
| Config file management | Traditional /etc files | Traditional /etc files | Declarative (no /etc management) |
| Learning new tools | None (already know it) | DNF5, SELinux basics | Nix language, flakes, Home Manager |

**Verdict:** Debian wins for lowest effort (it is the current system -- no migration). AlmaLinux has the lowest long-term major upgrade burden (10-year lifecycle) but the highest repo management complexity. NixOS has the highest initial learning investment but the lowest ongoing maintenance once set up (declarative config, no drift).

### Learning Curve

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Familiarity | Current system | **Prior experience with AlmaLinux** | New paradigm |
| Package management | Known (apt) | Known (dnf/dnf5, minor syntax changes) | New (Nix language, derivations, flakes) |
| Security model | Known (AppArmor) | Different (SELinux -- more powerful, more complex) | Similar to Debian (optional AppArmor) |
| System configuration | Known (/etc files) | Similar (/etc files, similar tools) | Completely different (Nix modules, rebuild) |
| Time to productive system | 0 (already there) | ~1 day (install + configure) | ~1-2 weeks (learn Nix + configure) |
| Time to mastery | Already there | ~1 week (SELinux, DNF5 differences) | ~1-3 months (Nix language, flake patterns) |

**Verdict:** Staying on Debian wins. AlmaLinux is a close second due to prior experience and similar paradigm. NixOS requires the most significant learning investment -- the Nix language is a real barrier, and the mental model shift from imperative to declarative system management takes time.

### Framework 13 AMD Hardware Support

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Kernel version | 6.12 | 6.12 | 6.x (tracks upstream) |
| AMD 7840U / Radeon 780M | Full support | Full support | Full support (nixos-hardware module) |
| fwupd / LVFS firmware | Supported | Supported | Supported |
| Power management | power-profiles-daemon | power-profiles-daemon | power-profiles-daemon |
| Fingerprint reader | Supported (fprintd) | Supported (fprintd) | Supported |
| Wi-Fi (MediaTek) | Supported | Supported | Supported |
| NixOS hardware module | N/A | N/A | `framework-13-7040-amd` (auto-tunes everything) |

**Verdict:** Functional tie. All three have kernel 6.12 with full Framework 13 support. NixOS has a slight edge due to the dedicated nixos-hardware module that auto-configures optimal settings, but the others achieve the same result with manual tweaks.

### Security Model

| Aspect | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| MAC framework | AppArmor (default) | **SELinux (enforcing, default)** | AppArmor (optional) |
| Security posture out of the box | Good | **Best (SELinux enforcing + RHEL hardening)** | Unique (immutable store, atomic updates) |
| Container security | AppArmor profiles | SELinux contexts (Podman-native) | AppArmor profiles |
| Flatpak sandboxing | Bubblewrap + portals | Bubblewrap + portals | Bubblewrap + portals |
| Supply chain security | apt signatures | RPM signatures + RHEL provenance | Reproducible builds, Nix store integrity |
| Firewall | nftables/iptables | firewalld (nftables backend) | Declarative nftables |

**Verdict:** AlmaLinux wins for traditional security (SELinux is more comprehensive than AppArmor). NixOS wins for supply-chain and immutability guarantees. Debian is adequate but the weakest overall. For a desktop user, all three are more than sufficient.

---

## Summary Scorecard

| Category | Debian 13 | AlmaLinux 10 | NixOS |
|---|---|---|---|
| Package freshness | C+ | C | A |
| Package creep prevention | C | C- | A+ |
| Reproducibility | D | D | A+ |
| DE flexibility | A | C+ (KDE uncertain) | A |
| Rollback capability | C | C+ (Snapper) | A+ |
| Developer experience | B+ | B+ | B+ |
| Work tools (Citrix/Zoom/VDI) | B+ | A+ | C+ |
| Maintenance burden | A (no migration) | B- (many repos) | B (after learning curve) |
| Learning curve | A (known) | B+ (familiar) | C (new paradigm) |
| Hardware support | A | A | A |
| Security model | B | A | B+ |

---

## Recommendation

### For This Specific User

The decision comes down to what you value most, ranked by priority:

**If package creep prevention is your top priority: NixOS.**

This was the original motivation for the evaluation, and NixOS is the only option that solves it structurally. Your system state is a git-tracked config file. Adding, removing, or changing packages is a declarative edit. Rollback is automatic. Garbage collection actually works. No other distribution can match this.

The cost is real: 1-3 months to become productive with the Nix language and ecosystem, and ongoing friction with FHS-assuming tools (Flutter, some build scripts). ZoomVDI is a genuine risk that must be validated in a VM before committing.

**If work tool reliability is your top priority: AlmaLinux 10.**

Citrix, Zoom, and ZoomVDI are your non-negotiable work tools. They all ship official RPMs tested on RHEL. Installing them on AlmaLinux is the lowest-friction, highest-confidence path. If your job depends on these tools working without fuss, this matters.

The cost: you inherit the traditional RPM package management model with all its creep potential. You need 12-15 repos. KDE availability is uncertain. Base packages freeze for 10 years.

**If minimizing disruption is your top priority: Stay on Debian 13.**

The system works. You know it. There is no migration risk, no learning curve, no repo wrangling. Debian 14 will come in ~2 years with fresh packages, and the upgrade path is well-tested.

The cost: nothing changes. Package creep continues. The system becomes harder to reproduce over time.

### The Opinionated Take

For a developer who is worried enough about package creep to evaluate two alternative distributions, **NixOS is the right answer**, provided you can validate ZoomVDI in Phase 1 VM testing.

Here is why:

1. Package creep is a chronic problem that gets worse over time. AlmaLinux and Debian both require you to fight it manually, forever. NixOS eliminates it by design. If this is truly your primary concern, half-measures will not satisfy you.

2. The learning curve is a one-time cost. Once you understand Nix and have your flake working, ongoing maintenance is actually lower than managing 15 RPM repos.

3. ZoomVDI is the only real blocker. If it works in nixpkgs (or via an FHS wrapper), NixOS has no remaining disqualifying issue. If it does not work, AlmaLinux becomes the obvious choice because work tools must work.

4. AlmaLinux is the best fallback. If NixOS validation fails (ZoomVDI does not work, Nix language is too frustrating, Citrix has issues), AlmaLinux 10 with Btrfs snapshots and a disciplined Flatpak-heavy approach is a solid, familiar option. Your prior good experience with it is real and should not be dismissed.

**Recommended approach:**

1. Phase 1: Test NixOS in a VM. Focus on Citrix Workspace, Zoom, and ZoomVDI.
2. If ZoomVDI works on NixOS: proceed with NixOS migration per the existing plan.
3. If ZoomVDI does not work on NixOS: evaluate AlmaLinux 10 in a VM. Verify KDE availability and RPM Fusion status.
4. If both work: choose NixOS (better long-term for your concerns).
5. If neither works for ZoomVDI: stay on Debian 13 and revisit in 6 months.

### Decision Matrix

| Scenario | Recommendation |
|---|---|
| ZoomVDI works on NixOS, willing to learn Nix | **NixOS** |
| ZoomVDI fails on NixOS, KDE available on AlmaLinux 10 | **AlmaLinux 10** |
| ZoomVDI fails on NixOS, KDE NOT available on AlmaLinux 10 | **AlmaLinux 10 with GNOME** (or stay on Debian) |
| Not willing to invest time in Nix language | **AlmaLinux 10** |
| Want absolute minimum disruption | **Stay on Debian 13** |
