# Introduction
Welcome to my guide on how to install a bare bones Arch linux server setup. It is assumed that the server uses:
* Wired connections (e.g. ethernet)
* UEFI
* Intel CPUs (an AMD section will be added in a future revision)

By following this guide, the server will be setup with grub, a LUKS encrypted disk with LVM on top, a btrfs filesystem and swap. If a desktop orientated install is desired, post installation, the general recommendations section of the venerable Arch Wiki (https://wiki.archlinux.org/index.php/General_recommendations) should be consulted.

Note: this guide was written after the base group was replaced by a metapackage of the same name (https://www.archlinux.org/news/base-group-replaced-by-mandatory-base-package-manual-intervention-required/).

# Pre installation
## Operating system image
Download, verify, write and boot the image.

## Verify the boot mode
Check if the system was booted in UEFI mode. To verify this, list the efivars directory:
```
ls /sys/firmware/efi/efivars
```
If the directory does not exist then the system may have been booted in BIOS or CSM mode. Check your motherboard's manual for details. This guide is written for a UEFI system.

## Connect to the internet
To set up a network connection, go through the following steps:
* Ensure your network interface is listed and enabled, for example with ip-link
* Connect to the network. Plug in the ethernet cable.
* Configure your network connection:
    * Static IP
    * Dynamic IP address - use DHCP `dhcpcd`

## Update the system clock
Use timedatectl to ensure the system clock is accurate:
```
timedatectl set-ntp true
```
To check the service status, use `timedatectl status`.

## Partitioning the disks
Open `gdisk` to create partitions for EFI and /boot.
Start by creating a new GPT partition table protected by a fake MBR.
```
gdisk /dev/sda
o (Create a new empty GUID partition table (GPT))
Proceed? Y
```
Create partitions:
```
n (Add a new partition)
Partition number 1
First sector 2048 (default)
Last sector +512M
Hex code EF00
```
```
n (Add a new partition)
Partition number 2
First sector 2099200 (default)
Last sector (press Enter to use remaining disk)
Hex code 8E00
```
It should look something like this:
```
Number Start (sector) End (sector) Size Code Name
1 2048 1050623 512.0 MiB EF00 EFI System
2 1050624 500118158 238.0 GiB 8E00 Linux LVM
```
If the partition layout looks okay, save and quit:
```
w
Y
```
## Setting up encryption
Setup the luks/dm-crypt encryption container:
```
cryptsetup luksFormat /dev/sda2
Are you sure? YES
Enter passphrase (twice)
```
Unlock and open the container. The device mapper will name the decrypted block device as *encrypted* in this example:
```
cryptsetup open /dev/sda2 encrypted
```
Create the LVM physical volume:
```
pvcreate /dev/mapper/encrypted
```
Create the LVM volume group. The volume group will be *vg_encrypted* in this example:
```
vgcreate vg_encrypted /dev/mapper/encrypted
```
Create the LVM logical volumes - an 8GB volume for swap and a volume for root which takes the remaining space. The volumes will be labelled *swap* and *root* respectively:
```
lvcreate -L 8G vg_encrypted -n lv_swap
lvcreate -l 100%FREE vg_encrypted -n lv_root
```

## Formatting
Format the first partition into a fat32 boot and efi file system:
```
mkfs.vfat -F32 /dev/sda1
```
Format the larger LVM logical volume into a BTRFS root file system - which is labelled *root* in this example:
```
mkfs.btrfs -L root /dev/mapper/vg_encrypted-lv_root
```
Format the remaining LVM logical volume into a swap partition - which is labelled *swap* in this example:
```
mkswap -L swap /dev/mapper/vg_encrypted-lv_swap
```
Activate the swap partition:
```
swapon /dev/mapper/vg_encrypted-lv_swap
```

# Mounting
Mount the `/` and `/boot` volumes at `/mnt` and `/mnt/boot` respectively:
```
mount /dev/mapper/vg_encrypted-lv_root /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

# Installation
## Select mirrors
Packages to be installed must be downloaded from mirror servers, which are defined in `/etc/pacman.d/mirrorlist`. On the live system, all mirrors are enabled, and sorted by their synchronization status and speed at the time the installation image was created.

The higher a mirror is placed in the list, the more priority it is given when downloading a package. You may want to edit the file accordingly, and move the geographically closest mirrors to the top of the list, although other criteria should be taken into account.

This file will later be copied to the new system by pacstrap, so it is worth getting right. 

## Install essential packages
As the ***base*** package no longer includes a lot of basic packages for common utilities, other packages such as:
* userspace utilities for the management of file systems used
* utilities for accessing LVM and encrypted partitions
* specific firmware for other devices not included in `linux-firmware`
* software necessary for networking
* text editors
* packages for accessing documentation in man and info pages `man-db man-pages texinfo`

Install the packages:
```
pacstrap /mnt base base-devel linux linux-firmware e2fsprogs btrfs-progs cryptsetup lvm2 grub efibootmgr intel-ucode sudo bash-completion vi vim less which p7zip usbutils gptfdisk iproute2 dhcpcd nftables man-db man-pages
```

# Configure the system
## Fstab
Generate a fstab file using PARTUUID as the identifier. PARTUUID is a unique identifier generated for all GPT partitions and is more persistent than UUID.
```
genfstab -p -t PARTUUID /mnt > /mnt/etc/fstab
```

## Chroot
Change root into the new system:
```
arch-chroot /mnt
```

## Time zone
Set the time zone:
```
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```
Generate `/etc/adjtime`:
```
hwclock --systohc
```
This command assumes the hardware clock is set to UTC. This can be checked by running:
```
timedatectl | grep local
```
If the output is not:
```
RTC in local TZ: no
```
Then the hardware clock can be set to UTC using:
```
timedatectl set-local-rtc 0
```

## Localisation
Uncomment `en_US.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`, and generate them with: 
```
locale-gen
```
Create `/etc/locale.conf`, and set the ***LANG*** variable accordingly: 
```
LANG=en_AU.UTF-8
```
If you set the keyboard layout, make the changes persistent in `/etc/vconsole.conf`:
```
KEYMAP=de-latin1
```

## Network configuration
Create the hostname (`/etc/hostname`) file:
```
myhostname
```

Add matching entries to the hosts (`/etc/hosts`) file:
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```
If the system has a permanent IP address, it should be used instead of `127.0.1.1`.

## Root password
Change the root password so login is possible after reboot:
```
passwd
```

## Initramfs
Modify ***MODULES*** and ***HOOKS*** in `/etc/mkinitcpio.conf` to match the lines below:
```
MODULES=(crc32_generic crc32c-intel)
HOOKS=(base udev autodetect modconf block keyboard encrypt lvm2 btrfs resume filesystems fsck)
```
Note that the modules and scripts in ***HOOKS*** are position sensitive. They are added in order from left to right.

Regenerate the initramfs:
```
mkinitcpio -P
```

## Bootloader
Choose a bootloader identifier. ***ArchLinux*** is the identifier for this example. Install the GRUB EFI application:
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ArchLinux
```

Identify the ***PARTUUID*** of the encrypted partition which should be `/dev/sda2`. The PARTUUIDs can be listed using:
```
blkid -s PARTUUID -o value /dev/sda2
```

Modify ***GRUB_CMDLINE_LINUX*** in `/etc/default/grub` to match the line below replacing the `{VALUES}` to match the actual values:
* **{PARTUUID}** is the PARTUUID of the encrypted partition.
* **{ENCRYPTED_PARTITION}** is the name unlocked encrypted block device. It is `encrypted` in this guide.
* **{VG}** is the name of the LVM volume group. It is `vg_encrypted` in this guide.
* **{LV_ROOT}** is the name of the LVM logical volume that contains the root partition. It is `lv_root` in this guide.
* **{LV_SWAP}** is the name of the LVM logical volume that contains the swap partition. It is `lv_swap` in this guide.
```
GRUB_CMDLINE_LINUX="cryptdevice=PARTUUID={PARTUUID}:{ENCRYPTED_PARTITION}:allow-discards root=/dev/mapper/{VG}-{LV_ROOT} rw resume=/dev/mapper/{VG}-{LV_SWAP}"
```

Generate the grub config (`/boot/grub/grub.cfg`):
```
grub-mkconfig -o /boot/grub/grub.cfg
```

## Reboot
Exit the chroot by typing `exit` or pressing `ctrl+d`.

Unmount all manually mounted partitions:
```
cd /
umount -R /mnt
```
Restart using `reboot`.

# Post installation
Head to the Arch Wiki for further setup specific customisations.

# Sources
* https://wiki.archlinux.org/
* https://fogelholk.io/installing-arch-with-lvm-on-luks-and-btrfs/
