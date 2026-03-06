# AlmaLinux 10 Setup Runbook

Step-by-step guide to set up AlmaLinux 10 on a Framework 13 AMD 7840U laptop, replicating the current Debian 13 KDE Plasma environment.

## Pre-Flight Checklist

Before starting, confirm:

- [ ] External backup drive with full home directory backup (see assessment.md Migration section)
- [ ] AlmaLinux 10 Workstation ISO downloaded and written to USB (`dd` or Fedora Media Writer)
- [ ] Framework 13 power adapter plugged in
- [ ] 2-4 hours of uninterrupted time
- [ ] Tailscale auth key or browser access for re-authentication
- [ ] Syncthing device IDs documented
- [ ] SSH keys backed up (`~/.ssh/`)
- [ ] GPG keys exported (armor format)
- [ ] LUKS passphrase chosen (or reuse current one)
- [ ] This runbook accessible on another device (phone/tablet) during install

## Step 1: Install AlmaLinux 10

### 1.1 Boot from USB

1. Insert AlmaLinux 10 USB installer
2. Reboot Framework 13, press F2 for BIOS, select USB boot
3. Select "Install AlmaLinux 10" from GRUB menu

### 1.2 Anaconda Installer Choices

**Language/Keyboard:** English (US) or your preference.

**Installation Destination:**
- Select the NVMe drive
- Choose "Custom" partitioning
- Encryption: Check "Encrypt my data" -- this creates a LUKS2 volume

**Partition Layout (recommended for Btrfs with LUKS):**

| Mount Point | Size | Filesystem | Notes |
|---|---|---|---|
| `/boot/efi` | 512 MB | FAT32 (EFI System Partition) | Unencrypted |
| `/boot` | 1 GB | ext4 | Unencrypted (GRUB needs to read it) |
| `/` | Remaining | Btrfs (inside LUKS2) | Encrypted |

**If Anaconda supports Btrfs subvolumes in the custom partitioner:**
Create these subvolumes inside the Btrfs root:
- `@` mounted at `/`
- `@home` mounted at `/home`
- `@var` mounted at `/var` (optional, keeps logs separate)

If Anaconda does not support Btrfs subvolume creation, install with a single Btrfs root and restructure post-install (see Step 3).

**Software Selection:**
- Base: "Workstation" (includes GNOME desktop)
- Additional: Check "Development Tools" if available

**Network:** Connect to Wi-Fi during install for immediate package access.

**User:** Create user `zervin` with admin (wheel group) privileges.

### 1.3 Complete Installation

Click "Begin Installation," wait for completion, reboot, remove USB.

### 1.4 First Boot

1. Accept the GNOME initial setup prompts
2. Connect to Wi-Fi if not already connected
3. Open a terminal (GNOME Terminal)

## Step 2: Verify Base System

```bash
# Check AlmaLinux version
cat /etc/almalinux-release
# Expected: AlmaLinux release 10.0 (...)

# Check kernel
uname -r
# Expected: 6.12.x...

# Check filesystem
df -Th /
# Expected: btrfs

# Check LUKS
lsblk -f
# Should show LUKS and btrfs

# Check GPU driver
lspci -k | grep -A3 VGA
# Expected: Kernel driver in use: amdgpu

# Check Wayland
echo $XDG_SESSION_TYPE
# Expected: wayland

# Check DNF5
dnf5 --version
```

## Step 3: Configure Btrfs Subvolumes (If Not Done During Install)

If Anaconda installed a flat Btrfs filesystem without subvolumes, restructure now for snapshot support:

```bash
# This is complex and requires booting from a live USB.
# Skip this step if subvolumes were created during install.

# If you need subvolumes for Snapper snapshots:
# 1. Boot from AlmaLinux live USB
# 2. Unlock LUKS: cryptsetup luksOpen /dev/nvme0n1p3 cryptroot
# 3. Mount btrfs: mount /dev/mapper/cryptroot /mnt
# 4. Create subvolumes and move data
# 5. Update fstab
# This is involved -- see Btrfs documentation for details.

# For a simpler approach: skip subvolumes and use Snapper with the flat layout.
# Snapper can still take snapshots of the root filesystem.
```

## Step 4: Enable Repositories

Run the complete repository setup. See `repo-manifest.md` for details on each repo.

```bash
# Become root
sudo -i

# Enable CRB (needed for EPEL dependencies)
dnf5 config-manager setopt crb.enabled=1

# Install EPEL
dnf5 install -y epel-release

# Install RPM Fusion (check availability first)
dnf5 install -y \
  https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-10.noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-10.noarch.rpm \
  || echo "RPM Fusion EL10 not available yet -- install VLC/Steam via Flatpak instead"

# Add Brave Browser repo
rpm --import https://brave-browser-rpm-release.s3.brave.com/brave-core.asc
dnf5 config-manager addrepo --from-repofile=https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo

# Add VS Code repo
rpm --import https://packages.microsoft.com/keys/microsoft.asc
cat > /etc/yum.repos.d/vscode.repo << 'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

# Add Docker CE repo
dnf5 config-manager addrepo --from-repofile=https://download.docker.com/linux/rhel/docker-ce.repo

# Add Tailscale repo
dnf5 config-manager addrepo --from-repofile=https://pkgs.tailscale.com/stable/rhel/10/tailscale.repo

# Add Google Cloud SDK repo
cat > /etc/yum.repos.d/google-cloud-sdk.repo << 'EOF'
[google-cloud-cli]
name=Google Cloud CLI
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Add GitHub CLI repo
dnf5 config-manager addrepo --from-repofile=https://cli.github.com/packages/rpm/gh-cli.repo

# Add Azul Zulu JDK repo
cat > /etc/yum.repos.d/azul.repo << 'EOF'
[azul-zulu]
name=Azul Zulu JDK
baseurl=https://repos.azul.com/zulu/rpm/
enabled=1
gpgcheck=1
gpgkey=https://repos.azul.com/azul-repo.key
EOF

# Add NodeSource repo (Node.js 22)
dnf5 install -y https://rpm.nodesource.com/pub_22.x/nodistro/repo/nodesource-release-nodistro-1.noarch.rpm
# Disable AppStream Node.js module to avoid conflict
dnf5 module disable -y nodejs || true

# Verify all repos
dnf5 repolist
```

## Step 5: System Update

```bash
# Full system update with all new repos
sudo dnf5 upgrade -y

# Reboot to pick up any kernel update
sudo reboot
```

## Step 6: Install KDE Plasma (If Available)

```bash
# Check if KDE is available
dnf5 search plasma-desktop

# If available:
sudo dnf5 group install -y "KDE Plasma Workspaces"
# Or the specific group name -- check with:
sudo dnf5 group list | grep -i kde

# Install SDDM (KDE's display manager)
sudo dnf5 install -y sddm

# Switch from GDM to SDDM
sudo systemctl disable gdm
sudo systemctl enable sddm

# Install KDE-specific extras
sudo dnf5 install -y \
  kdeconnect \
  dolphin \
  kate \
  konsole \
  spectacle \
  ark \
  okular \
  gwenview \
  kde-connect \
  xdg-desktop-portal-kde

# Reboot to SDDM
sudo reboot

# After reboot: at SDDM login, select "Plasma (Wayland)" session
```

**If KDE is NOT available**, continue with GNOME 47 and enhance it:

```bash
# Install GNOME extensions and utilities
sudo dnf5 install -y \
  gnome-extensions-app \
  gnome-shell-extension-appindicator \
  gnome-shell-extension-gsconnect \
  gnome-tweaks \
  dconf-editor

# Configure fractional scaling in GNOME
# Settings > Displays > Scale: 150% (recommended for Framework 13)

# Set up GNOME Online Accounts
# Settings > Online Accounts > Add Google account (for Drive, Gmail, Calendar)
# Settings > Online Accounts > Add Microsoft account (for OneDrive, Outlook)
```

## Step 7: Install Work-Critical Packages

These three are non-negotiable. Install them first and verify they work before proceeding.

### 7.1 Citrix Workspace

```bash
# Download latest Citrix Workspace RPM from:
# https://www.citrix.com/downloads/workspace-app/linux/workspace-app-for-linux-latest.html
# Select "Workspace app for Linux (x86_64) - RPM"

# Install (adjust filename for current version)
sudo dnf5 install -y ~/Downloads/ICAClient-*.x86_64.rpm

# Accept EULA
# The RPM post-install may prompt for EULA acceptance

# Launch to verify
/opt/Citrix/ICAClient/selfservice &

# If SELinux blocks it:
sudo ausearch -m AVC -ts recent | grep -i ica
# If denials found:
sudo audit2allow -a -M citrix-workspace
sudo semodule -i citrix-workspace.pp
```

### 7.2 Zoom

```bash
# Download latest Zoom RPM from:
# https://zoom.us/download?os=linux
# Select "RPM (64-bit)"

# Install
sudo dnf5 install -y ~/Downloads/zoom_x86_64.rpm

# Launch to verify
zoom &
```

### 7.3 ZoomVDI Universal Plugin

```bash
# Download ZoomVDI Universal Plugin RPM from:
# https://zoom.us/download?os=linux
# Look for "VDI" section, select RPM

# Install
sudo dnf5 install -y ~/Downloads/zoomvdi-*.rpm

# Verify the plugin is detected by Zoom
# The VDI plugin integrates with the Zoom client and Citrix Workspace
```

**Verification checkpoint:** Log into Citrix Workspace, join a Zoom meeting through VDI. If this works, proceed. If not, troubleshoot before continuing.

## Step 8: Install Browsers

```bash
sudo dnf5 install -y \
  brave-browser \
  firefox
  # google-chrome-stable  # Uncomment if you add the Google Chrome repo
```

## Step 9: Install Development Tools

### 9.1 Core Dev Packages (RPM)

```bash
sudo dnf5 install -y \
  code \
  git \
  gh \
  google-cloud-cli \
  nodejs \
  npm \
  python3 \
  python3-pip \
  golang \
  zulu21-jdk \
  gcc \
  gcc-c++ \
  make \
  cmake \
  ninja-build \
  openssl-devel \
  pkg-config \
  clang \
  llvm
```

### 9.2 VS Code Insiders (Optional)

```bash
sudo dnf5 install -y code-insiders
```

### 9.3 Docker CE

```bash
sudo dnf5 install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable --now docker
sudo usermod -aG docker zervin

# Log out and back in for group change to take effect
```

### 9.4 Podman (Already Installed or Quick Install)

```bash
sudo dnf5 install -y podman podman-compose buildah skopeo

# Test rootless Podman
podman run --rm docker.io/library/hello-world
```

### 9.5 Rust via rustup

```bash
# As user zervin (not root)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Choose option 1 (default installation)
source "$HOME/.cargo/env"

# Verify
rustc --version
cargo --version
```

### 9.6 Bun

```bash
# As user zervin
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# Verify
bun --version
```

### 9.7 uv (Python Package Manager)

```bash
# As user zervin
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc

# Verify
uv --version
```

### 9.8 Flutter / Dart SDK

```bash
# As user zervin
mkdir -p ~/projects
cd ~/projects

# Download Flutter SDK
git clone https://github.com/flutter/flutter.git -b stable

# Add to PATH in ~/.bashrc
echo 'export PATH="$HOME/projects/flutter/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Run Flutter doctor to check dependencies
flutter doctor

# Install Android SDK dependencies (if needed for mobile dev)
# Flutter will prompt for what is missing
```

### 9.9 Zed Editor (AppImage)

```bash
# Download latest Zed AppImage from https://zed.dev/download
# Or use the install script:
curl -fsSL https://zed.dev/install.sh | sh

# The install script places the binary in ~/.local/bin/
# Verify
zed --version
```

## Step 10: Install System Services

### 10.1 Tailscale

```bash
sudo dnf5 install -y tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
# Follow the browser auth link
```

### 10.2 Syncthing

```bash
# Check EPEL first
sudo dnf5 install -y syncthing || {
  echo "Syncthing not in EPEL 10, installing via Flatpak"
  flatpak install -y flathub me.kozec.syncthingtk
}

# If installed via RPM, enable user service
systemctl --user enable --now syncthing.service

# Access web UI: http://localhost:8384

# Restore Syncthing config from backup if available
# cp -r /mnt/backup/syncthing-config/* ~/.config/syncthing/
```

### 10.3 Ollama

```bash
# Official install script
curl -fsSL https://ollama.com/install.sh | sh

# This installs ollama binary and creates a systemd service
sudo systemctl enable --now ollama

# Verify
ollama --version

# Pull a model to test
ollama pull llama3.2
```

### 10.4 CUPS (Printing)

```bash
sudo dnf5 install -y cups cups-filters
sudo systemctl enable --now cups

# For network printer discovery
sudo dnf5 install -y avahi nss-mdns
sudo systemctl enable --now avahi-daemon
```

### 10.5 Bluetooth

```bash
# Should be installed already, verify
sudo systemctl enable --now bluetooth
```

### 10.6 Power Profiles Daemon

```bash
sudo dnf5 install -y power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon

# Set balanced mode (good for battery)
powerprofilesctl set balanced

# Verify profiles
powerprofilesctl list
```

### 10.7 Firmware Updates (fwupd)

```bash
sudo dnf5 install -y fwupd
sudo systemctl enable --now fwupd

# Check for Framework firmware updates
sudo fwupdmgr refresh
sudo fwupdmgr get-updates
# Apply if available:
# sudo fwupdmgr update
```

### 10.8 lm-sensors

```bash
sudo dnf5 install -y lm_sensors
sudo sensors-detect  # Accept defaults
sensors  # Verify output
```

## Step 11: Install Desktop Applications

### 11.1 RPM-Based Apps

```bash
sudo dnf5 install -y \
  libreoffice \
  thunderbird \
  gimp

# VLC -- from RPM Fusion if available, otherwise Flatpak
sudo dnf5 install -y vlc || flatpak install -y flathub org.videolan.VLC

# Steam -- from RPM Fusion nonfree if available, otherwise Flatpak
sudo dnf5 install -y steam || flatpak install -y flathub com.valvesoftware.Steam
```

### 11.2 Insync (Google Drive / OneDrive)

```bash
# Try RPM repo first (see repo-manifest.md for setup)
sudo dnf5 install -y insync || {
  echo "Insync RPM not available for EL10."
  echo "Use GNOME Online Accounts for Google Drive/OneDrive instead."
  echo "Or download direct RPM from https://www.insync.io/downloads"
}
```

### 11.3 Flatpak Applications

```bash
# Set up Flatpak if not already done
sudo dnf5 install -y flatpak
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Communication
flatpak install -y flathub com.discordapp.Discord

# Notes / PKM
flatpak install -y flathub md.obsidian.Obsidian
flatpak install -y flathub com.logseq.Logseq

# Creative / Media
flatpak install -y flathub com.bambulab.BambuStudio
flatpak install -y flathub org.inkscape.Inkscape
flatpak install -y flathub org.upscayl.Upscayl
flatpak install -y flathub net.mkiol.SpeechNote
flatpak install -y flathub org.nicotine_plus.Nicotine

# Email (alternative to Thunderbird)
flatpak install -y flathub eu.betterbird.Betterbird

# Gaming
flatpak install -y flathub com.valvesoftware.Steam  # If not installed via RPM Fusion

# Utilities
flatpak install -y flathub org.fkoehler.KTailctl  # KDE Tailscale integration
```

### 11.4 Manage Flatpak Permissions with Flatseal

```bash
flatpak install -y flathub com.github.tchx84.Flatseal

# Use Flatseal to grant filesystem access where needed
# E.g., Obsidian may need access to ~/projects/ for vault files
```

## Step 12: Framework 13 AMD Specific Tweaks

### 12.1 Display Scaling

**GNOME:**
```bash
# Via Settings GUI:
# Settings > Displays > Scale > 150%

# Via command line:
gsettings set org.gnome.desktop.interface text-scaling-factor 1.0
# The 150% setting uses Wayland fractional scaling, not text scaling
# It is set through the Mutter display configuration, not gsettings
```

**KDE (if installed):**
```bash
# System Settings > Display and Monitor > Display Configuration > Scale: 150%
# KDE supports finer scaling increments (125%, 137.5%, 150%, etc.)
```

### 12.2 AMD P-State Configuration

```bash
# Verify AMD P-State EPP is active (should be default on kernel 6.12)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
# Expected: amd-pstate-epp

# If not active, add kernel parameter:
sudo grubby --update-kernel=ALL --args="amd_pstate=active"
sudo reboot
```

### 12.3 Battery Charge Limit (Framework EC)

```bash
# Check if framework_laptop kernel module is available
lsmod | grep framework

# If available, set charge limit to 80% for battery longevity
echo 80 | sudo tee /sys/class/power_supply/BAT1/charge_control_end_threshold

# Make persistent via udev rule:
cat > /etc/udev/rules.d/99-battery-charge-limit.rules << 'EOF'
SUBSYSTEM=="power_supply", ATTR{charge_control_end_threshold}="80"
EOF
```

### 12.4 Suspend / Resume

```bash
# RHEL 10 with kernel 6.12 should handle suspend on Framework 13 properly
# If suspend issues occur, try s2idle:
sudo grubby --update-kernel=ALL --args="mem_sleep_default=s2idle"

# Verify suspend mode
cat /sys/power/mem_sleep
# Expected: [s2idle] deep
# (brackets around the active mode)
```

### 12.5 Wi-Fi Power Save (Disable for Stability)

```bash
# MediaTek Wi-Fi on Framework 13 can be flaky with power save enabled
cat > /etc/NetworkManager/conf.d/wifi-powersave-off.conf << 'EOF'
[connection]
wifi.powersave = 2
EOF

sudo systemctl restart NetworkManager
```

## Step 13: Set Up Btrfs Snapshots (Snapper)

This provides your rollback safety net. Strongly recommended.

```bash
sudo dnf5 install -y snapper

# Create Snapper config for root filesystem
sudo snapper -c root create-config /

# Configure automatic timeline snapshots
sudo sed -i 's/TIMELINE_CREATE="no"/TIMELINE_CREATE="yes"/' /etc/snapper/configs/root
sudo sed -i 's/TIMELINE_CLEANUP="no"/TIMELINE_CLEANUP="yes"/' /etc/snapper/configs/root

# Set retention policy
sudo sed -i 's/TIMELINE_LIMIT_HOURLY=.*/TIMELINE_LIMIT_HOURLY="5"/' /etc/snapper/configs/root
sudo sed -i 's/TIMELINE_LIMIT_DAILY=.*/TIMELINE_LIMIT_DAILY="7"/' /etc/snapper/configs/root
sudo sed -i 's/TIMELINE_LIMIT_WEEKLY=.*/TIMELINE_LIMIT_WEEKLY="4"/' /etc/snapper/configs/root
sudo sed -i 's/TIMELINE_LIMIT_MONTHLY=.*/TIMELINE_LIMIT_MONTHLY="3"/' /etc/snapper/configs/root

# Enable Snapper timers
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer

# Test: create a manual snapshot
sudo snapper -c root create -d "Post-install baseline"

# List snapshots
sudo snapper -c root list
```

### Taking Pre/Post Snapshots Around Updates

```bash
# Before a major update:
sudo snapper -c root create -d "pre-update" -t pre
sudo dnf5 upgrade -y
sudo snapper -c root create -d "post-update" -t post

# If update breaks something, rollback:
sudo snapper -c root undochange <pre-number>..<post-number>
sudo reboot
```

## Step 14: Restore User Data from Backup

```bash
# Mount backup drive
sudo mount /dev/sdX1 /mnt/backup

# Restore SSH keys
cp -r /mnt/backup/ssh-keys/.ssh ~/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub

# Restore GPG keys
gpg --import /mnt/backup/gpg-public-keys.asc
gpg --import /mnt/backup/gpg-secret-keys.asc
gpg --import-ownertrust /mnt/backup/gpg-ownertrust.txt

# Restore project files
rsync -avhP /mnt/backup/home-zervin/projects/ ~/projects/

# Restore dotfiles (selective -- do not blindly copy all dotfiles)
cp /mnt/backup/home-zervin/.gitconfig ~/
cp /mnt/backup/home-zervin/.bashrc ~/  # Review before using -- paths may differ
cp /mnt/backup/home-zervin/.bash_aliases ~/

# Restore Syncthing config
mkdir -p ~/.config/syncthing
cp -r /mnt/backup/syncthing-config/* ~/.config/syncthing/
systemctl --user restart syncthing.service

# Unmount backup
sudo umount /mnt/backup
```

## Step 15: SELinux Verification

After all packages are installed, check for SELinux denials:

```bash
# Check for any SELinux denials
sudo ausearch -m AVC -ts boot

# If there are denials, review them:
sudo sealert -a /var/log/audit/audit.log

# For each legitimate denial, create a custom policy:
sudo audit2allow -a -M my-custom-policies
sudo semodule -i my-custom-policies.pp

# Common apps that may need SELinux policy adjustments:
# - Citrix Workspace (ICAClient)
# - Zed editor (AppImage)
# - Ollama (if installed via script)
# - Manual binary installs in ~/.local/bin/

# Verify SELinux is enforcing
getenforce
# Expected: Enforcing

# Do NOT set SELinux to permissive or disabled.
# Fix individual denials with audit2allow instead.
```

## Step 16: Post-Install Verification

Run through this checklist to confirm everything works:

```bash
# --- System ---
echo "=== System ===" 
cat /etc/almalinux-release
uname -r
getenforce
dnf5 repolist | wc -l

# --- Hardware ---
echo "=== Hardware ==="
lspci -k | grep -A2 VGA
sensors
powerprofilesctl list
fwupdmgr get-devices | head -5

# --- Services ---
echo "=== Services ==="
systemctl is-active docker
systemctl is-active tailscaled
systemctl --user is-active syncthing
systemctl is-active ollama
systemctl is-active cups
systemctl is-active bluetooth
systemctl is-active power-profiles-daemon

# --- Dev tools ---
echo "=== Dev Tools ==="
node --version
python3 --version
rustc --version
go version
java -version
bun --version
uv --version
git --version
gh --version
docker --version
podman --version
gcloud --version | head -1

# --- Applications ---
echo "=== Applications ==="
which brave-browser && echo "Brave: OK"
which code && echo "VS Code: OK"
which zoom && echo "Zoom: OK"
which zed && echo "Zed: OK" || echo "Zed: check ~/.local/bin/"
flatpak list --app --columns=application | head -20

# --- Tailscale ---
echo "=== Tailscale ==="
tailscale status | head -5
```

## Step 17: Regular Maintenance

### Weekly

```bash
# Update all RPM packages
sudo dnf5 upgrade -y

# Update Flatpak apps
flatpak update -y

# Update Rust
rustup update

# Update Bun
bun upgrade
```

### Monthly

```bash
# Clean up orphaned packages
sudo dnf5 autoremove -y

# Clean DNF cache
sudo dnf5 clean all

# Remove old Flatpak runtimes
flatpak uninstall --unused -y

# Review installed packages for anything unnecessary
dnf5 repoquery --userinstalled --qf '%{name}' | sort

# Check Snapper snapshot usage
sudo snapper -c root list
sudo btrfs filesystem usage /

# Check for firmware updates
sudo fwupdmgr refresh && sudo fwupdmgr get-updates
```

### Quarterly

```bash
# Review all enabled repositories
dnf5 repolist

# Check for packages from disabled/removed repos
dnf5 list extras

# Verify SELinux denials have not accumulated
sudo ausearch -m AVC -ts recent
```

## Appendix: Package Group Quick Reference

For reinstall or new machine setup, here are all packages grouped logically:

```bash
# Work-critical (manual RPM install)
# ICAClient-*.rpm, zoom_x86_64.rpm, zoomvdi-*.rpm

# Browsers (from vendor repos)
sudo dnf5 install -y brave-browser firefox

# Dev tools (from vendor repos + base)
sudo dnf5 install -y code git gh google-cloud-cli nodejs npm \
  python3 python3-pip golang zulu21-jdk \
  gcc gcc-c++ make cmake ninja-build openssl-devel pkg-config clang llvm

# Containers (Docker from vendor repo, Podman from base)
sudo dnf5 install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin \
  podman podman-compose buildah skopeo

# System services (from vendor repos + base)
sudo dnf5 install -y tailscale syncthing fwupd power-profiles-daemon \
  lm_sensors cups cups-filters avahi nss-mdns

# Desktop apps (from base + EPEL + RPM Fusion)
sudo dnf5 install -y libreoffice thunderbird gimp vlc

# KDE (from EPEL/KDE SIG -- if available)
# sudo dnf5 group install -y "KDE Plasma Workspaces"

# Flatpak apps
flatpak install -y flathub \
  com.discordapp.Discord \
  md.obsidian.Obsidian \
  com.logseq.Logseq \
  com.bambulab.BambuStudio \
  org.inkscape.Inkscape \
  org.upscayl.Upscayl \
  net.mkiol.SpeechNote \
  org.nicotine_plus.Nicotine \
  eu.betterbird.Betterbird \
  org.fkoehler.KTailctl \
  com.github.tchx84.Flatseal

# Manual installs (as user, not root)
# rustup (Rust), bun, uv, Flutter (git clone), Zed (install script), Ollama (install script)
```
