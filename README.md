# Installing Gentoo on my Lenovo Legion 5 Pro

My personal notes, following the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64)

Laptop specs
- **CPU:** AMD Ryzen 5 5600H (6 cores, 12 threads)
- **RAM:** 16GB
- **Storage:** 1TB NVMe SSD
- **Graphics:** NVIDIA GeForce RTX 3060 Laptop GPU
- **Network:** MediaTek Wi-Fi 6 MT7921 Wireless LAN Card

## Preparing the disks

### Overview

We are using GPT + UEFI + Btrfs + GRUB2 with the following scheme
| Partition      | Filesystem | Size             | Mountpoint | Description
|----------------|------------|------------------|------------|-----------------------------------------
| /dev/nvme0n1p1 | FAT32      | 512 MiB          | /efi       | EFI system partition
| /dev/nvme0n1p2 | swap       | 12 GiB           | none       | Generous amount of swap (core count * 2)
| /dev/nvme0n1p3 | Btrfs      | Rest of the disk | /          | Root filesystem

* [What size should the ESP be?](https://forums.gentoo.org/viewtopic-p-8534167.html?sid=3c6cbac0f4df783e368a749df8bfd2f1#8534167)
* [Official UEFI specification](https://uefi.org/sites/default/files/resources/UEFI%202_5.pdf)

Btrfs volumes and subvolumes
| Partition      | (Sub)volume name | Mountpoint          | Description
|----------------|------------------|---------------------|-----------------------------
| /dev/nvme0n1p3 | rootfs           | none                | Btrfs root filesystem volume
| /dev/nvme0n1p3 | @gentoo          | /                   | Root subvolume
| /dev/nvme0n1p3 | @home            | /home               | Personal files
| /dev/nvme0n1p3 | @boot            | /boot               | Kernels and files for GRUB2
| /dev/nvme0n1p3 | @snapshots       | /snapshots          | Btrfs snapshots
| /dev/nvme0n1p3 | @steam           | /home/temp04/.steam | Steam games and files
| /dev/nvme0n1p3 | @virt            | /virt               | Virtual machines

---

### Make partitions

Using `parted`

```console
parted -a optimal /dev/nvme0n1
unit mib
mkpart esp 1 513
mkpart swap 513 12801
mkpart rootfs 12801 -1
set 1 boot on
print
```

### Create filesystems

Format partitions
```console
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.btrfs /dev/nvme0n1p3
```
Make swap
```console
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```
### Create subvolumes

Mount the root partition
```console
mkdir -p /mnt/gentoo
mount /dev/nvme0n1p3 /mnt/gentoo
```

Btrfs subvolumes
```console
cd /mnt/gentoo
btrfs subvol create @gentoo
btrfs subvol create @home
btrfs subvol create @boot
btrfs subvol create @snapshots
btrfs subvol create @steam
btrfs subvol create @virt
```
