# Installing Gentoo on my Lenovo Legion 5 Pro

My personal notes, following the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64).

## Disk partitioning

We are using GPT + UEFI + Btrfs with the following scheme:

| Partition        | Filesystem | Size             | Mountpoint | Description
|------------------|------------|------------------|------------|-------------------------------------------
| `/dev/nvme0n1p1` | FAT32      | 512M             | `none`     | EFI system partition
| `/dev/nvme0n1p2` | swap       | 12G              | `none`     | Generous amount of swap (core count * 2)
| `/dev/nvme0n1p3` | Btrfs      | 100G             | `/`        | Root filesystem
| `/dev/nvme0n1p4` | XFS        | Rest of the disk | `/home`    | Home partition

* [What size should the ESP be?](https://forums.gentoo.org/viewtopic-p-8534167.html?sid=3c6cbac0f4df783e368a749df8bfd2f1#8534167)
* [rEFInd on the Gentoo Wiki](https://wiki.gentoo.org/wiki/Refind)
