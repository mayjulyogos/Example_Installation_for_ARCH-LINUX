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

# Enable swap
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
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable --disable-shim-lock
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
# 1. Install Sudo
pacman -S --noconfirm sudo

# 2. Create personal user and add to wheel group
useradd -m -G wheel my_username

# 3. Set user password
passwd my_username

# 4. Configure sudoers with strict read-only permissions (0440)
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
chmod 0440 /etc/sudoers.d/wheel
```

<br>

---

<br>

## Configure Swappiness (Optional)
- This is important for reducing the usage of your storage device
- Linux kernel usually set the swappiness to 60 which will prioritize the usage of storage device other than our RAM
```bash
# Check the current swappiness value
cat /proc/sys/vm/swappiness

# Edit `/etc/sysctl.conf` to change the value permanently
# Press Ctrl + o and Ctrl + x to save the configured file
vm.swappiness = 10

# Run this following command line to apply changes
su # Enter root environment
sysctl -p
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

## Step 10. GPU Drivers Installation
* You may install the GPU Driver by following the GPU you use so you don't need to install intel GPU if you are using AMD GPU
```bash
sudo pacman -S nvidia-utils nvidia-settings nvidia-open-dkms linux-headers libva-nvidia-driver libva-utils # nvidia drivers

sudo pacman -S linux-firmware mesa vulkan-intel intel-media-driver libva-utils intel-gpu-tools # intel drivers

sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon # AMD GPU
```

<br>

---

<br>

## CUDA
```bash
sudo pacman -S cuda
```

<br>

---

<br>

## Step 11. Configure Audio and Bluetooth
```bash
sudo pacman -Syu
pacman -S \
pipewire \
wireplumber \
pipewire-alsa \
pipewire-pulse \
pipewire-jack \
alsa-utils \
sof-firmware \
alsa-firmware

pacman -S bluez bluez-utils

systemctl enable --now bluetooth.service

reboot
```

<br>

---

<br>

## Step 12. Firewalld
```bash
# Install firewalld and the graphical configuration tool
sudo pacman -S firewalld network-manager-applet

# Enable and start the service
sudo systemctl enable --now firewalld

sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet rhgb rd.driver.blacklist=nouveau,nova_core modprobe.blacklist=nouveau,nova_core nvidia-drm.modeset=1 nvidia-drm.fbdev=1 acpi_backlight=native apparmor=1 lsm=landlock,lockdown,yama,apparmor,bpf"
# Ctrl + O, Enter, then Ctrl + X

sudo grub-mkconfig -o /boot/grub/grub.cfg

sudo pacman -S firewall-config

# Automatically inject nvidia modules into mkinitcpio.conf for early loading (KMS)
sudo sed -i 's/^MODULES=(\s*/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm /; s/ )/)/' /etc/mkinitcpio.conf

# Regenerate the initramfs
sudo mkinitcpio -P

# Regenerate grub one more time to apply the command line arguments
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
