# Perbesar ukuran font terminal agar jelas saat direkam
setfont ter-132b
ping -c 3 archlinux.org

# 1. Sinkronisasi waktu jaringan & optimasi mirrorlist
timedatectl status
timedatectl set-ntp true

# Cek daftar disk
lsblk -f


# 2. Partisi Disk NVMe (/dev/nvme0n1) menggunakan skema GPT
sgdisk --zap-all /dev/nvme0n1
parted -s /dev/nvme0n1 mklabel gpt
parted -s /dev/nvme0n1 mkpart ESP fat32 1MiB 1024MiB
parted -s /dev/nvme0n1 set 1 esp on
parted -s /dev/nvme0n1 mkpart primary linux-swap 1024MiB 5120MiB
parted -s /dev/nvme0n1 mkpart primary btrfs 5120MiB 100%

# 3. Format Partisi
mkfs.fat -F32 -n "EFI" /dev/nvme0n1p1
mkswap -L "ARCH_SWAP" /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkfs.btrfs -f -L "ARCH_LINUX" /dev/nvme0n1p3

# 4. Membuat Subvolume Btrfs
mount /dev/nvme0n1p3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@pkg
umount /mnt

# 5. Mounting Subvolume (Pastikan TANPA SPASI setelah koma)
BTRFS_OPTS="noatime,compress=zstd:1,ssd,discard=async,space_cache=v2"

mount -o $BTRFS_OPTS,subvol=@ /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot/efi,home,.snapshots,var/log,var/cache/pacman/pkg,mnt/wdblue}

mount -o $BTRFS_OPTS,subvol=@home /dev/nvme0n1p3 /mnt/home

mount -o $BTRFS_OPTS,subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots

mount -o $BTRFS_OPTS,subvol=@var_log /dev/nvme0n1p3 /mnt/var/log

mount -o $BTRFS_OPTS,subvol=@pkg /dev/nvme0n1p3 /mnt/var/cache/pacman/pkg

mount /dev/nvme0n1p1 /mnt/boot/efi

# 6. Mount Secondary Storage (HDD WD Blue /dev/sda1)
mount /dev/sda1 /mnt/mnt/wdblue

# Reflector Mirrorlist (Indonesia & Singapura)
reflector --country Indonesia,Singapore --protocol https --latest 15 --download-timeout 5 --sort rate --save /etc/pacman.d/mirrorlist

# 7. Install paket inti dan kernel multi-opsi + AMD Microcode
pacstrap -K /mnt base base-devel linux linux-headers linux-lts linux-lts-headers linux-zen linux-zen-headers linux-firmware amd-ucode btrfs-progs neovim git nano networkmanager sudo reflector

# 8. Generate FSTAB berbasis UUID
genfstab -U /mnt >> /mnt/etc/fstab

# Masuk ke sistem baru
arch-chroot /mnt

# 1. Zona Waktu & Lokalisasi
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc

# Buka file locale.gen, lalu cari dan hapus tanda pagar (#) pada baris: en_US.UTF-8 UTF-8
nano /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# 2. Hostname
echo "archlinux" > /etc/hostname

# Buat file /etc/hosts secara manual
nano /etc/hosts


# 3. Pembuatan User & Akses Sudo
useradd -m -G wheel,video,audio,storage,optical,input -s /bin/bash famouzak
passwd famouzak
passwd # Atur password root

# Gunakan visudo (cara paling aman dan profesional untuk edit sudoers)
visudo
# (Cari baris: # %wheel ALL=(ALL:ALL) ALL -> Hapus tanda pagar # di depannya)

# 4. Optimasi Pacman & Aktifkan Multilib (Steam/32-bit)
nano /etc/pacman.conf
# (Aksi di nano: 
# - Ubah #ParallelDownloads = 5 menjadi ParallelDownloads = 10
# - Hilangkan tanda # pada Color dan aktifkan ILoveCandy di bawahnya
# - Hilangkan tanda # pada blok [multilib] dan baris Include di bawahnya)

pacman -Sy

# 5. Install Driver AMD Radeon, Audio PipeWire, & KDE Plasma 6
pacman -S --noconfirm --needed \
  git networkmanager pacman-contrib \
  mesa lib32-mesa \
  vulkan-radeon lib32-vulkan-radeon \
  libva-mesa-driver lib32-libva-mesa-driver \
  lib32-vulkan-icd-loader vulkan-icd-loader vulkan-tools \
  pipewire pipewire-pulse pipewire-alsa pipewire-jack wireplumber pavucontrol easyeffects lsp-plugins-lv2 calf alsa-utils \
  alacritty plasma-meta gwenview okular konsole dolphin kdeconnect kdenlive sddm wayland egl-wayland xdg-user-dirs \
  qt6-base qt6-declarative qt6-svg

# 6. Setup SDDM Login Manager Tema Elegant
git clone https://github.com/sniper1720/elegant-sddm-archlinux-theme.git /tmp/elegant-sddm
mkdir -p /usr/share/sddm/themes/
cp -r /tmp/elegant-sddm/elegant-archlinux /usr/share/sddm/themes/
rm -rf /tmp/elegant-sddm

mkdir -p /etc/sddm.conf.d
nano /etc/sddm.conf.d/theme.conf
# (Isi dengan:
# [Theme]
# Current=elegant-archlinux
# )

# 7. Install GRUB & Snapshot Tools
pacman -S --noconfirm --needed grub efibootmgr grub-btrfs snapper snap-pac inotify-tools os-prober

# Deploy Tema GRUB HyperFluent
mkdir -p /boot/grub/themes
git clone --depth=1 https://github.com/Coopydood/HyperFluent-GRUB-Theme.git /tmp/hyperfluent-grub
cp -r /tmp/hyperfluent-grub/arch /boot/grub/themes/HyperFluent
rm -rf /tmp/hyperfluent-grub

# Kustomisasi GRUB secara manual
nano /etc/default/grub
# (Ubah atau tambahkan baris berikut:
# GRUB_TIMEOUT=3
# GRUB_GFXMODE=1920x1080x32,auto
# GRUB_DISABLE_SUBMENU=y
# GRUB_TERMINAL_OUTPUT="gfxterm"
# GRUB_THEME="/boot/grub/themes/HyperFluent/theme.txt"
# GRUB_TOP_LEVEL="/boot/vmlinuz-linux-zen"
# )

# 8. Konfigurasi Early KMS untuk Btrfs & AMDGPU di mkinitcpio
nano /etc/mkinitcpio.conf
# (Cari baris MODULES=() dan ubah menjadi: MODULES=(btrfs amdgpu))
mkinitcpio -P

# 9. Install GRUB & Generate Config (Fallback Removable)
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable
grub-mkconfig -o /boot/grub/grub.cfg

# 10. Aktifkan Services Penting
systemctl enable NetworkManager
systemctl enable sddm
systemctl enable grub-btrfsd

# Keluar dari chroot & Reboot
exit
umount -R /mnt
reboot

# 1. Konfigurasi Snapper
sudo umount /.snapshots || true
sudo rm -rf /.snapshots
sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir -p /.snapshots
sudo mount -a
sudo chmod 750 /.snapshots
sudo chown :wheel /.snapshots
sudo snapper -c root set-config ALLOW_USERS=famouzak SYNC_USER=yes
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer

# 2. Install YAY (AUR Helper)
git clone https://aur.archlinux.org/yay.git /tmp/yay
cd /tmp/yay && makepkg -si --noconfirm
cd ~ && rm -rf /tmp/yay

# 3. Install Gaming Stack (Official Repos)
sudo pacman -S --noconfirm --needed \
  steam wine-staging winetricks \
  gamemode lib32-gamemode gamescope \
  mangohud lib32-mangohud goverlay lact \
  obs-studio obs-studio-plugin-browser ffmpeg vlc mpv gstreamer \
  gnutls lib32-gnutls giflib \
  v4l2loopback-dkms v4l2loopback-utils flatpak

# 4. Install AUR Packages
yay -S --noconfirm --needed \
  brave-bin \
  darkly-bin \
  protonplus \
  lib32-giflib

# 5. Install Flatpak Apps (Sober)
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.vinegarhq.Sober
