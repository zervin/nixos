# Migration Runbook: Debian 13 to NixOS

## Pre-Flight Checklist

Before starting, confirm:
- [ ] External backup drive available (USB or network, >500GB free)
- [ ] NixOS minimal ISO downloaded and written to USB stick
- [ ] This config repo cloned and tested in VM (Phase 1 complete)
- [ ] USB stick or second USB with this config repo (or network access to clone it)
- [ ] Framework 13 power adapter plugged in
- [ ] 2-4 hours of uninterrupted time
- [ ] Tailscale auth key noted (or ability to re-auth via browser)
- [ ] Syncthing device IDs documented

## Step 1: Backup Current System

Run these commands on the current Debian system.

### 1.1 Full Home Directory Backup

```bash
# To external USB drive (adjust mount point)
sudo mount /dev/sdX1 /mnt/backup

# Backup entire home directory
rsync -avhP --delete \
  --exclude='.cache' \
  --exclude='.local/share/Trash' \
  --exclude='.local/share/Steam/steamapps' \
  --exclude='snap' \
  /home/zervin/ /mnt/backup/home-zervin/

# Backup list of installed packages for reference
dpkg --get-selections > /mnt/backup/debian-packages.txt
flatpak list --app --columns=application > /mnt/backup/flatpak-apps.txt
systemctl list-unit-files --state=enabled > /mnt/backup/enabled-services.txt
```

### 1.2 Critical Files Checklist

Verify these are in the backup:

```bash
# SSH keys
ls -la /mnt/backup/home-zervin/.ssh/

# GPG keys (export separately for safety)
gpg --export-secret-keys --armor > /mnt/backup/gpg-secret-keys.asc
gpg --export --armor > /mnt/backup/gpg-public-keys.asc
gpg --export-ownertrust > /mnt/backup/gpg-ownertrust.txt

# Projects
ls /mnt/backup/home-zervin/projects/

# Browser profiles (if not using sync)
ls /mnt/backup/home-zervin/.mozilla/
ls /mnt/backup/home-zervin/.config/BraveSoftware/

# KDE config
ls /mnt/backup/home-zervin/.config/plasma*
ls /mnt/backup/home-zervin/.config/kde*

# VS Code settings
ls /mnt/backup/home-zervin/.config/Code/User/settings.json

# Flatpak app data
ls /mnt/backup/home-zervin/.var/app/
```

### 1.3 Document Current State

```bash
# Network configuration
nmcli connection show > /mnt/backup/network-connections.txt

# Syncthing config
cp -r ~/.config/syncthing/ /mnt/backup/syncthing-config/

# Ollama model list
ollama list > /mnt/backup/ollama-models.txt

# Docker volumes (if any critical data)
docker volume ls > /mnt/backup/docker-volumes.txt
```

### 1.4 Verify Backup

```bash
# Spot check critical files
diff ~/.ssh/id_ed25519 /mnt/backup/home-zervin/.ssh/id_ed25519
ls -la /mnt/backup/home-zervin/projects/ | wc -l
du -sh /mnt/backup/home-zervin/
```

## Step 2: Prepare NixOS Installer USB

### 2.1 Download ISO

```bash
# Download NixOS minimal ISO (smaller, CLI installer)
# From: https://nixos.org/download#nixos-iso
# Choose: NixOS Minimal ISO, 64-bit AMD/Intel
wget https://channels.nixos.org/nixos-unstable/latest-nixos-minimal-x86_64-linux.iso
```

### 2.2 Write to USB

```bash
# Identify USB device (BE CAREFUL - wrong device = data loss)
lsblk
# Write ISO
sudo dd if=latest-nixos-minimal-x86_64-linux.iso of=/dev/sdX bs=4M status=progress
sync
```

### 2.3 Prepare Config USB (or plan to clone via network)

Option A: Copy config repo to a second USB stick
```bash
# Format a second USB stick and copy the config
mkfs.ext4 /dev/sdY1
mount /dev/sdY1 /mnt/config
cp -r /home/zervin/projects/nixos/* /mnt/config/
umount /mnt/config
```

Option B: Plan to clone from Git after connecting to WiFi in the installer

## Step 3: Boot NixOS Installer

### 3.1 BIOS/UEFI Setup

1. Power off the Framework laptop
2. Insert NixOS USB stick
3. Power on, press F12 for boot menu (or F2 for BIOS setup)
4. Select USB stick
5. NixOS installer boots to a root shell

### 3.2 Connect to Network

```bash
# WiFi (Framework 13 WiFi should work out of the box)
sudo systemctl start wpa_supplicant
wpa_cli
> scan
> scan_results
> add_network
> set_network 0 ssid "YourWiFi"
> set_network 0 psk "YourPassword"
> enable_network 0
> quit

# Or use nmtui if available
sudo nmtui

# Verify
ping -c 3 nixos.org
```

### 3.3 Enable Nix Flakes in Installer

```bash
export NIX_CONFIG="experimental-features = nix-command flakes"
```

## Step 4: Partition and Encrypt

### 4.1 Identify the NVMe Drive

```bash
lsblk
# Should show /dev/nvme0n1 (the 931.5GB NVMe)
# DANGER: The following steps will WIPE this drive
```

### 4.2 Partition the Drive

```bash
# Wipe existing partition table
sgdisk --zap-all /dev/nvme0n1

# Create EFI partition (512MB)
sgdisk -n 1:0:+512M -t 1:EF00 -c 1:"EFI" /dev/nvme0n1

# Create main partition (rest of disk)
sgdisk -n 2:0:0 -t 2:8300 -c 2:"NixOS" /dev/nvme0n1

# Verify
sgdisk -p /dev/nvme0n1
```

### 4.3 Set Up LUKS Encryption

```bash
# Format as LUKS2 (you will be prompted for a passphrase)
cryptsetup luksFormat --type luks2 /dev/nvme0n1p2

# Open the LUKS container
cryptsetup open /dev/nvme0n1p2 cryptroot
```

### 4.4 Create Btrfs Filesystem with Subvolumes

```bash
# Format as btrfs
mkfs.btrfs -L nixos /dev/mapper/cryptroot

# Mount and create subvolumes
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@nix
btrfs subvolume create /mnt/@swap
umount /mnt

# Mount subvolumes with compression
mount -o subvol=@,compress=zstd,noatime /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,nix,swap,boot}
mount -o subvol=@home,compress=zstd,noatime /dev/mapper/cryptroot /mnt/home
mount -o subvol=@nix,compress=zstd,noatime /dev/mapper/cryptroot /mnt/nix
mount -o subvol=@swap,noatime /dev/mapper/cryptroot /mnt/swap

# Format and mount EFI partition
mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
mount /dev/nvme0n1p1 /mnt/boot
```

### 4.5 Optional: Create Swap File

```bash
# With 54GB RAM, swap is optional but useful for hibernation
# Create a 16GB swap file (or skip if not hibernating)
btrfs filesystem mkswapfile --size 16g /mnt/swap/swapfile
swapon /mnt/swap/swapfile
```

## Step 5: Install NixOS

### 5.1 Generate Hardware Configuration

```bash
nixos-generate-config --root /mnt
# This creates:
#   /mnt/etc/nixos/configuration.nix (we will replace this)
#   /mnt/etc/nixos/hardware-configuration.nix (we KEEP this)
```

### 5.2 Get Your Configuration

```bash
# Option A: Clone from Git
nix-shell -p git
cd /mnt
git clone <your-repo-url> /mnt/etc/nixos-config

# Option B: Copy from USB
mount /dev/sdY1 /mnt2
cp -r /mnt2/* /mnt/etc/nixos-config/
umount /mnt2
```

### 5.3 Copy Hardware Configuration

```bash
# Copy the generated hardware-configuration.nix into your config
cp /mnt/etc/nixos/hardware-configuration.nix \
   /mnt/etc/nixos-config/hosts/framework/hardware-configuration.nix
```

### 5.4 Review Hardware Configuration

```bash
# Verify LUKS device is detected
cat /mnt/etc/nixos-config/hosts/framework/hardware-configuration.nix
# Look for:
#   boot.initrd.luks.devices."cryptroot" = { device = "/dev/disk/by-uuid/..."; };
# and correct filesystem mounts with subvolume options
```

### 5.5 Install

```bash
# Install NixOS using your flake configuration
nixos-install --flake /mnt/etc/nixos-config#framework --no-root-passwd

# If using traditional configuration.nix instead of flakes:
# nixos-install --root /mnt

# Set the root password when prompted (or use --no-root-passwd if your config sets it)
```

### 5.6 Post-Install Setup Before Reboot

```bash
# Copy your config repo to the installed system for future use
cp -r /mnt/etc/nixos-config /mnt/home/zervin/projects/nixos
# Fix ownership (will be corrected after first boot, but helps)
nixos-enter --root /mnt -c 'chown -R zervin:users /home/zervin/projects/nixos'
```

### 5.7 Reboot

```bash
umount -R /mnt
reboot
# Remove USB stick when system restarts
```

## Step 6: First Boot

### 6.1 LUKS Prompt

- System will prompt for LUKS passphrase at boot
- Enter the passphrase set in Step 4.3
- SDDM login screen should appear with KDE Plasma session

### 6.2 Login and Verify

```bash
# Log in as your user
# Open Konsole and verify:

# System basics
uname -a          # Should show NixOS kernel
nixos-version     # Should show version

# Desktop
echo $XDG_SESSION_TYPE    # Should be "wayland"
echo $DESKTOP_SESSION     # Should be "plasma" or "plasmawayland"

# Services
systemctl status docker
systemctl status ollama
systemctl status tailscaled
systemctl status syncthing
systemctl status cups

# GPU
glxinfo | grep "OpenGL renderer"   # Should show AMD Radeon 780M
vulkaninfo | grep "deviceName"     # Should show RADV PHOENIX

# Audio
wpctl status   # Should show PipeWire + WirePlumber

# Network
ip addr        # Should have network interfaces
```

## Step 7: Restore Data

### 7.1 Mount Backup Drive

```bash
sudo mount /dev/sdX1 /mnt/backup
```

### 7.2 Restore Files

```bash
# SSH keys
cp -r /mnt/backup/home-zervin/.ssh/ ~/.ssh/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub

# GPG keys
gpg --import /mnt/backup/gpg-secret-keys.asc
gpg --import /mnt/backup/gpg-public-keys.asc
gpg --import-ownertrust /mnt/backup/gpg-ownertrust.txt

# Projects
rsync -avhP /mnt/backup/home-zervin/projects/ ~/projects/

# Documents, Pictures, etc.
rsync -avhP /mnt/backup/home-zervin/Documents/ ~/Documents/
rsync -avhP /mnt/backup/home-zervin/Pictures/ ~/Pictures/
rsync -avhP /mnt/backup/home-zervin/Downloads/ ~/Downloads/

# VS Code settings
mkdir -p ~/.config/Code/User/
cp /mnt/backup/home-zervin/.config/Code/User/settings.json ~/.config/Code/User/
cp /mnt/backup/home-zervin/.config/Code/User/keybindings.json ~/.config/Code/User/

# Browser profiles (if not using sync)
rsync -avhP /mnt/backup/home-zervin/.mozilla/ ~/.mozilla/
rsync -avhP /mnt/backup/home-zervin/.config/BraveSoftware/ ~/.config/BraveSoftware/

# Flatpak app data
rsync -avhP /mnt/backup/home-zervin/.var/app/ ~/.var/app/

# Syncthing config (device IDs and folder config)
rsync -avhP /mnt/backup/syncthing-config/ ~/.config/syncthing/
```

### 7.3 Restore KDE Settings (Optional)

```bash
# Only if you want to preserve KDE customizations
# WARNING: Some paths may differ between Debian and NixOS KDE
rsync -avhP /mnt/backup/home-zervin/.config/plasma* ~/.config/
rsync -avhP /mnt/backup/home-zervin/.config/kde* ~/.config/
# Log out and back in for KDE to pick up the settings
```

## Step 8: Authenticate Services

### 8.1 Tailscale

```bash
sudo tailscale up
# Opens browser for authentication
# Or use auth key:
# sudo tailscale up --authkey=tskey-auth-xxxxx
```

### 8.2 Syncthing

```bash
# Open Syncthing web UI
xdg-open http://localhost:8384
# Re-add devices and folders as needed
# Or if you restored the Syncthing config, devices should re-appear
# but you may need to re-accept on remote devices
```

### 8.3 Docker

```bash
# Verify Docker works
docker run hello-world

# If you need Docker Hub auth:
docker login
```

### 8.4 Ollama Models

```bash
# Re-pull all models (these are cloud-proxied, so fast)
ollama pull haiku
ollama pull opus
ollama pull sonnet
ollama pull qwen3-coder-next
ollama pull mistral-large-3
ollama pull qwen3-vl
ollama pull kimi-k2.5
ollama pull glm-5
ollama pull minimax-m2.5
```

### 8.5 Flatpak Apps

```bash
# If not using nix-flatpak declarative management:
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Install Flatpak apps not in nixpkgs
flatpak install flathub eu.betterbird.Betterbird
flatpak install flathub net.mkiol.SpeechNote
# Add others as needed based on your Flatpak list
```

### 8.6 Browser Logins

Open each browser and log into:
- Google account (for Chrome sync, Gmail, Google Drive)
- Firefox Sync (restores bookmarks, passwords, extensions)
- Brave Sync
- Any web apps (GitHub, Tailscale admin, etc.)

### 8.7 Application Logins

- Discord: Log in
- Zoom: Log in
- Teams: Log in
- Insync: Log in to Google account
- Obsidian: Open vault from ~/projects or ~/Documents
- Logseq: Open graph from filesystem

### 8.8 Development Tools

```bash
# Verify dev tools
node --version
python3 --version
rustc --version
go version
java --version
bun --version
uv --version
gh auth login
gcloud auth login

# Flutter (if configured)
flutter doctor
```

## Step 9: Verify Everything Works

### 9.1 Checklist

Run through each category and confirm:

**Desktop:**
- [ ] KDE Plasma 6 running on Wayland
- [ ] SDDM login screen works
- [ ] Multi-monitor works (if applicable)
- [ ] Screen brightness controls work
- [ ] Volume controls work
- [ ] WiFi connects and reconnects
- [ ] Bluetooth pairs with devices
- [ ] KDE Connect detects phone

**Applications:**
- [ ] Brave opens and has sync data
- [ ] Firefox ESR opens and has sync data
- [ ] VS Code opens, extensions loaded
- [ ] Zed editor opens
- [ ] GIMP opens
- [ ] LibreOffice opens
- [ ] VLC plays media
- [ ] Steam launches and detects games
- [ ] Bambu Studio opens

**Services:**
- [ ] Docker: `docker ps` works
- [ ] Ollama: `ollama list` shows models
- [ ] Tailscale: `tailscale status` shows connected
- [ ] Syncthing: Web UI accessible, folders syncing
- [ ] CUPS: Test print page

**Hardware:**
- [ ] GPU acceleration (run `glxgears` or check `about:support` in Firefox)
- [ ] Audio output (speakers and headphone jack)
- [ ] Webcam (`cheese` or similar)
- [ ] Suspend and resume (close lid, reopen)
- [ ] Keyboard backlight
- [ ] Fingerprint reader (if configured)

**Development:**
- [ ] Git clone a repo, make changes, push
- [ ] Run a Docker Compose project
- [ ] Build a Rust project
- [ ] Run a Node.js project
- [ ] Python virtual environment with uv

## Step 10: Post-Migration Maintenance

### 10.1 Set Up Config Repo

```bash
cd ~/projects/nixos
git add -A
git commit -m "Working NixOS configuration post-migration"
git push
```

### 10.2 Learn Update Workflow

```bash
# Update all inputs (nixpkgs, home-manager, etc.)
cd ~/projects/nixos
nix flake update

# Build without switching (safe test)
sudo nixos-rebuild build --flake .#framework

# Switch to new build
sudo nixos-rebuild switch --flake .#framework

# If something breaks:
sudo nixos-rebuild switch --rollback

# Or select previous generation at boot menu
```

### 10.3 Garbage Collection

```bash
# Remove old generations (keeps current + 5 previous by default if configured)
sudo nix-collect-garbage --delete-older-than 30d

# Or manually list and delete generations
sudo nix-env --list-generations --profile /nix/var/nix/profiles/system
sudo nix-env --delete-generations +5 --profile /nix/var/nix/profiles/system
```

### 10.4 Adding New Software

To add a new package:
1. Edit the appropriate module file (e.g., `home/zervin/apps.nix`)
2. Add the package name to the list
3. Run `sudo nixos-rebuild switch --flake .#framework`
4. Commit the change to Git

To try a package temporarily without installing:
```bash
nix run nixpkgs#cowsay -- "Hello NixOS"
nix shell nixpkgs#htop nixpkgs#ncdu  # Drop into shell with packages
```

## Troubleshooting

### Boot fails after rebuild
Select a previous generation from the boot menu (systemd-boot shows all generations). Then fix the config and rebuild.

### Package not found
```bash
nix search nixpkgs#<package-name>
# Or browse https://search.nixos.org/packages
```

### Service not starting
```bash
systemctl status <service-name>
journalctl -u <service-name> -e
```

### LUKS not prompting for password
Check `hardware-configuration.nix` for `boot.initrd.luks.devices` entry. It must reference the correct UUID of the LUKS partition.

### WiFi not working in installer
Framework 13 AMD uses MediaTek WiFi (mt7921e). This is supported in the NixOS kernel by default. If issues arise:
```bash
# Load module manually
modprobe mt7921e
# Check dmesg for errors
dmesg | grep mt7921
```

### Flatpak apps have no system integration
Ensure XDG portals are configured:
```nix
xdg.portal = {
  enable = true;
  extraPortals = [ pkgs.xdg-desktop-portal-kde ];
};
```
This should be handled automatically by the KDE Plasma module.

### Ollama GPU acceleration not working
```bash
# Check if ROCm is available for AMD GPU
ollama ps
# The Radeon 780M may have limited ROCm support
# For the cloud-proxied models this does not matter
```
