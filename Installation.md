# Arch Linux Installation

<br>

## Step 1. Set Keyboard Layout (Optional)
```bash
ls /usr/share/kbd/keymaps/**/us*.map.gz
loadkeys us
echo KEYMAP=us > /etc/vconsole.conf
```

<br>

---

<br>

## Step 2. Verify Boot Mode
* If the directory exists and doesn't error, you are in UEFI mode. If it doesn't exist, you are using BIOS mode and please keep attention of the following steps which there are some perts will be specified for BIOS mode (so there are some changes).
```bash
ls /sys/firmware/efi/efivars 
```

<br>

---

<br>

## Step 3. Test Internet Connection & Set System Clock
```bash
timedatectl set-ntp true
timedatectl status

ping -c 4 google.com
```

<br>

---

<br>

## Step 4. Network configuration
* IWCTL
```bash
iwctl
device list # Find your wireless device name
station wlan0 scan #scan for networks (replace wlan0 with your actual device name)
station wlan0 get-networks #list available networks
station wlan0 connect MyWiFiName # Connect to your Wi-Fi (replace MyWiFiName with your actual network SSID)

exit
```
*  If you are plugged into Ethernet
```bash
systemctl restart systemd-networkd
```

<br>

---

<br>

## Step 5. Format the disk and create partitions
* For UEFI mode
```bash
# Partition the disk
cfdisk /dev/nvme0n1

# Inside cfdisk create:
# 1. nvme0n1p1: 1 GB EFI System partition (Type: EFI System)
# 2. nvme0n1p2: 16 GB Linux Swap partition (Type: Linux swap)
# 3. nvme0n1p3: Remaining space for Linux Root (Type: Linux root x86-64)

# Format partitions
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.btrfs -f /dev/nvme0n1p3

# Mount the root partition temporarily to create subvolumes
mount /dev/nvme0n1p3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
umount /mnt

# Mount subvolumes with optimized Btrfs mount options
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot/efi,home,var/cache,var/log}

mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/home
mount -o noatime,compress=zstd,subvol=@cache /dev/nvme0n1p3 /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@log /dev/nvme0n1p3 /mnt/var/log

# Mount EFI to /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
swapon /dev/nvme0n1p2
```
<br>

* For BIOS mode
```bash
# Partition the disk with cfdisk
cfdisk /dev/nvme0n1

# Create the following layout:
# nvme0n1p1: 1 GB partition (Type: BIOS boot) — Leave this unformatted; GRUB uses raw space here.
# nvme0n1p2: 16 GB Linux Swap partition (Type: Linux swap)
# nvme0n1p3: Remaining space (Type: Linux root x86-64)

# Format swap and root (do NOT format nvme0n1p1)
mkswap /dev/nvme0n1p2
mkfs.btrfs -f /dev/nvme0n1p3

# Mount root partition temporarily to create subvolumes
mount /dev/nvme0n1p3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
umount /mnt

# Mount subvolumes with Btrfs options
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{home,var/cache,var/log} # Note: boot/efi is NOT needed for BIOS

mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/home
mount -o noatime,compress=zstd,subvol=@cache /dev/nvme0n1p3 /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@log /dev/nvme0n1p3 /mnt/var/log

# Enable Swap
swapon /dev/nvme0n1p2
```

<br>

### Btrfs structure
```text
Btrfs System Layout
├── @           ---> /               (Root System)
├── @home       ---> /home           (User Data)
├── @cache      ---> /var/cache      (Package/App Caches)
├── @log        ---> /var/log        (System Logs)
└── @snapshots  ---> /.snapshots     (Snapper Restore Points)
```

<br>

---

<br>

## Step 6. Install base System 
```bash
pacstrap -K /mnt base linux linux-firmware nano btrfs-progs

# Generate filesystem table
genfstab -U /mnt >> /mnt/etc/fstab
```

<br>

---

<br>

## Step 7. Enter the Chroot Environment 
* For UEFI mode
```bash
arch-chroot /mnt

# Hostname
echo my-arch-laptop > /etc/hostname

# --- FIXED LOCALE SETUP (Automated sed to eliminate typos) ---
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
export LANG=en_US.UTF-8
export LC_ALL=C.UTF-8

# CPU Microcode
pacman -S --noconfirm intel-ucode # Or amd-ucode for AMD CPUs

# Install GRUB
pacman -S --noconfirm grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

<br>

* For BIOS mode
```bash
arch-chroot /mnt

# Hostname
echo my-arch-laptop > /etc/hostname

# --- FIXED LOCALE SETUP (Automated sed to eliminate typos) ---
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
export LANG=en_US.UTF-8
export LC_ALL=C.UTF-8

# CPU Microcode
pacman -S --noconfirm intel-ucode # Or amd-ucode for AMD CPUs

# Install GRUB to MBR / BIOS drive (Target whole disk, NOT partition)
pacman -S --noconfirm grub os-prober
grub-install --target=i386-pc /dev/nvme0n1
grub-mkconfig -o /boot/grub/grub.cfg
```

<br>

---

<br>

## Step 8. User Management
```bash
# Set Root Password
passwd

# Create Personal User
useradd -m -G wheel my_username
passwd my_username

# Seamlessly enable wheel group permissions without using interactive visudo
pacman -S --noconfirm sudo
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel

# Enable Network Management service
pacman -S networkmanager
systemctl enable NetworkManager

# How to use NetworkManager
nmtui
```

<br>

---

<br>

## Step 9. Clean up and First boot
```bash
exit
umount -R /mnt
reboot
```

<br>

---

<br>

## Step 10. Install GNOME, GDM and GPU DRIVER
* You may install the GPU Driver by following the GPU you use so you don't need to install intel GPU if you are using AMD GPU
```bash
sudo pacman -S gnome gdm # install gnome and gdm

sudo pacman -S nvidia-utils nvidia-settings nvidia-open-dkms linux-headers libva-nvidia-driver libva-utils # nvidia drivers

sudo pacman -S linux-firmware mesa vulkan-intel intel-media-driver libva-utils intel-gpu-tools # intel drivers

sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon # AMD GPU

sudo systemctl enable gdm # enable login screen

systemctl reboot -i # reboot into your new desktop
```

<br>

---

<br>

## Step 11. CUDA (Optional)
```bash
# Install CUDA
sudo pacman -S cuda
```

<br>

---

<br>

## Step 12. Configure Audio and Bluetooth
```bash
sudo pacman -Syu
sudo pacman -S \
pipewire \
wireplumber \
pipewire-alsa \
pipewire-pulse \
pipewire-jack \
alsa-utils \
sof-firmware \
alsa-firmware

sudo pacman -S bluez bluez-utils

sudo systemctl enable --now bluetooth.service

reboot
```

<br>

---

<br>

# Step 13. Firewalld & GNOME Integration
```bash
# Install firewalld and the graphical configuration tool
sudo pacman -S firewalld network-manager-applet

# Enable and start the service
sudo systemctl enable --now firewalld

# Install AppArmor and the default profiles
sudo pacman -S apparmor

# Enable the systemd service
sudo systemctl enable apparmor.service

sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet rhgb rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core nvidia-drm.modeset=1 nvidia-drm.fbdev=1 acpi_backlight=native apparmor=1 lsm=landlock,lockdown,yama,apparmor,bpf"
# Ctrl + O, Enter, then Ctrl + X

sudo grub-mkconfig -o /boot/grub/grub.cfg

sudo pacman -S firewall-config

# Automatically inject nvidia modules into mkinitcpio.conf for early loading (KMS)
sudo sed -i 's/^MODULES=(/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm /' /etc/mkinitcpio.conf

# Regenerate the initramfs
sudo mkinitcpio -P

# Regenerate grub one more time to apply the command line arguments
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

<br>

---

<br>

## Step 14. Snapper + grub-btrfs Atomic Rollback System

* For UEFI mode

This gives you Fedora Atomic / NixOS-style rollbacks on top of your existing btrfs layout (`@`, `@home`, `@cache`, `@log`). Every `pacman` transaction from here on automatically creates a bootable restore point — no manual snapshotting needed for normal use.

### Step 14.1. Install the packages
```bash
sudo pacman -S --noconfirm snapper grub-btrfs inotify-tools snap-pac
```
- `snapper` — creates and manages the snapshots
- `grub-btrfs` — adds a "snapshots" submenu to GRUB so you can boot into any snapshot
- `snap-pac` — automatically triggers a snapshot before and after every pacman transaction
- `inotify-tools` — required by the `grub-btrfsd` daemon that watches for new snapshots

<br>

### Step 14.2. Create a dedicated `@snapshots` subvolume
Your original partitioning (Step 5) didn't create a subvolume for snapshots, so add one now. This must be a sibling of `@`, not nested inside it, or recursive snapshotting will break.
```bash
# Temporarily mount the top-level (unnamed) subvolume of the btrfs filesystem
sudo mkdir -p /mnt/btrfs-root
sudo mount -o subvol=/ /dev/nvme0n1p3 /mnt/btrfs-root

# Create the new subvolume as a sibling of @, @home, @cache, @log
sudo btrfs subvolume create /mnt/btrfs-root/@snapshots

# Unmount, we're done with the top-level view
sudo umount /mnt/btrfs-root
```

<br>

### Step 14.3. Mount `@snapshots` at `/.snapshots`
```bash
sudo mkdir -p /.snapshots

# Add it to fstab so it persists across reboots
echo "/dev/nvme0n1p3 /.snapshots btrfs noatime,compress=zstd,subvol=@snapshots 0 0" | sudo tee -a /etc/fstab

sudo mount -a
```

<br>

### Step 14.4. Initialize the snapper config
`snapper create-config` wants to create its own `.snapshots` subvolume, so we let it create one, then swap in the one we already mounted.
```bash
# 1. Remove the empty directory (if it exists) to ensure a clean mount point
sudo rm -rf /.snapshots

# 2. Generate the root config (Snapper will automatically create a default /.snapshots subvolume)
sudo snapper -c root create-config /

# 3. Delete the automatically generated subvolume so we can replace it with our @snapshots subvolume
sudo btrfs subvolume delete /.snapshots

# 4. Recreate the mount point directory
sudo mkdir -p /.snapshots

# 5. Mount all filesystems listed in /etc/fstab (mounts @snapshots to /.snapshots)
sudo mount -a

# 6. Apply strict permission settings
sudo chmod 750 /.snapshots
```

<br>

### Step 14.5. Tune snapshot retention (optional but recommended)
```bash
sudo nano /etc/snapper/configs/root
```
Set these values:
```
TIMELINE_CREATE="yes"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="4"
TIMELINE_LIMIT_MONTHLY="3"
```
Ctrl + O, Enter, then Ctrl + X to save and exit.

Enable the timers that create and clean up timeline snapshots:
```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

<br>

### Step 14.6. Wire snapshots into GRUB
```bash
# Auto-regenerates grub.cfg whenever a new snapshot is created
sudo systemctl enable --now grub-btrfsd

# Build the initial snapshot submenu (your EFI is at /boot)
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
From this point on, `grub-btrfsd` watches `/.snapshots` and regenerates `/boot/grub/grub.cfg` automatically every time `snap-pac` creates a snapshot. You don't need to re-run `grub-mkconfig` manually going forward.

<br>

### Step 14.7. (Optional) Track `/home` separately
Because `@home` is its own subvolume, the root config above does **not** cover it — a system rollback won't touch your personal files, which is usually what you want. If you'd also like undo history for `/home`:
```bash
sudo snapper -c home create-config /home
```
It shares the same `snapper-timeline.timer` and `snapper-cleanup.timer` already enabled above.

<br>

### Step 14.8. Make the snapshot submenu easy to find and keep it short
grub-btrfs always nests snapshots inside a submenu — there's no supported config option to fully flatten them onto the main GRUB page (this isn't exposed by the tool, despite some guides online claiming otherwise). What you *can* do is name the submenu clearly and keep it short so it's fast to use:
```bash
sudo nano /etc/default/grub-btrfs/config
```
Give the submenu a distinct name so it stands out on the boot screen:
```
GRUB_BTRFS_SUBMENUNAME="Rollback to Snapshot"
```
Cap how many snapshots populate the menu, so it doesn't get flooded over time (timeline snapshots accumulate fast with the hourly/daily retention enabled in Step 13.5):
```
GRUB_BTRFS_LIMIT="10"
```
Save (Ctrl + O, Enter) and exit (Ctrl + X), then regenerate GRUB once to apply it:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
From now on `grub-btrfsd` keeps this configuration applied automatically as new snapshots are created — no need to repeat this step. In practice, this means one extra Enter press at boot to open the submenu, then pick your snapshot — there's no safe way around that extra step without hand-patching a file that pacman will overwrite on the next `grub-btrfs` update.

<br>

### How to use it day to day

**List snapshots:**
```bash
sudo snapper -c root list
```

**Take a manual snapshot before something risky** (e.g. before Step 10's Nvidia driver install, or before a `yay -Syu`):
```bash
sudo snapper -c root create --description "new regular snapshot"
```

Boot into a snapshot to inspect it (non-destructive, read-only):
Reboot → in the GRUB menu select `Arch Linux snapshots` → pick the timestamped entry.

**Actually roll back to a snapshot** (your current state is preserved as a new snapshot first, so this is non-destructive):
```bash
sudo snapper -c root list          # find the snapshot number, e.g. 42
sudo snapper -c root rollback 42
sudo reboot
```

**Diff what changed between two snapshots:**
```bash
sudo snapper -c root status 40..42
```

SAFE:
Boot snapshot → inspect

DESTRUCTIVE:
snapper rollback → change system state

No changes are needed to any pacman/yay commands elsewhere in this guide — `snap-pac` hooks into pacman transactions automatically, including AUR builds installed via `yay` since it calls pacman under the hood.

<br>

### If unable to boot into system
```
# 1. Mount your main @ subvolume to /mnt
mount -o subvol=@ /dev/nvme0n1p3 /mnt

# 2. Ensure snapshot & EFI mount points exist inside /mnt
mkdir -p /mnt/.snapshots
mkdir -p /mnt/boot/efi

# 3. Mount the snapshots subvolume and EFI partition
mount -o subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots
mount /dev/nvme0n1p1 /mnt/boot/efi

# 4. Chroot into your actual system
arch-chroot /mnt

# 5. List and restore your target snapshot
snapper -c root list
snapper -c root rollback <snapshot_number>

# 6. Exit chroot, unmount cleanly, and reboot
exit
umount -R /mnt
reboot
```

<br>

---

<br>

* For BIOS mode

### Step 14.1. Install the packages
```bash
sudo pacman -S --noconfirm snapper grub-btrfs inotify-tools snap-pac
```

- `snapper` — creates and manages the snapshots[cite: 1]
- `grub-btrfs` — adds a "snapshots" submenu to GRUB so you can boot into any snapshot[cite: 1]
- `snap-pac` — automatically triggers a snapshot before and after every pacman transaction[cite: 1]
- `inotify-tools` — required by the `grub-btrfsd` daemon that watches for new snapshots[cite: 1]

<br>

### Step 14.2. Create a dedicated `@snapshots` subvolume
Your original partitioning didn't create a subvolume for snapshots, so add one now[cite: 1]. This must be a sibling of `@`, not nested inside it, or recursive snapshotting will break[cite: 1].
```bash
# Temporarily mount the top-level (unnamed) subvolume of the btrfs filesystem
sudo mkdir -p /mnt/btrfs-root
sudo mount -o subvol=/ /dev/nvme0n1p3 /mnt/btrfs-root

# Create the new subvolume as a sibling of @, @home, @cache, @log
sudo btrfs subvolume create /mnt/btrfs-root/@snapshots

# Unmount, we're done with the top-level view
sudo umount /mnt/btrfs-root
```

<br>

### Step 14.3. Mount `@snapshots` at `/.snapshots`
```bash
sudo mkdir -p /.snapshots

# Add it to fstab so it persists across reboots
echo "/dev/nvme0n1p3 /.snapshots btrfs noatime,compress=zstd,subvol=@snapshots 0 0" | sudo tee -a /etc/fstab

sudo mount -a
```

<br>

### Step 14.4. Initialize the snapper config
`snapper create-config` wants to create its own `.snapshots` subvolume, so we let it create one, then swap in the one we already mounted[cite: 1].
```bash
# 1. Remove the empty directory (if it exists) to ensure a clean mount point
sudo rm -rf /.snapshots

# 2. Generate the root config (Snapper will automatically create a default /.snapshots subvolume)
sudo snapper -c root create-config /

# 3. Delete the automatically generated subvolume so we can replace it with our @snapshots subvolume
sudo btrfs subvolume delete /.snapshots

# 4. Recreate the mount point directory
sudo mkdir -p /.snapshots

# 5. Mount all filesystems listed in /etc/fstab (mounts @snapshots to /.snapshots)
sudo mount -a

# 6. Apply strict permission settings
sudo chmod 750 /.snapshots
```

<br>

### Step 14.5. Tune snapshot retention (optional but recommended)
```bash
sudo nano /etc/snapper/configs/root
```
Set these values:
```text
TIMELINE_CREATE="yes"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="4"
TIMELINE_LIMIT_MONTHLY="3"
```
Press `Ctrl + O`, `Enter`, then `Ctrl + X` to save and exit.

Enable the timers that create and clean up timeline snapshots:
```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

<br>

### Step 14.6. Wire snapshots into GRUB
```bash
# Auto-regenerates grub.cfg whenever a new snapshot is created
sudo systemctl enable --now grub-btrfsd

# Build the initial snapshot submenu
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
From this point on, `grub-btrfsd` watches `/.snapshots` and regenerates `/boot/grub/grub.cfg` automatically every time `snap-pac` creates a snapshot[cite: 1]. You don't need to re-run `grub-mkconfig` manually going forward[cite: 1].

<br>

### Step 14.7. (Optional) Track `/home` separately
Because `@home` is its own subvolume, the root config above does **not** cover it — a system rollback won't touch your personal files, which is usually what you want[cite: 1]. If you'd also like undo history for `/home`:
```bash
sudo snapper -c home create-config /home
```
It shares the same `snapper-timeline.timer` and `snapper-cleanup.timer` already enabled above[cite: 1].

<br>

### Step 14.8. Make the snapshot submenu easy to find and keep it short
grub-btrfs always nests snapshots inside a submenu[cite: 1]. What you *can* do is name the submenu clearly and keep it short so it's fast to use:
```bash
sudo nano /etc/default/grub-btrfs/config
```
Give the submenu a distinct name so it stands out on the boot screen:
```text
GRUB_BTRFS_SUBMENUNAME="Rollback to Snapshot"
```
Cap how many snapshots populate the menu, so it doesn't get flooded over time:
```text
GRUB_BTRFS_LIMIT="10"
```
Save (`Ctrl + O`, `Enter`) and exit (`Ctrl + X`), then regenerate GRUB once to apply it:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

<br>

### How to use it day to day

**List snapshots:**
```bash
sudo snapper -c root list
```

**Take a manual snapshot before something risky:**
```bash
sudo snapper -c root create --description "new regular snapshot"
```

**Boot into a snapshot to inspect it (non-destructive, read-only):**
Reboot → in the GRUB menu select `Rollback to Snapshot` → pick the timestamped entry[cite: 1].

**Actually roll back to a snapshot:**
```bash
sudo snapper -c root list          # find the snapshot number, e.g. 42
sudo snapper -c root rollback 42
sudo reboot
```

**Diff what changed between two snapshots:**
```bash
sudo snapper -c root status 40..42
```

<br>

### If unable to boot into system (BIOS Emergency Recovery)
If your system becomes completely unbootable, boot from your Arch Linux Installation USB and perform the recovery steps below. *(Note: Unlike UEFI, you do not need to mount an EFI system partition).*

```bash
# 1. Mount your main @ subvolume to /mnt
mount -o subvol=@ /dev/nvme0n1p3 /mnt

# 2. Ensure snapshot mount point exists inside /mnt
mkdir -p /mnt/.snapshots

# 3. Mount the snapshots subvolume
mount -o subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots

# 4. Chroot into your actual system
arch-chroot /mnt

# 5. List and restore your target snapshot
snapper -c root list
snapper -c root rollback <snapshot_number>

# 6. Exit chroot, unmount cleanly, and reboot
exit
umount -R /mnt
reboot
```
