# Increase console font size
setfont ter-132b

# Check internet connection
ping -c 4 ping.archlinux.org

# Check time synchronization
timedatectl status
timedatectl set-ntp true

# Check storage devices
lsblk -f 

# Wipe existing partition structures
sgdisk --zap-all /dev/nvme0n1

# Create GPT partition table
parted -s /dev/nvme0n1 mklabel gpt

# Partition 1: EFI — 1 GiB
parted -s /dev/nvme0n1 mkpart ESP fat32 1MiB 1024MiB
parted -s /dev/nvme0n1 set 1 esp on

# Partition 2: Swap — 4 GiB
parted -s /dev/nvme0n1 mkpart primary linux-swap 1024MiB 5120MiB

# Partition 3: Btrfs — remaining space
parted -s /dev/nvme0n1 mkpart primary btrfs 5120MiB 100%

# Verify partition layout
parted /dev/nvme0n1 print
lsblk -f

# EFI
mkfs.fat -F32 -n "EFI" /dev/nvme0n1p1

# Swap
mkswap -L "ARCH_SWAP" /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# Btrfs
mkfs.btrfs -f -L "ARCH_LINUX" /dev/nvme0n1p3

mount /dev/nvme0n1p3 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@pkg

btrfs subvolume list /mnt

umount /mnt

BTRFS_OPTS="noatime,compress=zstd:1,discard=async"

# Mount root subvolume
mount -o "$BTRFS_OPTS,subvol=@" /dev/nvme0n1p3 /mnt

# Create mount points
mkdir -p /mnt/boot/efi
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/var/log
mkdir -p /mnt/var/cache/pacman/pkg
mkdir -p /mnt/mnt/wdblue

# Mount Btrfs subvolumes
mount -o "$BTRFS_OPTS,subvol=@home" /dev/nvme0n1p3 /mnt/home
mount -o "$BTRFS_OPTS,subvol=@snapshots" /dev/nvme0n1p3 /mnt/.snapshots
mount -o "$BTRFS_OPTS,subvol=@var_log" /dev/nvme0n1p3 /mnt/var/log
mount -o "$BTRFS_OPTS,subvol=@pkg" /dev/nvme0n1p3 /mnt/var/cache/pacman/pkg

# Mount EFI
mount /dev/nvme0n1p1 /mnt/boot/efi

# Mount secondary HDD
mount /dev/sda1 /mnt/mnt/wdblue

# Verify mounts
findmnt -R /mnt

reflector \
  --country Indonesia,Singapore \
  --protocol https \
  --latest 10 \
  --download-timeout 10 \
  --sort rate \
  --save /etc/pacman.d/mirrorlist

pacstrap -K /mnt \
  base \
  base-devel \
  linux \
  linux-headers \
  linux-zen \
  linux-zen-headers \
  linux-firmware \
  amd-ucode \
  btrfs-progs \
  neovim \
  git \
  nano \
  networkmanager \
  sudo \
  reflector

genfstab -U /mnt >> /mnt/etc/fstab

# Inspect generated fstab
nano /mnt/etc/fstab

# Enter installed system
arch-chroot /mnt

# Set timezone
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc

# Edit locale configuration
nano /etc/locale.gen

(Uncomment #en_US.UTF-8 UTF-8)

# Generate locale
locale-gen

# Set system language
nano /etc/locale.conf
(LANG=en_US.UTF-8)

# Set Hostname
nano /etc/hostname
(archlinux)

# Create User Name
useradd -m -G wheel -s /bin/bash famouzak

# Set user password
passwd famouzak

# Set root password
passwd

# EDITOR=nano visudo
(uncomment # %wheel ALL=(ALL:ALL) ALL)

# Pacman Configuration (Enable Multilib, Color, IloveCandy)

nano /etc/pacman.conf (Uncomment Below or Add)

ParallelDownloads = 10
#Color
ILoveCandy
#[multilib]
#Include = /etc/pacman.d/mirrorlist

pacman -Syu (Update Repository)

# Install Package ( Mesa, Audio, KDE Plasma)

pacman -S --needed \
  git networkmanager pacman-contrib \
  mesa lib32-mesa \
  vulkan-radeon lib32-vulkan-radeon \
  libva-mesa-driver lib32-libva-mesa-driver \
  lib32-vulkan-icd-loader vulkan-icd-loader vulkan-tools \
  pipewire pipewire-pulse pipewire-alsa pipewire-jack \
  wireplumber pavucontrol easyeffects \
  lsp-plugins-lv2 calf alsa-utils \
  alacritty plasma-meta \
  gwenview okular konsole dolphin kdeconnect kdenlive \
  sddm wayland egl-wayland xdg-user-dirs \
  qt6-base qt6-declarative qt6-svg

# Clone SDDM Elegant
git clone https://github.com/sniper1720/elegant-sddm-archlinux-theme.git /tmp/elegant-sddm

mkdir -p /usr/share/sddm/themes/

cp -r /tmp/elegant-sddm/elegant-archlinux /usr/share/sddm/themes/

rm -rf /tmp/elegant-sddm

# Make Config File
mkdir -p /etc/sddm.conf.d
nano /etc/sddm.conf.d/theme.conf 
([Theme]
Current=elegant-archlinux)

# Install GRUB
pacman -S --needed \
  grub efibootmgr grub-btrfs snapper snap-pac \
  inotify-tools os-prober

# Snapper Configuration
snapper -c root create-config /

# Set Permission
chmod 750 /.snapshots
chown :wheel /.snapshots
snapper -c root set-config ALLOW_USERS=famouzak SYNC_USER=yes

# Aktivate Snapper
systemctl enable snapper-timeline.timer
systemctl enable snapper-cleanup.timer

# Customise GRUB (Use Hyprfluent Theme)
mkdir -p /boot/grub/themes

git clone --depth=1 \
  https://github.com/Coopydood/HyperFluent-GRUB-Theme.git \
  /tmp/hyperfluent-grub

cp -r /tmp/hyperfluent-grub/arch \
  /boot/grub/themes/HyperFluent

rm -rf /tmp/hyperfluent-grub

# Configure GRUB
nano /etc/default/grub

GRUB_TIMEOUT=3
GRUB_GFXMODE=1920x1080x32,auto
GRUB_DISABLE_SUBMENU=y
GRUB_TERMINAL_OUTPUT="gfxterm"
GRUB_THEME="/boot/grub/themes/HyperFluent/theme.txt"
GRUB_TOP_LEVEL="/boot/vmlinuz-linux-zen"

# Configure mkinitcpio
nano /etc/mkinitcpio.conf
MODULES=(btrfs amdgpu)

mkinitcpio -P

# INSTALL GRUB
grub-install \
  --target=x86_64-efi \
  --efi-directory=/boot/efi \
  --bootloader-id=GRUB \
  --removable

grub-mkconfig -o /boot/grub/grub.cfg

# Activate Services

systemctl enable NetworkManager
systemctl enable sddm
systemctl enable grub-btrfsd

exit
umount -R /mnt
reboot

# POST INSTALL SETUP (AFTER LOGIN TO KDE PLASMA)


# Install YAY (AUR HELPER)
git clone https://aur.archlinux.org/yay.git

cd yay

makepkg -si

cd ..

# Install Gaming & Multimedia

sudo pacman -S --needed \
  steam wine-staging winetricks \
  gamemode lib32-gamemode gamescope \
  mangohud lib32-mangohud goverlay lact \
  obs-studio obs-studio-plugin-browser \
  ffmpeg vlc mpv gstreamer \
  gnutls lib32-gnutls giflib \
  v4l2loopback-dkms v4l2loopback-utils \
  flatpak

# Add User to Acces Gamemode
sudo usermod -aG gamemode famouzak

# Verify after logout then login
gamemoded -t

# Activate LACT
sudo systemctl enable --now lactd

# Verifikasi dkms status
dkms status

# INSTALL AUR PACKAGE
yay -S --needed \
  brave-bin \
  darkly-bin \
  protonplus \
  lib32-giflib

# flatpak configuration
flatpak remote-add --if-not-exists \
  flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Install Sober (Roblox) flatpak
flatpak install flathub org.vinegarhq.Sober
