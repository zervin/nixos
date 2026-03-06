# AlmaLinux 10 Repository Manifest

This document lists every repository needed for the full AlmaLinux 10 setup, how to enable each one, what key packages it provides, and any caveats.

**Important:** AlmaLinux 10 / RHEL 10 uses **DNF5** as the default package manager. Most DNF4 commands work the same way, but some subcommands have changed. This document uses DNF5 syntax throughout.

## Repository Summary Table

| # | Repository | Type | Packages Provided | Status for EL10 |
|---|---|---|---|---|
| 1 | BaseOS | Base | Kernel, systemd, core utils, NetworkManager | GA, always enabled |
| 2 | AppStream | Base | GNOME, Firefox, Thunderbird, Python, LibreOffice | GA, always enabled |
| 3 | CRB | Base (disabled) | Dev headers, libraries needed by EPEL | GA, must enable |
| 4 | EPEL 10 | Community | Syncthing, htop, KDE (maybe), extra CLI tools | Available, maturing |
| 5 | RPM Fusion Free | Community | VLC, ffmpeg, multimedia libs | **Verify availability** |
| 6 | RPM Fusion Nonfree | Community | Steam, NVIDIA drivers | **Verify availability** |
| 7 | Brave Browser | Vendor | `brave-browser` | Available |
| 8 | Microsoft VS Code | Vendor | `code`, `code-insiders` | Available |
| 9 | Docker CE | Vendor | `docker-ce`, `docker-compose-plugin` | Available |
| 10 | Tailscale | Vendor | `tailscale` | Available |
| 11 | Google Cloud SDK | Vendor | `google-cloud-cli` | Available |
| 12 | GitHub CLI | Vendor | `gh` | Available |
| 13 | Azul Zulu JDK | Vendor | `zulu21-jdk` | Available |
| 14 | NodeSource | Vendor | `nodejs` (22.x, 20.x) | Available |
| 15 | Insync | Vendor | `insync` | **Verify EL10 support** |
| 16 | Flathub (Flatpak) | Flatpak remote | Discord, Obsidian, Logseq, Bambu Studio, etc. | Available |

---

## Base Repositories (Enabled by Default)

### 1. BaseOS

**Enabled by default.** Contains the core operating system packages.

```bash
# Verify it is enabled
dnf5 repolist | grep baseos
```

**Key packages:**
- `kernel`, `systemd`, `NetworkManager`, `firewalld`
- `btrfs-progs`, `lvm2`, `cryptsetup` (LUKS)
- `podman`, `buildah`, `skopeo`
- `fwupd`, `lm_sensors`, `bluez`
- `pipewire`, `wireplumber`
- `cups`, `cups-filters`
- `power-profiles-daemon`
- `git`, `make`, `gcc`, `cmake`

### 2. AppStream

**Enabled by default.** Contains applications, language runtimes, and module streams.

```bash
dnf5 repolist | grep appstream
```

**Key packages:**
- `gnome-shell`, `gnome-session`, `gdm`, `gnome-control-center`
- `firefox`, `thunderbird`
- `libreoffice-*`
- `python3` (3.12), `python3-pip`
- `golang`
- `gimp` (may be here or in EPEL)
- `mesa-dri-drivers`, `mesa-vulkan-drivers`, `mesa-va-drivers`

**Module streams** (versioned application stacks):
```bash
# List available module streams
dnf5 module list

# Examples of module streams in RHEL 10:
# nodejs:22  (Node.js 22 LTS)
# python3.12 (system default)
# postgresql:16

# Enable a module stream
dnf5 module enable nodejs:22
dnf5 install nodejs
```

---

## CRB (CodeReady Builder)

### 3. CRB

**Disabled by default. Must enable.**

CRB contains development headers, static libraries, and tools needed to build packages from source and required as dependencies by many EPEL packages. Always enable this before enabling EPEL.

```bash
# Enable CRB
dnf5 config-manager setopt crb.enabled=1

# Verify
dnf5 repolist | grep crb
```

**Key packages:**
- Development headers (`*-devel` packages)
- `doxygen`, `texlive`, `ninja-build`
- Various libraries needed by EPEL packages as build dependencies

**Caveat:** Some documentation and older guides use `dnf config-manager --set-enabled crb` (DNF4 syntax). On RHEL 10 with DNF5, the correct syntax is `dnf5 config-manager setopt crb.enabled=1`.

---

## Community Repositories

### 4. EPEL 10 (Extra Packages for Enterprise Linux)

**Must install.** EPEL is maintained by the Fedora project and provides packages that are in Fedora but not in RHEL.

```bash
# Install EPEL release package
dnf5 install epel-release

# Or manually:
dnf5 install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

# Verify
dnf5 repolist | grep epel
```

**Key packages (expected, verify availability):**
- `syncthing` -- file synchronization
- `htop`, `neofetch`, `bat`, `fd-find`, `ripgrep` -- CLI utilities
- `snapd` -- Snap support (if wanted, probably not)
- `kde-*` / `plasma-*` -- KDE Plasma desktop (via KDE SIG, **verify**)
- `kdeconnect` -- phone integration for KDE
- `inkscape` -- vector graphics (if not in AppStream)
- Additional Python packages, dev tools, system utilities

**Caveats:**
- EPEL 10 will have fewer packages than EPEL 9 in its first year. Coverage grows over time as packagers rebuild for EL10.
- EPEL never conflicts with base RHEL packages (by policy). If a package is in RHEL base, EPEL will not ship a different version.
- EPEL packages do not receive the same security patch SLA as RHEL base packages.

### 5. RPM Fusion Free (EL10)

**Must install, if available.** RPM Fusion provides packages that Fedora and RHEL cannot ship due to patent, licensing, or policy reasons.

```bash
# Install RPM Fusion Free
dnf5 install https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-10.noarch.rpm

# Verify
dnf5 repolist | grep rpmfusion-free
```

**Key packages:**
- `vlc` -- media player
- `ffmpeg` -- multimedia framework (codecs)
- `x264`, `x265` -- video codecs
- `mesa-va-drivers-freeworld` -- full VA-API hardware video acceleration
- `mesa-vdpau-drivers-freeworld` -- VDPAU drivers

**Caveats:**
- **RPM Fusion EL10 availability is not guaranteed at this time.** RPM Fusion historically lags behind new EL major releases by several months. Check https://rpmfusion.org/ for current status.
- If not available, VLC can be installed via Flatpak, and codecs can be sourced from alternative repos or compiled manually.

### 6. RPM Fusion Nonfree (EL10)

```bash
# Install RPM Fusion Nonfree (requires Free repo first)
dnf5 install https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-10.noarch.rpm

# Verify
dnf5 repolist | grep rpmfusion-nonfree
```

**Key packages:**
- `steam` -- gaming platform
- NVIDIA proprietary drivers (not needed for AMD GPU, but available)

**Caveats:**
- Same availability concern as RPM Fusion Free for EL10.
- Steam can be installed via Flatpak as an alternative if RPM Fusion is unavailable.
- For AMD GPU (Radeon 780M), the open-source Mesa drivers in base repos are sufficient. No proprietary driver needed.

---

## Third-Party Vendor Repositories

### 7. Brave Browser

```bash
# Import GPG key
rpm --import https://brave-browser-rpm-release.s3.brave.com/brave-core.asc

# Add repo
dnf5 config-manager addrepo --from-repofile=https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo

# Install
dnf5 install brave-browser
```

**Provides:** `brave-browser`

**Caveat:** Brave's RPM repo targets Fedora/RHEL. Should work on EL10 without issues.

### 8. Microsoft VS Code

```bash
# Import GPG key
rpm --import https://packages.microsoft.com/keys/microsoft.asc

# Add repo
cat > /etc/yum.repos.d/vscode.repo << 'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

# Install
dnf5 install code          # VS Code Stable
dnf5 install code-insiders # VS Code Insiders
```

**Provides:** `code`, `code-insiders`

**Caveat:** None. Microsoft's RPM repo is well-maintained and targets RHEL/Fedora.

### 9. Docker CE

```bash
# Add Docker repo
dnf5 config-manager addrepo --from-repofile=https://download.docker.com/linux/rhel/docker-ce.repo

# Install
dnf5 install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start
systemctl enable --now docker

# Add user to docker group (log out and back in after)
usermod -aG docker zervin
```

**Provides:** `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`

**Caveats:**
- Docker's RHEL repo uses `/linux/rhel/` path. For AlmaLinux, this works because AlmaLinux is binary-compatible with RHEL.
- Docker CE and Podman can coexist, but do NOT install `podman-docker` (which aliases `docker` to `podman`) if you want to use actual Docker CE.
- Docker daemon runs as root. For rootless containers, Podman is the better choice on RHEL.

### 10. Tailscale

```bash
# Add Tailscale repo
dnf5 config-manager addrepo --from-repofile=https://pkgs.tailscale.com/stable/rhel/10/tailscale.repo

# Install
dnf5 install tailscale

# Enable and start
systemctl enable --now tailscaled

# Authenticate
tailscale up
```

**Provides:** `tailscale`, `tailscaled`

**Caveat:** Tailscale's repo URL includes the RHEL major version. Use `10` for EL10. If the exact URL has changed, check https://pkgs.tailscale.com/stable/.

### 11. Google Cloud SDK

```bash
# Add repo
cat > /etc/yum.repos.d/google-cloud-sdk.repo << 'EOF'
[google-cloud-cli]
name=Google Cloud CLI
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Install
dnf5 install google-cloud-cli
```

**Provides:** `google-cloud-cli` (`gcloud`, `gsutil`, `bq`)

**Caveat:** Google may not have an EL10-specific repo immediately. The EL9 repo (`el9`) typically works on EL10 because `google-cloud-cli` is a self-contained package with minimal system dependencies. If this fails, use the standalone installer: `curl https://sdk.cloud.google.com | bash`.

### 12. GitHub CLI

```bash
# Add repo
dnf5 config-manager addrepo --from-repofile=https://cli.github.com/packages/rpm/gh-cli.repo

# Install
dnf5 install gh
```

**Provides:** `gh`

**Caveat:** None. GitHub CLI's RPM repo targets Fedora/RHEL generically.

### 13. Azul Zulu JDK

```bash
# Add Azul repo
cat > /etc/yum.repos.d/azul.repo << 'EOF'
[azul-zulu]
name=Azul Zulu JDK
baseurl=https://repos.azul.com/zulu/rpm/
enabled=1
gpgcheck=1
gpgkey=https://repos.azul.com/azul-repo.key
EOF

# Install Zulu JDK 21
dnf5 install zulu21-jdk

# Verify
java -version
# Expected: openjdk version "21.x.x" ... Zulu21...
```

**Provides:** `zulu21-jdk` (and other versions: `zulu17-jdk`, `zulu11-jdk`, etc.)

**Caveat:** Azul's repo is architecture-specific. The `baseurl` above works for x86_64. Azul repos are well-maintained and target RHEL.

### 14. NodeSource

```bash
# Install NodeSource repo for Node.js 22
dnf5 install https://rpm.nodesource.com/pub_22.x/nodistro/repo/nodesource-release-nodistro-1.noarch.rpm

# Install
dnf5 install nodejs

# Verify
node --version
# Expected: v22.x.x
```

**Alternative: AppStream module streams**
```bash
# RHEL 10 AppStream has Node.js module streams
dnf5 module enable nodejs:22
dnf5 install nodejs

# This gives you the RHEL-maintained Node.js 22 LTS
# Advantage: RHEL security patches
# Disadvantage: May lag behind upstream Node.js releases
```

**Provides:** `nodejs`, `npm`

**Caveat:** If using NodeSource, it overrides the AppStream `nodejs` package. Do not mix both. Choose one source for Node.js. For a developer who wants the latest Node.js, NodeSource is preferred. For stability, the AppStream module stream is better.

### 15. Insync

```bash
# Add Insync repo
rpm --import https://d2t3ff60b2tber.cloudfront.net/repomd.xml.key

cat > /etc/yum.repos.d/insync.repo << 'EOF'
[insync]
name=insync repo
baseurl=http://yum.insync.io/fedora/$releasever/
gpgcheck=1
gpgkey=https://d2t3ff60b2tber.cloudfront.net/repomd.xml.key
enabled=1
metadata_expire=120m
EOF

# Install
dnf5 install insync
```

**Provides:** `insync`

**Caveats:**
- Insync's RPM repo uses Fedora version numbering (`$releasever`). On AlmaLinux 10, `$releasever` is `10`, which Insync may not have a repo for.
- **Alternative base URL for RHEL:** `http://yum.insync.io/rhel/$releasever/` -- try this if the Fedora URL fails.
- If neither works, Insync is also available as a direct `.rpm` download from insync.io, or use GNOME Online Accounts as a replacement for Google Drive/OneDrive.
- **Recommended approach:** Use GNOME Online Accounts for Google Drive and OneDrive. Use Insync only if you need it in KDE sessions and GNOME Online Accounts is insufficient.

---

## Flatpak (Flathub)

### 16. Flathub Remote

```bash
# Flatpak should be installed by default on GNOME desktop
dnf5 install flatpak

# Add Flathub remote
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Verify
flatpak remotes
```

**Key Flatpak applications for this setup:**

```bash
# Communication
flatpak install flathub com.discordapp.Discord

# Notes / PKM
flatpak install flathub md.obsidian.Obsidian
flatpak install flathub com.logseq.Logseq

# Creative
flatpak install flathub com.bambulab.BambuStudio
flatpak install flathub org.inkscape.Inkscape
flatpak install flathub org.upscayl.Upscayl

# Media
flatpak install flathub net.mkiol.SpeechNote
flatpak install flathub org.nicotine_plus.Nicotine

# Email
flatpak install flathub eu.betterbird.Betterbird

# Gaming
flatpak install flathub com.valvesoftware.Steam   # If RPM Fusion unavailable

# Utilities
flatpak install flathub org.fkoehler.KTailctl      # Needs KDE runtime
```

**Caveats:**
- Flatpak apps run in sandboxes, which may limit file system access. Use Flatseal to manage permissions.
- Flatpak apps use their own runtime libraries (GNOME or KDE runtime), so they are larger than RPM equivalents.
- Flatpak auto-updates can be enabled via GNOME Software or `flatpak update` in a cron/timer.

---

## DNF5 vs DNF4 Syntax Reference

Key differences to be aware of:

| Operation | DNF4 (RHEL 9) | DNF5 (RHEL 10) |
|---|---|---|
| Install package | `dnf install pkg` | `dnf5 install pkg` |
| Remove package | `dnf remove pkg` | `dnf5 remove pkg` |
| Update all | `dnf update` | `dnf5 upgrade` |
| Search | `dnf search term` | `dnf5 search term` |
| List installed | `dnf list installed` | `dnf5 list --installed` |
| Enable repo | `dnf config-manager --set-enabled repo` | `dnf5 config-manager setopt repo.enabled=1` |
| Add repo from URL | `dnf config-manager --add-repo URL` | `dnf5 config-manager addrepo --from-repofile=URL` |
| Module enable | `dnf module enable mod:stream` | `dnf5 module enable mod:stream` |
| Autoremove | `dnf autoremove` | `dnf5 autoremove` |
| History | `dnf history` | `dnf5 history` |
| Clean cache | `dnf clean all` | `dnf5 clean all` |

**Note:** On RHEL 10, `dnf` is a symlink to `dnf5`. You can use either command name. This document uses `dnf5` explicitly for clarity.

---

## Repository Conflict Matrix

Potential conflicts between repositories:

| Conflict | Repos Involved | Issue | Resolution |
|---|---|---|---|
| Node.js | NodeSource vs AppStream module | Both provide `nodejs` | Disable AppStream module if using NodeSource: `dnf5 module disable nodejs` |
| Docker vs Podman-docker | Docker CE repo vs base | `podman-docker` aliases `docker` to `podman` | Do not install `podman-docker` if using Docker CE |
| Mesa drivers | Base vs RPM Fusion | RPM Fusion provides `-freeworld` variants with full codec support | Install RPM Fusion variants to replace base Mesa VA-API drivers |
| Python | System Python vs user-installed | System Python 3.12 is pinned | Use `uv` or `pyenv` for other Python versions; never touch system Python |

---

## Complete Repository Enable Script

For convenience, here is the full sequence to enable all repositories:

```bash
#!/bin/bash
# enable-all-repos.sh
# Run as root after fresh AlmaLinux 10 install

set -euo pipefail

echo "=== Enabling CRB ==="
dnf5 config-manager setopt crb.enabled=1

echo "=== Installing EPEL ==="
dnf5 install -y epel-release

echo "=== Installing RPM Fusion (if available) ==="
dnf5 install -y \
  https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-10.noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-10.noarch.rpm \
  || echo "WARNING: RPM Fusion EL10 may not be available yet. Skipping."

echo "=== Adding Brave Browser repo ==="
rpm --import https://brave-browser-rpm-release.s3.brave.com/brave-core.asc
dnf5 config-manager addrepo --from-repofile=https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo

echo "=== Adding VS Code repo ==="
rpm --import https://packages.microsoft.com/keys/microsoft.asc
cat > /etc/yum.repos.d/vscode.repo << 'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

echo "=== Adding Docker CE repo ==="
dnf5 config-manager addrepo --from-repofile=https://download.docker.com/linux/rhel/docker-ce.repo

echo "=== Adding Tailscale repo ==="
dnf5 config-manager addrepo --from-repofile=https://pkgs.tailscale.com/stable/rhel/10/tailscale.repo

echo "=== Adding Google Cloud SDK repo ==="
cat > /etc/yum.repos.d/google-cloud-sdk.repo << 'EOF'
[google-cloud-cli]
name=Google Cloud CLI
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

echo "=== Adding GitHub CLI repo ==="
dnf5 config-manager addrepo --from-repofile=https://cli.github.com/packages/rpm/gh-cli.repo

echo "=== Adding Azul Zulu JDK repo ==="
cat > /etc/yum.repos.d/azul.repo << 'EOF'
[azul-zulu]
name=Azul Zulu JDK
baseurl=https://repos.azul.com/zulu/rpm/
enabled=1
gpgcheck=1
gpgkey=https://repos.azul.com/azul-repo.key
EOF

echo "=== Adding NodeSource repo (Node.js 22) ==="
dnf5 install -y https://rpm.nodesource.com/pub_22.x/nodistro/repo/nodesource-release-nodistro-1.noarch.rpm
dnf5 module disable -y nodejs || true  # Disable AppStream module to avoid conflict

echo "=== Setting up Flatpak + Flathub ==="
dnf5 install -y flatpak
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

echo "=== All repositories configured ==="
dnf5 repolist
```

**Note:** Review and adjust URLs before running. Some third-party repo URLs may have changed since this document was written.
