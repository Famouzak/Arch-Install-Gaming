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
(