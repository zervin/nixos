# Project Status -- 2026-03-05

## Phase: Planning & Assessment (Iteration 3 -- AlmaLinux Parallel Evaluation)

### Completed This Cycle
- Full inventory of current Debian 13 system (hardware, packages, services, Flatpaks, dev stack)
- Feasibility assessment of NixOS migration for Framework 13 AMD 7840U
- Architecture design: stable + unstable overlay pattern, dual DE (KDE + GNOME), GNOME Online Accounts
- Migration plan with phased approach and package creep prevention strategy
- Step-by-step migration runbook (NixOS)
- Configuration skeleton with dual nixpkgs inputs, overlay pattern, dual DE, devShell examples
- Package management runbook covering anti-creep practices, devShells, GC, generation management
- **NEW: AlmaLinux 10 full assessment** -- RHEL 10 base, package ecosystem map, package mapping, KDE availability analysis, package creep risk comparison, SELinux, Podman, Framework 13 hardware
- **NEW: AlmaLinux 10 repository manifest** -- all 16 repo sources documented with enable commands, key packages, caveats, DNF5 syntax, conflict matrix
- **NEW: AlmaLinux 10 setup runbook** -- complete step-by-step from install through post-install verification, including Btrfs/Snapper snapshots, SELinux verification, Framework 13 tweaks
- **NEW: Three-way comparison** (Debian 13 vs AlmaLinux 10 vs NixOS) with opinionated recommendation and decision matrix

### In Progress
- Nothing in progress (planning deliverables complete, awaiting review and decision)

### Blocked
- **ZoomVDI plugin on NixOS**: Unknown whether `zoomvdi` exists in nixpkgs. Needs investigation during Phase 1 VM testing. Not a blocker for planning, but is a validation item.
- **KDE on AlmaLinux 10**: KDE Plasma is NOT in RHEL base repos. Availability via EPEL 10 / KDE SIG must be verified before committing to AlmaLinux. Not a blocker for planning, but is a validation item.

### Next Up
1. Human review of all docs (NixOS evaluation + AlmaLinux evaluation + comparison)
2. **Decision: Which migration path to pursue?** (See comparison.md for recommendation and decision matrix)
3. If NixOS: begin Phase 0 (learn Nix language, set up flake structure)
4. If AlmaLinux: begin VM testing (validate KDE availability, RPM Fusion EL10, ZoomVDI RPM)
5. If staying on Debian: document current system state and package creep mitigation strategies

### Decisions Made (This Iteration)
- **Channel strategy (NixOS)**: Stable base (nixos-24.11, then 25.05) with selective unstable overlay -- confirmed by user
- **Citrix Workspace (NixOS)**: Work-critical, pull from unstable overlay to avoid libwebkitgtk2 issues -- confirmed
- **Zoom (NixOS)**: Work-critical, stable is adequate -- confirmed
- **ZoomVDI (NixOS)**: Work-critical, needs Phase 1 investigation -- acknowledged
- **Insync**: Replaced by GNOME Online Accounts in GNOME sessions; Flatpak or rclone for KDE sessions -- confirmed
- **Dual DE**: KDE Plasma 6 + GNOME simultaneously, session picker at SDDM -- confirmed
- **Package creep**: Addressed with declarative-only discipline (NixOS) or Btrfs snapshots + manual discipline (AlmaLinux)
- **AlmaLinux 10 added as parallel evaluation**: User has prior positive experience; evaluation now complete

### Decisions Still Needed
- **Which distro to migrate to?** Options: NixOS, AlmaLinux 10, or stay on Debian 13. See `docs/comparison.md` for detailed analysis and recommendation.
- **ZoomVDI packaging (NixOS)**: Is it in nixpkgs? Does `zoom-us` include VDI? If not, what wrapper approach? -- Resolve in Phase 1
- **KDE on AlmaLinux 10**: Is KDE Plasma 6 available via EPEL 10 or KDE SIG? -- Resolve during AlmaLinux VM testing
- **RPM Fusion EL10 status**: Are free and nonfree repos available for EL10? -- Check rpmfusion.org before AlmaLinux VM testing
- **Partition strategy**: Fresh LUKS + btrfs install vs testing on external USB first -- Recommendation: VM test first, then fresh install with backup
- **direnv (NixOS)**: Enable for automatic devShell activation? -- Recommendation: yes, it is in the skeleton config but optional

### Risks
- **ZoomVDI on NixOS** -- Medium -- May not be in nixpkgs; fallback is FHS wrapper or Flatpak. Must validate in Phase 1 before wiping Debian.
- **KDE on AlmaLinux 10** -- Medium-High -- Not in RHEL base repos. EPEL/KDE SIG availability for EL10 is unverified. If unavailable, must use GNOME.
- **RPM Fusion EL10** -- Medium -- RPM Fusion lags new EL releases. Steam, VLC, and multimedia codecs depend on it. Flatpak alternatives exist.
- **EPEL 10 coverage** -- Low-Medium -- Still maturing; some packages from EPEL 9 may not be rebuilt for EL10 yet.
- **Package creep on AlmaLinux** -- High -- 12-15 repos, no declarative manifest, no automatic rollback. Requires manual discipline that NixOS makes structural.
- **Citrix from unstable (NixOS)** -- Low -- Well-tested approach; the overlay pattern is standard NixOS practice.
- **Dual DE portal conflicts** -- Low -- XDG portal config handles this; each session uses its own portal via $XDG_CURRENT_DESKTOP.
- **Flutter dev environment (NixOS)** -- Medium -- Requires FHS wrapper and devShell. Documented with working example.
- **Learning curve (NixOS)** -- Medium -- First NixOS system requires time investment; mitigated by VM testing phase.

### Documents

| File | Description | Status |
|---|---|---|
| `docs/architecture.md` | NixOS system architecture: flake structure, stable+unstable overlay, dual DE, security | Updated v2 |
| `docs/dev-plan.md` | NixOS migration plan: phases, feasibility, package creep prevention, gotchas | Updated v2 |
| `docs/configuration-skeleton.nix` | NixOS + Home Manager config with overlay, dual DE, devShell examples | Updated v2 |
| `docs/runbooks/migration-steps.md` | NixOS step-by-step install runbook: backup, partition, install, restore, verify | v1 |
| `docs/runbooks/package-management.md` | NixOS package management: three contexts, anti-creep rules, GC, devShells | v1 |
| `docs/almalinux/assessment.md` | AlmaLinux 10 full assessment: RHEL 10 base, packages, KDE, creep, security, hardware | **New** |
| `docs/almalinux/repo-manifest.md` | AlmaLinux 10 repository reference: all repos, enable commands, packages, caveats | **New** |
| `docs/almalinux/setup-runbook.md` | AlmaLinux 10 step-by-step setup: install, repos, packages, services, Framework tweaks | **New** |
| `docs/comparison.md` | Three-way comparison: Debian 13 vs AlmaLinux 10 vs NixOS with recommendation | **New** |
| `docs/status.md` | This file | **Updated v3** |
