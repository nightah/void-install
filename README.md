# void-install
[void-install](https://github.com/nightah/void-install) is a set of instructions to install Void Linux on a ZFS root.

## Voidlive ISO
- Grab an ISO that has the ZFS dkms embedded, in this case I will be using [hrmpf-x86_64-4.14.10_1-20180102.iso](https://github.com/chneukirchen/hrmpf/releases/download/v20180102/hrmpf-x86_64-4.14.10_1-20180102.iso) you can grab the [latest iso here](https://github.com/chneukirchen/hrmpf/releases).
- Make a bootable usb key.
- Substitute for sdX as appropriate:
```
dd bs=512K if=./hrmpf-x86_64-4.14.10_1-20180102.iso of=/dev/sdX; sync
```
## Partitioning the destination drive(s)
Using GRUB on a BIOS (or UEFI machine in legacy boot mode) machine but using a GPT partition table:
```
Part     Size   Type
----     ----   -------------------------
   1     XXXG   Solaris Root (bf00)
   2       2M   BIOS boot partition (ef02)
```
**For a single drive zpool:**
```
# gdisk /dev/sda
GPT fdisk (gdisk) version 1.0.1

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help): n
Partition number (1-128, default 1): 2
First sector (34-1465149134, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-1465149134, default = 1465149134) or {+-}size{KMGTP}: +2M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): ef02
Changed type of partition to 'BIOS boot partition'

Command (? for help): n
Partition number (1-128, default 1):
First sector (34-1465149134, default = 6144) or {+-}size{KMGTP}:
Last sector (268288-1465149134, default = 1465149134) or {+-}size{KMGTP}:
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): bf00
Changed type of partition to 'Solaris root'

Command (? for help): p
Disk /dev/sda: 1465149168 sectors, 698.6 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): AC45081A-160B-4DFC-BA93-58953FD6D80D
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 1465149134
Partitions will be aligned on 2048-sector boundaries
Total free space is 526302 sectors (257.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1          268288      1464886990   698.4 GiB   BF00  Solaris root
   2            2048            6143   2.0 MiB     EF02  BIOS boot partition

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
The operation has completed successfully.
```
**For a mirrored zpool:**

When the above step has been completed on the first drive of your mirror, clone the table onto our second device, sdb.
```
# sgdisk -b sda-part.table /dev/sda
The operation has completed successfully.
# sgdisk -l sda-part.table /dev/sdb
Creating new GPT entries.
The operation has completed successfully.
```
Being an exact copy, we now have two drives with identical GUID’s.

Let’s give sdb a new GUID and confirm that sda and sdb are indeed unique:
```
# sgdisk -G /dev/sdb
The operation has completed successfully.
# sgdisk -p /dev/sdb | grep -i guid
Disk identifier (GUID): 0FCC8550-ED78-407D-BEEA-D43FB73C7257
# sgdisk -p /dev/sda | grep -i guid
Disk identifier (GUID): 169BB909-2184-4BFF-9A41-107740B0F043
```
## Setup the ZFS filesystem
- Boot using the previously created bootable usb key.
- Build and load ZFS modules:
```
# xbps-reconfigure -a
# modprobe zfs
```
ZFS pools use hostid as a precaution to keep pools from inadvertenly being imported by the wrong system.

When running ZFS on root, the machine's hostid will not be available at the time of mounting the root filesystem.
Voidlinux, being a true FOSS project, has no need to set a hostid and falls back to a default of 00000000.

To remedy this we will create `/etc/hostid` basing it on the MAC address of the first ethernet device:
```
# ip link sh | grep ether
    link/ether f4:6d:04:1e:58:12 brd ff:ff:ff:ff:ff:ff
```
- Create `/etc/hostid` ensuring to strip out the colons:
```
# echo "f46d041e5812" > /etc/hostid
# hostid
64363466
```
## Create the ZFS pool:
**For a single drive zpool:**
```
# zpool create -f -o ashift=12 -m none nerv /dev/disk/by-id/id-to-partition-part1
```
**For a mirrored zpool:**
```
# zpool create -f -o ashift=12 -m none nerv mirror /dev/disk/by-id/id-to-partition-part1 /dev/disk/by-id/id-to-partition-part1
```
## ZFS tuning
```
zfs set relatime=on nerv
zfs set compression=on nerv
```
## Create datasets
```
# zfs create -o mountpoint=none nerv/ROOT
# zfs create -o mountpoint=/ nerv/ROOT/void
```
## Configure the root filesystem
- Export the pool:
```
# zpool export nerv
```
- Re-import the pool:
```
# zpool import -d /dev/disk/by-id -R /mnt nerv
```
If this command fails and you are asked to import your pool via its numeric ID, run `zpool import` to find out the ID of your pool then use a command such as: `zpool import 9876543212345678910 -R /mnt nerv`
## Install Void Linux
- Install the base system:
```
# xbps-install -S -R https://repo.voidlinux.eu/current -r /mnt base-system grub intel-ucode zfs
```
- Copy the hostid file we created above:
```
# cp /etc/hostid /mnt/etc
```
- Mount and chroot:
```
# mount -t proc proc /mnt/proc
# mount -t sysfs sys /mnt/sys
# mount -B /dev /mnt/dev
# mount -t devpts pts /mnt/dev/pts
# pwd
/root
# env -i HOME=/root TERM=$TERM chroot /mnt bash -l
# pwd
/
```
- Set the keyboard layout:
```
# echo "KEYMAP=\"us\"" >> /etc/rc.conf
```
- Set the time zone:
```
# echo "TIMEZONE=\"Australia/Melbourne\"" >> /etc/rc.conf
```
- Set the hardware clock:
```
# echo "HARDWARECLOCK=\"UTC\"" >> /etc/rc.conf
```
- Configure your locale:
```
# cp /etc/default/libc-locales /etc/default/libc-locales.dist
# echo "en_AU.UTF-8 UTF-8" > /etc/default/libc-locales
# echo "en_AU ISO-8859-1" >> /etc/default/libc-locales
# xbps-reconfigure -f glibc-locales
```
- Configure a hostname for the box:
```
# echo nerv > /etc/hostname
```
- Confirm the ZFS modules are loaded:
```
# lsmod | grep zfs
zfs                  2629632  7
zunicode              331776  1 zfs
zavl                   16384  1 zfs
zcommon                36864  1 zfs
znvpair                57344  2 zfs,zcommon
spl                    73728  3 zfs,zcommon,znvpair
```
- Set your ZFS cachefile
```
# zpool set cachefile=/etc/zfs/zpool.cache nerv
```
- Let GRUB know which dataset to boot from:
```
# zpool set bootfs=nerv/ROOT/void nerv
```
## Install and configure the bootloader (GRUB)
- Configure /etc/default/grub based on your preferences
```
# cat /etc/default/grub
#
# Configuration file for GRUB.
#
GRUB_DEFAULT=0
#GRUB_HIDDEN_TIMEOUT=0
#GRUB_HIDDEN_TIMEOUT_QUIET=false
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Void"
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4 elevator=noop"
# Uncomment to use basic console
#GRUB_TERMINAL_INPUT="console"
# Uncomment to disable graphical terminal
#GRUB_TERMINAL_OUTPUT=console
GRUB_BACKGROUND=/usr/share/void-artwork/splash.png
#GRUB_GFXMODE=1920x1080x32
#GRUB_DISABLE_LINUX_UUID=true
#GRUB_DISABLE_RECOVERY=true
GRUB_DISABLE_OS_PROBER=true
```
The Linux kernel attempts to optimize disk I/O via the use of various "elevator" schemes. We want to let ZFS handle it’s own I/O optimizations so we’ll use the noop elevator, which is basically a FIFO queue sans any extra logic.
- Check that GRUB can recognise that we're using ZFS, because we're utilising /dev/disk/by-id paths we need to set the environment variable `ZPOOL_VDEV_NAME_PATH=1`:
```
# ZPOOL_VDEV_NAME_PATH=1 grub-probe /
zfs
```
- Install GRUB:

**For a single drive zpool:**
```
# ZPOOL_VDEV_NAME_PATH=1 grub-install /dev/sda
Installing for i386-pc platform.
Installation finished. No error reported.
```
**For a mirrored zpool:**
```
# ZPOOL_VDEV_NAME_PATH=1 grub-install /dev/sda
Installing for i386-pc platform.
Installation finished. No error reported.
# ZPOOL_VDEV_NAME_PATH=1 grub-install /dev/sdb
Installing for i386-pc platform.
Installation finished. No error reported.
```
- Configure Dracut:
```
# echo "hostonly=\"yes\"" > /etc/dracut.conf.d/zol.conf
# echo "nofsck=\"yes\"" >> /etc/dracut.conf.d/zol.conf
# echo "add_dracutmodules+=\" zfs \"" >> /etc/dracut.conf.d/zol.conf
# echo "omit_dracutmodules+=\" btrfs resume \"" >> /etc/dracut.conf.d/zol.conf
```
Setting "hostonly" tells dracut to skip all the extraneous needful things bundled into a generic kernel and generate an initramfs trimmed for booting our host’s particular hardware.
- Regenerate the kernel and initramfs. As of writing this, Voidlinux is on 4.14 kernel:
```
# xbps-reconfigure -f linux4.14
linux4.6: configuring ...
Executing post-install kernel hook: 10-dracut ...
Executing post-install kernel hook: 20-dkms ...
Available DKMS module: spl-0.7.5
Available DKMS module: zfs-0.7.5
Building DKMS module: spl-0.7.5 done.
Building DKMS module: zfs-0.7.5 done.
Executing post-install kernel hook: 50-grub ...
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.14.10_1
Found initrd image: /boot/initramfs-4.14.10_1.img
done
linux4.14: configured successfully.
```
## Sanity checks
```
# lsinitrd -m
Image: /boot/initramfs-4.14.11_1.img: 11M
========================================================================
Version:

dracut modules:
bash
dash
i18n
drm
kernel-modules
zfs
rootfs-block
terminfo
udev-rules
usrmount
base
fs-lib
shutdown
========================================================================
```
Our initramfs does include the ZFS dkms.
```
# lsinitrd | grep zpool.cache
-rw-r--r--   1 root     root         1360 Jan  4 11:48 etc/zfs/zpool.cache
```
Our initramfs also includes the reference to the zpool cache.
- Set a root password before leaving the chroot environment:
```
# passwd root
New password:
Retype new password:
passwd: password updated successfully
# exit
# pwd
/root
```
- Unmount the chroot environment:
```
# umount -n /mnt/{dev/pts,dev,sys,proc}
# mount | grep void
nerv/ROOT/void on / type zfs (rw,relatime,xattr,noacl)
```
- Export our pool:
```
# zpool export nerv
```
- Reboot
```
# reboot
```
- Enable basic networking and ssh:
```
# ln -s /etc/sv/dhcpcd /var/service/
# ln -s /etc/sv/sshd /var/service/
```
- Create user:
```
useradd -m -s /bin/bash -U -G wheel,audio,video,cdrom,input amir
```
- Snaphot your fresh install:
```
# zfs snapshot nerv/ROOT/void@04012018-fresh
# zfs list -t snapshot
NAME                                 USED  AVAIL  REFER  MOUNTPOINT
nerv/ROOT/void@04012018-fresh      0      -   316M  -
```
## Install and configure additional packages
```
# xbps-install autofs avahi borg cronie docker docker-compose fortune-mod git google-authenticator-libpam gvim-huge hddtemp intel-ucode lm_sensors mailx net-tools netcat nfs-utils ntp postfix rsync rxvt-unicode rxvt-unicode-terminfo samba smartmontools socklog-void tftp-hpa the_silver_searcher unzip wget zsh
```
- Configure additional permissions:
```
# usermod -aG docker,socklog amir
```
- Configure Samba:
```
# useradd -r -u 1050 -g 1050 -s /usr/bin/nologin samba
# smbpasswd -a samba
# smbpasswd -a amir
# usermod -aG samba amir
```

## Credits/Sources
- [Steppin' Into the Void](http://www.teamcool.net/posts/zol-voidlinux.html)
- [Installing Arch Linux on ZFS](https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_ZFS)
## Version
- **04/01/18:** Initial Release
- **14/01/18:** Updated with additional configuration and updated application configs for Void