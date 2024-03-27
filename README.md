# Raspberry Pi 5 with Arch Linux ARM and Encrypted Root

Until our Raspberry Pi 5 is fully supported by [Arch Linux ARM](https://archlinuxarm.org/platforms) we can get it running by "... removing U-Boot and replacing the mainline kernel with a directly booting kernel from the Raspberry Pi foundation" - from [this excellent guide](https://kiljan.org/2023/11/24/arch-linux-arm-on-a-raspberry-pi-5-model-b/) which you should read first.

This guide boots an encrypted root Arch Linux ARM on an SSD connected to a Raspberry Pi 5. Root is automatically unlocked from a key file on a USB key or you can enter the password with a keyboard. While it's not covered here, adding unlock via SSH should be the same as any other Arch install. Hopefully.

If you're looking for a guide to install to a SD card instead of the SSD, try the aforementioned guide above or see the incomplete notes at the end for trying to build in a host that's not a Pi.

## Hardware

* Raspberry Pi 5
* SD card
* NVMe connected via PCIe (tested with [Pimoroni NVMe Base](https://shop.pimoroni.com/products/nvme-base))
* SSD (tested with Solidigm P44 Pro, 512GB)
* USB key (to hold the encrypted root key file)

## Preamble

We'll need Raspberry Pi OS Lite installed on the SD card to kick things off. They have a great [install guide](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system). I used the Raspberry Pi Imager so I could click buttons and add my public SSH key to the image easily.

Boot the Pi from the SD card and connect via SSH.

Run the following commands as root.

```bash
sudo su -
```

## Install

We'll need to change the boot order to boot from NVMe and you can learn all about [NVMe SSD booting](https://www.jeffgeerling.com/blog/2023/nvme-ssd-boot-raspberry-pi-5) from Jeff. Thanks Jeff.

My [preferred order](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#BOOT_ORDER) is `0xf461` which corresponds to (and is read from right to left):

* **1** First, SD card (which will only used to rescue the system in the future - otherwise you'll have to disconnect the SSD to boot from an SD card)
* **6** Second, NVMe
* **4** Third, USB

Edit EEPROM:

```bash
rpi-eeprom-config --edit

BOOT_ORDER=0xf416
PCIE_PROBE=1
```

For the [lowest power state on halt](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#POWER_OFF_ON_HALT) because my Pi doesn't have a HAT and doesn't use GPIO pins:

```
POWER_OFF_ON_HALT=1
WAKE_ON_GPIO=0
```

Install required packages:

```bash
apt update
apt install libarchive-tools cryptsetup curl

# cryptsetup for luks encryption
# libarchive-tools for bsdtar
# curl for ... curl
```

Download Arch Linux ARM:

```bash
curl -L -o alarm.tar.gz 'http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz'
curl -L -o alarm.tar.gz.sig 'http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz.sig'
```

Verify file signature:

```bash
gpg --keyserver 'hkp://keyserver.ubuntu.com' --keyserver-options auto-key-retrieve --verify 'alarm.tar.gz.sig'
```

Mount the USB key and create a [key file](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Keyfiles). My USB key already formatted as ext4:

```bash
mkdir -p /media/usb/
mount /dev/sda1 /media/usb/
dd bs=512 count=4 if=/dev/random of=/media/usb/unlock.key iflag=fullblock
```

Partition the target SSD with 2 partitions (Arch's install guide has a [section on partitioning](https://wiki.archlinux.org/title/Installation_guide)):

* 512MiB boot: `/dev/nvme0n1p1`
* Remaining space for root: `/dev/nvme0n1p2`

Format boot device:

```bash
mkfs.vfat -F 32 /dev/nvme0n1p1
```

Decide on your `--pbkdf-*` parameters:

> By default cryptsetup will [benchmark](https://man7.org/linux/man-pages/man8/cryptsetup-luksformat.8.html) your host and use a memory-hard PBKDF algorithm that can require up to 4GB of RAM. If these settings exceed your Raspberry Pi's available RAM, it will make it impossible to unlock the partition. To work around this, set the [--pbkdf-memory](https://man7.org/linux/man-pages/man8/cryptsetup-luksformat.8.html) and [--pbkdf-parallel](https://man7.org/linux/man-pages/man8/cryptsetup-luksformat.8.html) arguments so when you multiply them, the result is less than your Pi's total RAM.

-- from the excellent [Pi Encrypted Boot with Remote SSH](https://github.com/ViRb3/pi-encrypted-boot-ssh) guide.

Setup luks and format root partition:

```bash
cryptsetup luksFormat --pbkdf-memory=1024000 --pbkdf-parallel=1 /dev/nvme0n1p2
cryptsetup luksAddKey /dev/nvme0n1p2 /media/usb/unlock.key
cryptsetup --key-file /media/usb/unlock.key open --type luks /dev/nvme0n1p2 root
mkfs.ext4 -m 1 /dev/mapper/root
```

Make a note of these device's UUID, we'll need them later (the SD card, `mmcblk0`, can be ignored):

```bash
lsblk -f

# NAME       FSTYPE UUID
# nvme0n1
# ├─nvme0n1p1
# │          vfat   aaaa-aaaa                            <- boot
# └─nvme0n1p2
#            crypto bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb <- encrypted root
#   └─root   ext4   cccccccc-cccc-cccc-cccc-cccccccccccc <- unlocked root
# sda
# └─sda1     ext4   dddddddd-dddd-dddd-dddd-dddddddddddd <- usb key
```

You can copy this text and fill it out unless your super power is memorizing UUIDs:

```
aaaa-aaaa = boot =
bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb = encrypted root =
cccccccc-cccc-cccc-cccc-cccccccccccc = unlocked root =
dddddddd-dddd-dddd-dddd-dddddddddddd = usb key =
```

Mount root device:

```bash
mount /dev/mapper/root /mnt/
```

Extract Arch Linux ARM's files:

```bash
bsdtar -xpf alarm.tar.gz -C /mnt/
```

Mount boot:

```bash
mount /dev/nvme0n1p1 /mnt/boot
```

Setup the chroot:

```bash
mount -t proc none /mnt/proc/
mount -t sysfs none /mnt/sys/
mount -o bind /dev /mnt/dev/
mount -o bind /dev/pts /mnt/dev/pts/

mkdir -p /mnt/run/systemd/resolve/
touch /mnt/run/systemd/resolve/resolv.conf
mount --bind /etc/resolv.conf /mnt/run/systemd/resolve/resolv.conf
```

Enter the chroot:

```bash
LANG=C chroot /mnt/ /bin/bash
```

Normal Arch [init things](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4) (see the Pi 4 install guide):

```bash
pacman-key --init
pacman-key --populate archlinuxarm
```

Update the system and swap kernels:

```bash
pacman -R linux-aarch64 uboot-raspberrypi
pacman -Syu --overwrite "/boot/*" linux-rpi-16k

# Some packages provide hints on actions to take and you'll have to decide which ones need attention.
# mkinitcpio probably failed, we'll fix that soon.
```

Update your [Localization](https://wiki.archlinux.org/title/Installation_guide#Localization):

```bash
vi /etc/locale.gen

# enable C.UTF-8
# enable your locale
```

```bash
vi /etc/locale.conf

LANG=<your locale>
```

Generate locales:

```bash
locale-gen
```

Update your [time zone](https://wiki.archlinux.org/title/System_time#Time_zone)

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

Update [mkinitcpio hooks](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition):

```bash
vi /etc/mkinitcpio.conf

# Update HOOKS to add encrypt.
# Update HOOKS to remove kms (my Pi froze on boot with it enabled).
# Update HOOKS to remove microcode (not supported on this platform).
# Maybe update MODULES if your usb key uses a file system that isn't ext4.

HOOKS=(base udev autodetect modconf keyboard keymap consolefont block encrypt filesystems fsck)
```

Update fstab:

```bash
vi /etc/fstab

# <file system> <dir> <type> <options> <dump> <pass>
UUID=cccccccc-cccc-cccc-cccc-cccccccccccc / ext4 rw,relatime 0 1
UUID=aaaa-aaaa /boot vfat rw,relatime 0 0
```

Update linux command line by replacing `root=/dev/mmcblk0p2`:

```bash
vi /boot/cmdline.txt

# Replace root=/dev/mmcblk0p2 with this (and update the cryptkey file system parameter, ext4, to the file system of your usb key):
cryptdevice=UUID=bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb:root cryptkey=UUID=dddddddd-dddd-dddd-dddd-dddddddddddd:ext4:/unlock.key root=/dev/mapper/root

# Full line should look something like this:
cryptdevice=UUID=bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb:root cryptkey=UUID=dddddddd-dddd-dddd-dddd-dddddddddddd:ext4:/unlock.key root=/dev/mapper/root rw rootwait console=serial0,115200 console=tty1 fsck.repair=yes
```

Update Pi's configuration (optional):

```
vi /boot/config.txt

# I disable things I don't need:
# - vc4-kms-v3d (try disable this if your Pi freezes on boot)
# - wifi
# - bluetooth
```

If you're looking to [unlock using SSH](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Remote_unlocking_of_root_(or_other)_partition), now is the time.

Run mkinitcpio:

```bash
mkinitcpio -P

# You should see: Initcpio image generation successful
# Without this image your Pi won't boot.
```

Add your public SSH key to root:

```bash
mkdir -p /root/.ssh && chmod 0700 /root/.ssh
echo "ssh-ed25519 [...]" | tee /root/.ssh/authorized_keys
chmod 0600 /root/.ssh/authorized_keys
```

Remove default (insecure) user and disable root's password because we'll be using SSH to connect (optional):

```bash
userdel -r alarm
usermod -p '!' root
```

Periodic pacman cache cleaner (optional):

```bash
pacman -S pacman-contrib
systemctl enable paccache.timer
```

Update your hostname:

```bash
vi /etc/hostname
```

Exit the chroot:

```
sync
history -c && exit
```

Power off the Pi:

```
poweroff
```

Real life steps:

1. Remove power
1. Remove the SD card
1. Apply power
1. Wait
1. Connect to your Pi using SSH as root

Enjoy!

## Troubleshooting

Unfortunately, plugging the Pi into a monitor and looking at the screen is the easiest debug method.

You might need to wait slightly longer for the Pi to boot due to unlocking the root partition.

Still nothing?

Power off, insert the SD card into the Pi, boot and SSH. Unlock and mount partitions then enter the chroot and start pressing buttons. Double check your UUID's and `cmdline.txt`. Perhaps you missed the `UUID=` when referencing the disk?

Maybe you'll need to remove the `kms` minitcpio HOOKS. Maybe you'll need to disable `vc4-kms-v3d` in the `config.txt`.

## Miscellaneous Notes

You can build the Arch Linux ARM image on a Arch host computer, like x86_64 or in a [VM](https://gitlab.archlinux.org/archlinux/arch-boxes) (basic image, QCOW2). You can probably use other linux distro's with the right packages installed.

Install packages required for the [chroot](https://wiki.archlinux.org/title/QEMU#Chrooting_into_arm/arm64_environment_from_x86_64):

```bash
pacman -S qemu-user-static qemu-user-static-binfmt arch-install-scripts
```

Create files to temporarily hold the file systems:

```bash
fallocate -l 512MiB pi-boot
fallocate -l 4GiB pi-root
```

Mount the files as devices:

```bash
losetup -f pi-boot
losetup -f pi-root
```

Check which file was assigned to which loop device:

```bash
losetup -l
```

From there you can install to the loop devices.

You can transfer your boot and root partitions from the loop devices to a real device using:

```bash
rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" /mnt/src//* /mnt/dest/
```
