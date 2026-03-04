# Project Status -- 2026-03-04

## Phase: Planning & Assessment (Iteration 2)

### Completed This Cycle
- Full inventory of current Debian 13 system (hardware, packages, services, Flatpaks, dev stack)
- Feasibility assessment of NixOS migration for Framework 13 AMD 7840U
- Architecture design: stable + unstable overlay pattern, dual DE (KDE + GNOME), GNOME Online Accounts
- Migration plan with phased approach and package creep prevention strategy
- Step-by-step migration runbook
- Configuration skeleton with dual nixpkgs inputs, overlay pattern, dual DE, devShell examples
- Package management runbook covering anti-creep practices, devShells, GC, generation management
- Incorporated user feedback: Citrix/Zoom/ZoomVDI as work-critical, stable base + unstable overlay, GNOME Online Accounts replacing Insync, package creep prevention as key concern

### In Progress
- Nothing in progress (planning deliverables complete, awaiting review)

### Blocked
- **ZoomVDI plugin**: Unknown whether `zoomvdi` exists in nixpkgs. Needs investigation during Phase 1 VM testing. Not a blocker for planning, but is a validation item.

### Next Up
1. Human review of updated docs (architecture, dev-plan, configuration skeleton, runbooks)
2. If approved: begin Phase 0 (learn Nix language, set up flake structure)
3. Phase 1 (VM testing) -- critical to validate ZoomVDI and Citrix from unstable overlay
4. Phase 2 onward per dev-plan.md

### Decisions Made (This Iteration)
- **Channel strategy**: Stable base (nixos-24.11, then 25.05) with selective unstable overlay -- confirmed by user
- **Citrix Workspace**: Work-critical, pull from unstable overlay to avoid libwebkitgtk2 issues -- confirmed
- **Zoom**: Work-critical, stable is adequate -- confirmed
- **ZoomVDI**: Work-critical, needs Phase 1 investigation -- acknowledged
- **Insync**: Replaced by GNOME Online Accounts in GNOME sessions; Flatpak or rclone for KDE sessions -- confirmed
- **Dual DE**: KDE Plasma 6 + GNOME simultaneously, session picker at SDDM -- confirmed
- **Package creep**: Addressed with declarative-only discipline, devShells, GC timer, generation limits

### Decisions Still Needed
- **ZoomVDI packaging**: Is it in nixpkgs? Does `zoom-us` include VDI? If not, what wrapper approach? -- Resolve in Phase 1
- **Partition strategy**: Fresh LUKS + btrfs install vs testing on external USB first -- Recommendation: VM test first, then fresh install with backup
- **direnv**: Enable for automatic devShell activation? -- Recommendation: yes, it is in the skeleton config but optional

### Risks
- **ZoomVDI** -- Medium -- May not be in nixpkgs; fallback is FHS wrapper or Flatpak. Must validate in Phase 1 before wiping Debian.
- **Citrix from unstable** -- Low -- Well-tested approach; the overlay pattern is standard NixOS practice.
- **Dual DE portal conflicts** -- Low -- XDG portal config handles this; each session uses its own portal via $XDG_CURRENT_DESKTOP.
- **Flutter dev environment** -- Medium -- Requires FHS wrapper and devShell. Documented with working example.
- **Learning curve** -- Medium -- First NixOS system requires time investment; mitigated by VM testing phase.

### Documents

| File | Description | Status |
|---|---|---|
| `docs/architecture.md` | System architecture: flake structure, stable+unstable overlay, dual DE, security | Updated v2 |
| `docs/dev-plan.md` | Migration plan: phases, feasibility, package creep prevention, gotchas | Updated v2 |
| `docs/configuration-skeleton.nix` | Full NixOS + Home Manager config with overlay, dual DE, devShell examples | Updated v2 |
| `docs/runbooks/migration-steps.md` | Step-by-step install runbook: backup, partition, install, restore, verify | v1 (still current) |
| `docs/runbooks/package-management.md` | Package management: three contexts, anti-creep rules, GC, devShells | New |
| `docs/status.md` | This file | Updated v2 |
