# arch-install

[HardenedArray/Efficient Encrypted UEFI-Booting Arch Installation](https://gist.github.com/HardenedArray/31915e3d73a4ae45adc0efa9ba458b07)

```shell
# OBJECTIVE:  Install Arch Linux with encrypted root and swap filesystems and boot from UEFI.  

# Note this encrypted installation method, while perfectly correct and highly secure, CANNOT support encrypted /boot and 
# also CANNOT be subsequently converted to support an encrypted /boot!!!  A CLEAN INSTALL will be required!

# Therefore, if you want to have an encrypted /boot or will want an encrypted /boot system at some point in the future,
# please ONLY follow my encrypted /boot installation guide, which lives here:

https://gist.github.com/HardenedArray/ee3041c04165926fca02deca675effe1

# My encrypted /boot guide varies in several different, critically important, ways from the correct and secure encrypted 
# root / and swap installation process I have outlined below.

# Note:  This method supports both dedicated Arch installs and those who wish to install Arch on a multi-OS-UEFI booting system.

# External USB HDD/SSD Installers Notes:  Encrypted Arch installs can be booted and run from an external USB HDD or SSD, but
# only when the installation is correctly set up.  There are several necessary changes to my standard procedure you'll want
# to make during the install process.  Read my External USB HDD/SSD Installation section below before proceeding.

# VirtualBox Installers Notes:  This installation method can also be used to install Arch Linux as an UEFI-booting 
# Guest system in VirtualBox.  You must have UEFI-booting enabled in VBox's Guest System Settings prior to installation.
# I have written a separate guide dedicated to the specifics of achieving an encrypted Arch Linux VirtualBox installation.
# My Arch Linux VirtualBox Guest installation guide is available at:

https://gist.github.com/HardenedArray/d5b70681eca1d4e7cfb88df32cc4c7e6

# The official Arch installation guide contains details that you should refer to during this installation process.
# That guide resides at:  https://wiki.archlinux.org/index.php/Installation_Guide

# Download the archlinux-*.iso image from https://www.archlinux.org/download/ and its GnuPG signature.
# Use gpg --verify to ensure your archlinux-*.iso is exactly what the Arch developers intended.  For example:

$ gpg -v archlinux-2019.11.01-x86_64.iso.sig
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
gpg: assuming signed data in 'archlinux-2019.11.01-x86_64.iso'
gpg: Signature made Fri Nov  1 16:34:35 2019 UTC
gpg:                using RSA key 4AA4767BBC9C4B1D18AE28B77F2D434B9741E8AC
gpg: using pgp trust model
gpg: Good signature from "Pierre Schmitz <pierre@archlinux.de>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 4AA4 767B BC9C 4B1D 18AE  28B7 7F2D 434B 9741 E8AC
gpg: binary signature, digest algorithm SHA256, key algorithm rsa2048

# Burn the archlinux-*.iso  to a 1+ Gb USB stick.  On linux, do something like:

dd bs=4M if=archlinux-*.iso of=/dev/sdX status=progress oflag=sync  

# If running Windows, use Rufus to burn the archlinux-*.iso to your USB stick in DD mode.
# Also, if you are running BitLocker to encrypt your Windows system, read my BitLocker notes below, before proceeding.

# UEFI-Boot from your USB stick. If your USB stick fails to boot, ensure that Secure Boot is disabled in your UEFI configuration.

# Set your keymap only if not you are not using the default English language.

# It is typically wiser to be hard wired to the Net during installation. However, Arch supports WiFi-only installs.  Also 
# note that in mid-2020 the Arch devs deprecrated the use of `wifi-menu`.  The current installation images support `iwd`, 
# which provides `iwctl`.  Carefully note that `iwd` will NOT be installed on your new system.  If you will require WiFi
# access following reboot, we install `iwd` in the pacstrap command and then enable it after we enter arch-chroot, below.

# Connect to WiFi using:

iwctl

# then to connect to your WiFi station; do something like:

station <wlan device name> connect <wifi-station-name tab-auto-complete>

# then enter your wifi station's passphrase

# It is possible to access this guide from within your Arch installation environment using the built-in elinks text browser.  
# For those interested, open a new terminal at tty2 using ctrl-alt-f2, then use elinks to search for 'HardenedArray Gists' 
# which should return the URL of my Arch installation guides:

https://gist.github.com/HardenedArray/31915e3d73a4ae45adc0efa9ba458b07

# You can then return to your installation terminal using ctrl-alt-f1.

# Create and size partitions appropriate to your goals using gdisk.
# Carefully Note:  Multi-OS booters who have an existing EFI partition on their drive should NOT create a new EFI partition.  
# Instead, we will append Arch as another OS to your existing EFI partition.  See my Multi-OS-Booting Notes, below.

gdisk /dev/sdX

# Create the partitions you need:

Partition X = 100 MiB EFI partition # Hex code EF00
Partition Y = 250 MiB Boot partition # Hex code 8300
Partition Z = Choose a reasonable size for your encrypted root and swap system partition, or just size it to the 
last sector of your drive. # Hex code 8300.  

# Review your partitions with 'p'.
# Write your gdisk changes with 'w'.  
# Reboot, if necessary, so the kernel reads your new partition structure.
,
# I strongly recommend you zero-out each of of your new partitions prior to creating filesystems on them.  Obviously, multi-OS
# booters should NEVER zero-out an existing EFI partition.  You can either use the Arch installer's ddrescue or, if you don't
# mind not having a progress indicator, it's more efficient to run:

cat /dev/zero > /dev/sdXY followed by
cat /dev/zero > /dev/sdXZ

# Create filesystems for /boot/efi and /boot

mkfs.vfat -F 32 /dev/sdXX
mkfs.ext2 /dev/sdXY  # Note that ext4 or btrfs are also fine choices for your /boot partition.

# Encrypt and open your system partition

cryptsetup -c aes-xts-plain64 -h sha512 -s 512 --use-random luksFormat /dev/sdXZ

cryptsetup luksOpen /dev/sdXZ 2016-Global-OpSec-Champion-LyingHillary # (or use any word or phrase you're fond of)

# Create encrypted LVM partitions

# These steps create a required root partition and an optional partition for swap.  Note that using a swap file with BTRFS is
# a very poor idea.  Swap partitions are not controlled by BTRFS so they work fine.  Read the BTRFS ArchWiki before proceeding.  
# Also note that BTRFS fully supports, detects, and properly configures settings for all modern SSDs, which is the drive type
# almost everyone should be running when installing ArchLinux!  HDDs are only useful for infrequently accessed data, and 
# for storing your SSD's critical directories as encrypted backups.

# Modify this structure only if you need additional, separate partitions.  The sizes used below are only suggestions.
# The VG and LV labels 'Arch, root and swap' can be changed to anything memorable to you.  Use your labels consistently, below!

pvcreate /dev/mapper/2016-Global-OpSec-Champion-LyingHillary
vgcreate Arch /dev/mapper/2016-Global-OpSec-Champion-LyingHillary
lvcreate -L +512M Arch -n swap
lvcreate -l +100%FREE Arch -n root

# Create filesystems on your encrypted partitions

mkswap /dev/mapper/Arch-swap
mkfs.ext4 /dev/mapper/Arch-root  

# Note that Arch Linux fully supports btrfs, and btrfs is also an excellent filesystem choice for your encrypted root.  
# If you want a btrfs filesystem on your root logical volume, instead of 'mkfs.ext4 /dev/mapper/Arch-root', do this:

mkfs.btrfs /dev/mapper/Arch-root

# If you've created a btrfs root filesystem, do not forget to append 'btrfs-progs' to the pacstrap installation command
# we use immediately after correctly mounting our partitions below.

# Mount the new system 

mount /dev/mapper/Arch-root /mnt
swapon /dev/mapper/Arch-swap
mkdir /mnt/boot
mount /dev/sdXY /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sdXX /mnt/boot/efi

# Optional - Edit the Mirrorlist To Optimize Package Download Speeds

nano /etc/pacman.d/mirrorlist

# Copy one or two mirrors near your physical location to the top of the mirrorlist.

# Install your Arch system

# If you read the contents of https://www.archlinux.org/ you would know the Arch developers made significant
# changes to the 'base' package in October 2019.

# The new base-metapackage does not contain a kernel nor an editor and several other important packages.  
# We will be addressing those issues in our pacstrap command below.

# This installation command provides a decent set of basic system programs which will also support WiFi through 
# iwd's `iwctl` after initially booting into your Arch system.  Having WiFi following installation is particularly
# critical for anyone running a modern ultrabook, as most are equipped with WiFi-only access to the Net.  Recommended, yet 
# optional:  make and enjoy some fresh java while the following command completes.  Once completed, you'll only 
# be a few minutes away from putting your new system to serious work!

pacstrap /mnt base base-devel grub efibootmgr dialog wpa_supplicant linux linux-headers nano dhcpcd 
iwd lvm2 linux-firmware man-pages

# Create and review FSTAB

genfstab -U /mnt >> /mnt/etc/fstab  # The -U option pulls in all the correct UUIDs for your mounted filesystems.
cat /mnt/etc/fstab  # Check your fstab carefully, and modify it, if required.

# Enter the new system

arch-chroot /mnt /bin/bash

# Set the system clock

ln -s /usr/share/zoneinfo/UTC /etc/localtime  # This will harmlessly fail if your system's CMOS clock is already set to UTC.
hwclock --systohc --utc

# If you require WiFi access following reboot, enable iwd:

systemctl enable iwd

# Assign your hostname

echo MyHostName > /etc/hostname

# Set or update your locale

# If English is your native language, you need to edit exactly two lines to correctly configure your locale language settings:

a. In /etc/locale.gen **uncomment only**: en_US.UTF-8 UTF-8
b. In /etc/locale.conf, you should **only** have this line: LANG=en_US.UTF-8

# Now run:

locale-gen

# Set your root password

passwd

# Create a User, assign appropriate Group membership, and set a User password.  'Wheel' is just one important Group.

useradd -m -G wheel -s /bin/bash MyUserName

passwd MyUserName

# Configure mkinitcpio with the correct HOOKS required for your initrd image

nano /etc/mkinitcpio.conf

# Use this HOOKS statement:

HOOKS="base udev autodetect modconf block keymap encrypt lvm2 resume filesystems keyboard fsck"

# Note that recent ArchLinux installation images have shipped with a new version of /etc/mkinitcpio.conf.  The 
# only difference is that the new version uses '(' and ')' instead of dual double quotation marks: ' " " '.  Therefore,
# the current HOOKS statement should be:

HOOKS=(base udev autodetect modconf block keymap encrypt lvm2 resume filesystems keyboard fsck)

# You do not need or want 'resume' in your HOOKS statement if you are not using swap.

# Generate your initrd image

mkinitcpio -p linux

# Install and Configure Grub-EFI

# The correct way to install grub on an UEFI computer, irrespective of your use of a HDD or SSD, and whether you are
# installing dedicated Arch, or multi-OS booting, is:

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux

# Edit /etc/default/grub so it includes a statement like this:

GRUB_CMDLINE_LINUX="cryptdevice=/dev/sdYZ:MyDevMapperMountpoint resume=/dev/mapper/MyVolGroupName-MyLVSwapName"

# Maintaining consistency with the examples provided above, you would use something like:

GRUB_CMDLINE_LINUX="cryptdevice=/dev/sdXZ:2016-Global-OpSec-Champion-LyingHillary resume=/dev/mapper/Arch-swap"

# If you are not using swap, eliminate the 'resume' statement above.

# I have also noticed that recent releases of grub2 now offer this new option within /etc/default/grub:

# Uncomment to enable booting from LUKS encrypted devices
# GRUB_ENABLE_CRYPTODISK=y

# Note that you do NOT need to enable that cryptodisk statement to boot your LUKS encrypted / and swap ArchLinux system,
# assuming you are **NOT** trying to decrypt an encrypted /boot.  If you want to encrypt /boot, read my encrypted boot
# installation guide, which requires an entirely different, and incompatiable, installation procedure.

# Generate Your Final Grub Configuration:

grub-mkconfig -o /boot/grub/grub.cfg

# Exit Your New Arch System

exit

# Unmount all partitions

umount -R /mnt
swapoff -a

# Reboot and Enjoy Your Encrypted Arch Linux System!

reboot

# After you are satisfied that your Arch system is running well, if you are like most people not running an Arch server, 
# you'll want a Desktop Environment so you can begin using your new system productively.  See my:  'Installing a 
# Plasma-KDE Desktop Post Arch Install' section below for some ideas and an efficient DE installation process.

__________________________ 


If you ever get dropped to the EFI Shell prompt when powering up Arch Linux, which I most often notice within 
VirtualBox when running Arch Linux as UEFI-enabled Guest System, do the following:

At the Shell prompt, type the following entries, as indicated (also remember we used --bootloader-id=ArchLinux, above):

Shell> fs0:
fs0:> \EFI\ArchLinux\grubx64.efi

Hit Enter and now you should see your graphical grub Arch Linux menu.  Note my atypical use of backslashes.

To prevent being dropped to the EFI Shell prompt in the future, enter your Arch Linux system, become root, and do:

# nano /boot/efi/startup.nsh

In your startup.nsh file, add these two lines:

fs0:
\EFI\ArchLinux\grubx64.efi  

Save and exit nano.  To test that you will no longer be dropped to the EFI Shell prompt, poweroff, not reboot, and fire up 
your Arch Linux system again.

If you simply cannot bear the agony of the EFI Shell's five second wait prior to its loading of startup.nsh, hit any key, 
except for 'esc', and you should be immediately directed to your (hopefully, beautifully configured) grub graphical 
Arch Linux boot screen.

This solution also works when you have installed Arch Linux as an UEFI-enabled Guest system within VirtualBox.

__________________________


External USB HDD/SSD Encrypted Arch Installation:

Almost all of my standard Arch install procedure can be followed without modification when installing Arch to an external device.

However, if you already have an encrypted Arch installation on a system HDD/SSD, you must ensure the names assigned to your
PV, VG and LVs are different than whatever you used on your system drive's Arch installation.  Failure to use different names 
will cause major udev and therefore, /dev/mapper, assignment problems for you, especially when you try to mount your
multiple encrypted Arch drives!

Additionally, we don't want to instruct grub to use standard device names as these are very likely to change when using an 
external USB drive.  For example, our external SSD may be assigned /dev/sdc by udev during installation, but when we try to
initially boot from it, udev may assign that external SSD to /dev/sdb, resulting in an unbootable system.

The solution is to use PARTUUID, as opposed to a standard device name, in the cryptdevice statement in /etc/default/grub.

Therefore, instead of using this example from above:

GRUB_CMDLINE_LINUX="cryptdevice=/dev/sdXZ:2016-Global-OpSec-Champion-LyingHillary resume=/dev/mapper/Arch-swap"

Run 'blkid' as root, and find the correct PARTUUID for your external device's encrypted partition.

N.B.:  PARTUUIDs are completely unrelated to UUIDs.

Substitute the correct PARTUUID for the standard device name.  You should end up with a statement that looks similar to this:

GRUB_CMDLINE_LINUX="cryptdevice=PARTUUID=4d2aed94-92d4-7b5e-b8df-81d7554495cf4:ArchUSBSSD resume=/dev/mapper/ArchSSD-swap"

Now regardless of the device name assigned by udev to your external drive, the kernel will be able to find the
correct cryptdevice.

All other parts of my installation procedure should be followed without modification.

__________________________


Multi-OS-Booting Notes:

I UEFI boot and run more than five operating systems from my SSD.
All of my OSes UEFI boot from my single, 100 MiB, EFI partition.
All of my OSes have encrypted root and swap, utilizing my SSD's native hardware-based AES-256-bit encryption support
with BitLocker or Linux's software-based LUKS on LVM encryption to secure my data, when at rest.
My Arch Linux install is just another encrypted Linux OS installation that happens to reside on my SSD.

If you multi-boot, ensure you mount Arch's /boot/efi at your existing ESP partition.
If you installed Windows 10 first, your EFI partition is likely to be /dev/sda2.

In all cases, /boot, /boot/efi, and '/' partitions, at a minimum, are required to be mounted during Arch installation.

As an example, an EFI-addicted, multi-OS booter might be doing something like:

mount /dev/mapper/Arch-root /mnt
swapon /dev/mapper/Arch-swap
mkdir /mnt/boot
mount /dev/sda17 /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda2 /mnt/boot/efi

In this example, the user is likely to be using /dev/sda18 as the physical drive partition where their encrypted
Arch root and swap filesystems will reside.  Note the user's re-use of their existing EFI partition which resides
at /dev/sda2.

Adapt, as necessary, for your drive's partition structure.

Following successful Arch system installation, the path to your Arch-EFI boot file should be:

/boot/efi/EFI/ArchLinux/grubx64.efi

When you are multi-OS booting correctly, you should have one directory per operating system, each residing at:

/boot/efi/EFI/

__________________________


BitLocker Users on Windows Notes:

If you are running hardware-based BitLocker encryption on Windows, I recommend you Turn Off BitLocker encryption prior to
installing Arch, or any other operating system.  

As I don't use software-based BitLocker, I cannot say whether leaving it enabled during Arch installation will cause problems.
Obviously, if you experience issues, you could turn BitLocker off temporarily.

You can tell if you are using AES-256 bit hardware-based BitLocker encryption when you run from within PowerShell,
as an Administrator:

PS C:\WINDOWS\system32> manage-bde -status 

You see this line:
Encryption Method:    Hardware Encryption - 1.3.111.2.1619.0.1.2

Also note that hardware-based BitLocker can either encrypt, or decrypt, a multi-hundred GiB drive in less than 3 seconds.
You can re-enable BitLocker after your new encrypted Arch system is UEFI booting correctly and running smoothly.

__________________________


Installing a Plasma-KDE Desktop Post Arch Install

After you have rebooted into your new Arch system, and are satisfied that every aspect of your system is running correctly,
if you're like most people not running an Arch server, you will likely want to install a desktop so you can utilize your new 
Arch system productively.

Your choice of desktop environment is entirely up to you.  Personally, I have tried them all.  It is my opinion that if 
you are running a modern, reasonably powered PC or laptop you are doing yourself a significant disservice by running any 
of the 'lightweight desktops.'  I also think the Gnome DE is best suited for children, or unskilled users.  Keep in mind 
that you can install multiple desktops, and then choose which one to fire up at each login, but that is beyond the scope 
of this guide.

I prefer the Plasma5-KDE environment over all the others.  If you would like to efficiently install a full Plasma5-KDE 
environment, do the following, in this order:

# Log in as root, and not as a user

# Fully update your Arch system:

pacman -Syu  # If a new kernel becomes available and is now installed, reboot, before proceeding.

# If you don't have network connectivity in your Arch system, run:

systemctl start dhcpcd <ethernet or wlan interface name>

# Now that you have an updated system, do:

pacman -S linux-headers

pacman -S dkms  # This will automatically rebuild your kernel modules as new upstream kernels are released.

pacman -S xorg  #  This will install a mandatory X server.

pacman -S xorg-apps

reboot

__________________________


# Log in as root, and not as a user, and do:

pacman -S plasma-meta  # This large package set will also provide us with sddm, the recommended Plasma5 login manager.

systemctl enable sddm

systemctl enable NetworkManager  # After your next reboot you will have full, correct, networking support from boot.

pacman -S kde-applications-meta

pacman -S xdg-user-dirs

# If you want full (US English) spelling support for all of your applications, do:

pacman -S hunspell-en_US hyphen-en libmythes mythes-en aspell-en

# Everyone has their own font preferences, but I agree with Arch's initial ttf-font recommendations because they look great!:

pacman -S ttf-dejavu ttf-liberation

reboot

__________________________


# Log in to sddm's GUI as your user

# Your first stop is System Settings.  Tweak 'all the things' into full compliance with 'your way.'

# Go ROCK your fully enabled Plasma DE, and your properly encrypted Arch Linux system!!!

__________________________
```
