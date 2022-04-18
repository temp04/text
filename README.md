# Installing Gentoo on my Lenovo Legion 5 Pro

My personal notes, following the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64).

Laptop specs:

- AMD Ryzen 5 5600H (6 cores, 12 threads)
- 16GB of RAM
- 1TB NVMe SSD
- NVIDIA GeForce RTX 3060 Laptop GPU
- MediaTek Wi-Fi 6 MT7921 Wireless LAN Card

## Disk partitioning

We are using GPT + UEFI + Btrfs + GRUB2 with the following scheme:

| Partition        | Filesystem | Size             | Mountpoint | Description
|------------------|------------|------------------|------------|-------------------------------------------
| `/dev/nvme0n1p1` | FAT32      | 512M             | `none`     | EFI system partition
| `/dev/nvme0n1p2` | swap       | 12G              | `none`     | Generous amount of swap (core count * 2)
| `/dev/nvme0n1p3` | Btrfs      | Rest of the disk | `/`        | Root filesystem

* [What size should the ESP be?](https://forums.gentoo.org/viewtopic-p-8534167.html?sid=3c6cbac0f4df783e368a749df8bfd2f1#8534167)

Btrfs volumes and subvolumes:

| Partition        | (Sub)volume name | Mountpoint            | Description
|------------------|------------------|-----------------------|-----------------------------
| `/dev/nvme0n1p3` | rootfs           | `none`                | Primary volume
| `/dev/nvme0n1p3` | @gentoo          | `/`                   | Root subvolume
| `/dev/nvme0n1p3` | @home            | `/home`               | Personal fiiles
| `/dev/nvme0n1p3` | @boot            | `none`                | Kernels and files for GRUB2
| `/dev/nvme0n1p3` | @snapshots       | `none`                | Keep snapshots unmounted
| `/dev/nvme0n1p3` | @steam           | `/home/temp04/.steam` | Steam games and files
