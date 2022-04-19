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

We are using GPT/UEFI + Btrfs + GRUB2 with the following scheme
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

* [Btrfs scheme reference](https://old.reddit.com/r/btrfs/comments/9us4hr/what_is_your_btrfs_partitionsubvolumes_scheme/e973hym/)

---

### Make partitions

Using `parted`

```bash
parted -a optimal /dev/nvme0n1
unit mib
mklabel gpt
mkpart esp 1 513
mkpart swap 513 12801
mkpart rootfs 12801 -1
set 1 boot on
print
quit
```

* [Very good ressource for Gentoo on ZFS](https://wiki.gentoo.org/wiki/User:Fearedbliss/Installing_Gentoo_Linux_On_ZFS)

### Make filesystems

Format partitions
```bash
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.btrfs /dev/nvme0n1p3
```
Make swap
```bash
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```
### Btrfs subvolumes

Mount the root partition
```bash
mkdir -p /mnt/gentoo
mount /dev/nvme0n1p3 /mnt/gentoo
```

Create subvolumes
```bash
cd /mnt/gentoo
btrfs subvol create @gentoo
btrfs subvol create @home
btrfs subvol create @boot
btrfs subvol create @snapshots
btrfs subvol create @steam
btrfs subvol create @virt
```

Mount subvolumes
```bash
# first, unmount
cd ..
umount -l gentoo

# then, remount
mount -o subvol=@gentoo /dev/nvme0n1p3 /mnt/gentoo

# make directories
cd gentoo
mkdir home boot snapshots virt efi

# mount remaining subvolumes
mount -o subvol=@home /dev/nvme0n1p3 /mnt/gentoo/home
mount -o subvol=@boot /dev/nvme0n1p3 /mnt/gentoo/boot
mount -o subvol=@snapshots /dev/nvme0n1p3 /mnt/gentoo/snapshots
mount -o subvol=@virt /dev/nvme0n1p3 /mnt/gentoo/virt

# mount esp
mount /dev/nvme0n1p1 /mnt/gentoo/efi
```

**TIP:** Show mounted subvolumes with `findmnt -nt btrfs`
* [Example setup from a Gentoo user](https://gist.github.com/renich/90e0a5bed8c7c0de40d40ac9ccac6dfd)


## Prepararing the chroot environement

### Installing stage3

Grab the latest stage3 tarball [from here](https://www.gentoo.org/downloads/)
```bash
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20220417T171236Z/stage3-amd64-desktop-systemd-20220417T171236Z.tar.xz

# extract archive
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

### Configure compile options

We keep it basic
```bash
nano ect/portage/make.conf

# safe cflags
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"

# a bit risky maybe
MAKEOPTS="-j8"

# sensible defaults
# 5.1 is 85% load for 6 cores (6*0.85)
EMERGE_DEFAULT_OPS="--ask --verbose --quiet --ask-enter-invalid --load-average 5.1"

# graphics
VIDEO_CARDS="nvidia"
```

### More configuration

Portage
```bash
mkdir -p etc/portage/repos.conf
cp usr/share/portage/config/repos.conf etc/portage/repos.conf/gentoo.conf
cat etc/portage/repos.conf/gentoo.conf
```

DNS
```bash
cp --dereference /etc/resolv.conf etc/
cat etc/resolv.conf
```

### Mounting the necessary filesystems

Here we go
```bash
mount --types proc /proc proc
mount --rbind /sys sys
mount --rbind /dev dev
mount --bind /run run

# systemd
mount --make-rslave sys
mount --make-rslave dev
mount --make-slave run
```

### Chrooting

Entering the new environement
```bash
chroot . /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

## Installing the Gentoo base system

### Emerge

First, fetch the lastest ebuild repository snapshot
```bash
emerge-webrsync
```

Then, read the news
```bash
eselect news list
eselect news read
```

Pick the right profile
```bash
eselect profile list

# we kde plasma
eselect profile set 9
```

Now, go to sleep :zzz:
```bash
# emerge -uDN @world
emerge --ask --verbose --update --deep --newuse @world
```
Install full KDE desktop (hopefully)
```bash
# i used makeopts -j2 on this
# probably bad
emerge kde-plasma/plasma-meta --jobs 6
```

### Kernel configuration

Edit this file to unmask `linux-firmware`
```bash
nano /etc/portage/package.license/kernel

# append
sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE
```

Necessary packages and utilities
```bash
# important packages
emerge linux-firmware gentoo-kernel-bin nvidia-drivers

# qol packages
emerge gentoolkit genlop bash-completion btrfs-progs konsole firefox-bin dev-vcs/git
```

* [Gentoo Wiki page for dispatch-conf](https://wiki.gentoo.org/wiki/Dispatch-conf)

### Systemd services

Enable these services for a painless reboot
```bash
systemctl enable sddm.service
systemctl enable NetworkManager
```

## Creating the fstab file

Current file
```bash
# <fs>                                          <mnt>   <type>  <opts>                  <dump/pass>
UUID="3C5E-6730"                                /efi    vfat    noauto                  0 2
UUID="f3120bf4-4283-4d68-9048-0aae99195fd7"     none    swap    sw                      0 0
UUID="c3830926-0109-42c1-a9cd-45f41f0d602e"     /       btrfs   defaults,subvol=@gentoo 0 1
UUID="c3830926-0109-42c1-a9cd-45f41f0d602e"     /boot   btrfs   defaults,subvol=@boot   0 2
UUID="c3830926-0109-42c1-a9cd-45f41f0d602e"     /home   btrfs   defaults,subvol=@home   0 2
UUID="c3830926-0109-42c1-a9cd-45f41f0d602e"     /virt   btrfs   defaults,subvol=@virt   0 2
```

* [Btrfs sysadmin guide on kernel.org](https://btrfs.wiki.kernel.org/index.php/SysadminGuide)

## Configuring the bootloader (GRUB2)

Install GRUB
```bash
emerge -DN grub
grub-install --target=x86_64-efi --efi-directory=/efi
```

Configure it
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Finalizing steps

Add user with sudo privileges
```bash
# install sudo
emerge sudo

# make admin user
useradd -m -G users,wheel,audio -s /bin/bash temp04
passwd temp04
```

If you can't use weak passwords, edit the following file
```bash
nano /etc/security/passwdqc.conf

# don't enforce password policy
enforce=none
```

Edit sudoers
```bash
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```

## Rebooting the system

Exit and pray 🙏
```bash
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```
