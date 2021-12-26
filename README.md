# Arch Linux Installation Guide

## Introduction
The purpose of this guide is to install and configure Arch Linux on a `UEFI` system. It will be setup with the following parameters:
+ UEFI
+ systemd-boot
+ NetworkManager
+ Xorg
+ KDE Plasma

**Note:** This is not intended to be a universal guide so installation procedures and configuration may differ based on individual setup.


## Table of Contents
  * [**Pre Installation**](#Pre-installation)
  * [**Disk Partitioning**](#preparing-the-disk-for-system)
    * [UEFI System](#Partitioning)
  * [**Base System Installation**](#base-system-installation)
    * [Base System](#install-base-system)
    * [Generate Fstab](#fstab)
  * [**Chroot**](#chroot)
    * [Date & Time](#set-time--date)
    * [Localisation](#localisation)
    * [Network Configuration](#network-configuration)
    * [ROOT Password](#set-root-password)
    * [Bootloader](#install-bootloader)
    * [CPU Microcode](#install-cpu-microcode)
    * [Network Manager](#install-network-manager)
## Pre installation
Before installation, make sure to:
+ Read the [official wiki](https://wiki.archlinux.org/title/installation_guide).
+ Aquire the ISO [image](https://archlinux.org/download/).
+ Verify the ISO signature.
+ Boot the live enviroment.

### Set Keyboard Layout:
The default keyboard layout is US. The layout can be changed with `loadkeys`:

UK:
```
loadkeys uk
```
### Verify the boot mode
Verify the boot mode and list the efivars directory: 
```
ls /sys/firmware/efi/efivars
```
The output should display the directory without any errors if the system is using UEFI. If not, it may be using Legacy/CSM.

### Update System Clock
Synchronise the system clock:
```
timedatectl set-ntp true
```
### Network Connection

#### Ethernet
Check the network interface is up:
```
ip link
```
The output should look similar to this:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp6s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 04:42:1a:ea:94:03 brd ff:ff:ff:ff:ff:ff
3: wlp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DORMANT group default qlen 1000
    link/ether f4:b3:01:69:fe:92 brd ff:ff:ff:ff:ff:ff
```
If the interface is DOWN, you should enable it:

```
ip link set enp6s0 up
```
OR
```
ip link set wlp5s0 up
```
+ `enp6s0` = wired interface
+ `wlp5s0` = wireless interface
#### Wi Fi
Connect to Wi Fi interface using iwctl:
```
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect SSID
```
#### Testing Network Connection:
```
ping archlinux.org
```
The ping should output a response if connected:
```
64 bytes from archlinux.org (95.217.163.246): icmp_seq=2 ttl=49 time=54.1 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=3 ttl=49 time=50.1 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=4 ttl=49 time=50.0 ms
```

 
# Partitioning
We need to partition the disk with a suitable layout depending on how we want to configure the system. 

## Unencrypted System

Recommended partition scheme:

| Partition     | Size             | Type            |
|:-------------:|:----------------:|:---------------:|
| `/boot`         | 512M           | EFI System      |
| `swap`          | 4GB            | Linux Swap      |
| `/`             | Remaining space| Linux filesystem| 

### Create partitions:
The disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` and `/dev/mmcblk`. 

List the disks to find the device name:

```
fdisk -l
```
Take note of the block device name of the disk you want to use for the installation. In this example it is `/dev/sda`:

```
Disk /dev/sda: 64 GiB, 68719476736 bytes, 134217728 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: A7B0EEB0-2290-4E4B-AD46-F0EFC0B1CBE0
```
Wipe the disk before creating the partitions using `gdisk`:
```
gdisk /dev/sda
```
+ Press <kbd>x</kbd> to enter expert mode.
+ Press <kbd>z</kbd> to wipe the disk.
+ Press <kbd>Y</kbd> and hit Enter to confirm.

Then create the partitions with `cgdisk`:
```
cgdisk /dev/sda
```
Create `boot` partition:
+ Navigate to <kbd>New</kbd> and hit <kbd>Enter</kbd>
+ Press <kbd>Enter</kbd> for 'First sector'.
+ Type `+512M` for last sector and hit <kbd>Enter</kbd>
+ Type `ef00` for partition type 'EFI System' and hit <kbd>Enter</kbd>
+ Type `boot` for partition label and hit <kbd>Enter</kbd>

Create `swap` partition:
+ Select `free space` and hit <kbd>Enter</kbd>
+ Navigate to <kbd>New</kbd> and hit <kbd>Enter</kbd>
+ Press <kbd>Enter</kbd> for 'First sector'.
+ Type `4G` for last sector and hit <kbd>Enter</kbd>
+ Type `8200` for partition type 'Linux Swap' and hit <kbd>Enter</kbd>
+ Type `swap` for partition label and hit <kbd>Enter</kbd>

Create `root` partition:
+ Select `free space` and hit <kbd>Enter</kbd>
+ Navigate to <kbd>New</kbd> and hit <kbd>Enter</kbd>
+ Press <kbd>Enter</kbd> for 'First sector'.
+ Press <kbd>Enter</kbd> for last sector and hit <kbd>Enter</kbd>
+ Type `8300` for partition type 'Linux filesystem' and hit <kbd>Enter</kbd>
+ Type `root` for partition label and hit <kbd>Enter</kbd>
+ Navigate to <kbd>Write</kbd>, type `YES` and hit <kbd>Enter</kbd>
+ Navigate to <kbd>Quit></kbd> and type `YES` and hit <kbd>Enter</kbd> to write changes.

### Formatting
After creating the partitions they need to be formatted with suitable filesystems:
  
Format `/dev/sda1` as `FAT32`. This will be the `boot` partition.
```
mkfs.fat -F 32 /dev/sda1 
```
Create the `swap` and enable it:
```
mkswap /dev/sda2 
swapon /dev/sda2
```
Format `/dev/sda3` as `EXT4`. This will be the `root` partition.
```
mkfs.ext4 /dev/sda3
```
### Mounting the file systems
Create and mount the partitions to their respective directories.

Mount `/dev/sda3` to `mnt`. This will be `/`.
```
mount /dev/sda3 /mnt
```
Create a `/boot` mountpoint:
```
mkdir /mnt/boot
```
Mount `/dev/sda1` to `/mnt/boot`. This will be `/boot`.
```
mount /dev/sda1 /mnt/boot
```
The `swap` partition does not need to be mounted since it is enabled.

Check the partitions are correct using the `lsblk` command:
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 701.3M  1 loop /run/archiso/airootfs
sda      8:0    0    64G  0 disk 
├─sda1   8:1    0   512M  0 part 
├─sda2   8:2    0     4G  0 part 
└─sda3   8:3    0  59.5G  0 part 
sr0     11:0    1 850.3M  0 rom  /run/archiso/bootmnt
```
# Base System Installation

### Install base system
To install the base system packages:
```
pacstrap /mnt base base-devel linux linux-firmware 
```
You may want to install additional needed packages not included within the base install such as:
+ `sudo`
+ `nano`
+ `vim`
+ `git`

**Note: While base-devel is not included within the default pacstrap command on the wiki, many packages will not work without it so we will include it.**

### Configuring the system
Now the base system needs to configured as follows:

#### Fstab:
Generate the fstab file which will be used to mount the filesystems:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
Check that the fstab file was generated properly and the entries are correct:
```
cat /mnt/etc/fstab
```
If the fstab file does not contain the entries, regenerate the file.

## Chroot

Chroot into the system:
```
arch-chroot /mnt
```
### Set Time & Date
Set the timezone:
```
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```
**Note: Available timezones can be listed with `timedatectl list-timezones`**.

Set the HWClock:
```
hwclock --systohc
```
This will generate `/etc/adjtime`.

## Localisation

Edit /etc/locale.gen and uncomment `en_US.UTF-8 UTF-8` and any other needed locales. e.g `en_GB.UTF-8 UTF-8`.

Then generate the locales:
```
locale-gen
```
Set the LANG variable:
```
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```
Set the console keyboard layout:
```
echo "KEYMAP=uk" > /etc/vconsole.conf
```
## Network configuration

### Set Hostname
Set the hostname e.g `arch`:
```
echo "arch" > /etc/hostname
```

### Configure the `hosts` file:
```
echo "127.0.0.1  localhost" >> /etc/hosts
echo "::1        localhost" >> /etc/hosts                   
echo "127.0.1.1  localhost.localdomain   arch" >> /etc/hosts
```
**Note: Append `arch` with the hostname you have set.**
### Set Root password
Set the root user password:
```
passwd
```
### Install Bootloader
A bootloader needs to be installed for system initialisation. For this configuration systemd-boot will be the default option:
```
bootctl install
pacman -S efibootmgr
```
Create and configure `/boot/loader/loader.conf`:

```
default  arch.conf
timeout  4
console-mode max
editor   no
```
Create a loader configuration file:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID="UUID" quiet splash rw
```
Additionally, install a secondary backup kernel and create the configuration file like the example above. E.g for LTS kernel, `/boot/loader/entries/arch-lts.conf` and `/boot/loader/entries/arch-lts-fallback.conf`. 

Rename `/vmlinuz-linux` to `/vmlinuz-linux-lts` and `/initramfs-linux.img` to `initramfs-linux-lts.img`.

### Install CPU Microcode
Microcode provides stability and security updates for the CPU. They should be installed for optimal operation. 
#### For AMD CPUs:
```
pacman -S amd-ucode
```
#### For Intel CPUs:
```
pacman -S intel-ucode
```
### Install Network Manager
A network manager is used to connect to a network interface and manage settings. NetworkManager will be used for this configuration:
```
pacman -S networkmanager
```
Enable the service:
```
systemctl enable NetworkManager.service
```
### Reboot
```
exit
umount -a
reboot
```
# Post Installation

### Users
Create a user and append `username` with your name:
```
useradd -m -G wheel -s /bin/bash username
```
Set the user password:
```
passwd username
```
## Post Install Configuration
Ensure you are connected to the internet before doing these steps.

### Sudoers
Edit the sudoers file and uncomment this line:
```
EDITOR=nano visudo
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL

### Multilib

Multilib support can be enabled by editing `/etc/pacman.conf` and uncommenting this line:
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```
### Update the system
Before installing any other packages, update the system:
```
sudo pacman -Syu
```
### Display server
A display server is needed to process and manage the GUI. Xorg will be the default choice in this configuration. Install the `xorg-server` pacxkage:
```
sudo pacman -S xorg-server
```
## Drivers
You will want to install the appropriate drivers for your hardware and system configuration for better optimisation and performance.
Since I will be using an RDNA2+ AMD GPU, `xf86-video-amdgpu` and `vulkan-radeon` will be installed:
```
sudo pacman -S xf86-video-amdgpu
sudo pacman -S vulkan-radeon
```
Optionally, to enable 32 bit application support install the `lib32-mesa` and `lib32-vulkan-radeon` packages:
```
sudo pacman -S lib32-mesa
sudo pacman -S lib32-vulkan-radeon
```
To enable hardware acceleration install `libva-mesa-driver`,  `lib32-libva-mesa-driver` for VA-API and `mesa-vdpau` and `lib32-mesa-vdpau` for VDPAU:
```
sudo pacman -S libva-mesa-driver
sudo pacman -S lib32-libva-mesa-driver
sudo pacman -S mesa-vdpau
sudo pacman -S lib32-mesa-vdpau
```
Edit `/etc/mkinitcpio.conf` and add `amdgpu` to the kernel MODULES:
```
sudo nano /etc/mkinitcpio.conf
```
```
The following modules are loaded before any boot hooks are
# run.  Advanced users may wish to specify all system modules
# in this array.  For instance:
#     MODULES=(piix ide_disk reiserfs)
MODULES=(amdgpu)
```
Don't forget to regenerate the initramfs:
```
sudo mkinitcpio -p linux
```
### KDE Applications

Packages | Description
--------- | ----------
plasma | KDE Plasma Desktop Environment.
konsole | KDE Terminal.
dolphin | KDE File Manager.
ark | Archiving Tool.
kate | Text Editor.
kcalc | Scientific Calculator.
spectacle | KDE screenshot capture utility.
partitionmanager | KDE Disk & Partion Manager.
### Display manager
A display manager is a graphical user interface which is displayed after the boot process. The `plasma-meta` package will provide a minimal installation: 
```
sudo pacman -S plasma-meta
```
### Login Manager
A login manager is needed to log into the desktop enviroment. However, `sddm` is a dependency of the `plasma-meta` package, so installing it is not required. Simply enable the service:
```
sudo systemctl enable sddm.service
```
### File Manager
A file manager is a piece of software to manage and order folders and data on the system. Since I am using KDE Plasma, the `dolphin` file manager will be installed:
```
sudo pacman -S dolphin
```
### Terminal emulator
A Linux system isn't fully functional without a terminal emulator. The default terminal emulator used by KDE Plasma is `konsole`, which we will install:
```
sudo pacman -S konsole
```
### Web browser
Obviously a modern system will want a web browser to navigate the web, so let's install `firefox`. You may choose another browser of your choice.
```
sudo pacman -S firefox
```
### Install AUR Helper
Install git:
```
sudo pacman -S git
```
Git clone yay and build the package:
```
git clone https://aur.archlinux.org/yay.git
cd yay/
makepkg -si
```
# Avahi

### Firewall
Arch Linux does not have any ports open by default. However, it is recommended to install a suitable Firewall for better security:
```
pacman -S ufw
```
Basic UFW configuration which will deny all by default, allow any protocol from inside a 192.168.0.1-192.168.0.255 LAN, and allow incoming Deluge and rate limited SSH traffic from anywhere:
```
sudo ufw default deny
sudo ufw allow from 192.168.0.0/24
sudo ufw allow Deluge
sudo ufw limit ssh
```
Enable the service:
```
sudo ufw enable
sudo systemctl enable ufw.service
```

### SSD TRIM
If you use an SSD, you can enable periodic TRIM to optimise drive performance and longevity:

Enable the service:
```
sudo systemctl enable fstrim.timer
```

### Package Cache Hook
To improve system performance enable `paccache.timer` which will clear the package cache weekly.
```
sudo pacman -S pacman-contrib
sudo systemctl enable paccache.timer
```

### Set Keyboard Layout
```
sudo localectl set-x11-keymap gb
```

## Audio
By default Arch uses PulseAudio as an audio server. Pipewire is newer and provides better functionality. 
Install and enable PipeWire:
```
sudo pacman -S pipewire
sudo pacman -S pipewire-pulse
```
## Tweaks

Let's make some system tweaks for better optimisation.

#### Update Mirrorlist
The mirrorlist should be configured for faster download speeds.

Install `reflector`:

```
sudo pacman -S reflector
```
Make a backup of the mirrorlist:
```
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```
Configure `/etc/xdg/reflector/reflector.conf`:
```
--save /etc/pacman.d/mirrorlist
--country GB,DE,FR
--protocol https
--latest 20
--sort rate
--age 12
```
Then enable and start `systemctl reflector.timer` to rank the mirrors weekly:
```
sudo systemctl enable reflector.timer
sudo systemctl start reflector.timer
```
You can create a pacman hook that will start reflector.service and remove the .pacnew file created every time pacman-mirrorlist gets an upgrade:
Create the file `/etc/pacman.d/hooks/mirrorupgrade.hook` and edit it:
```
sudo mkdir -p /etc/pacman.d/hooks
sudo touch /etc/pacman.d/hooks/mirrorupgrade.hook
sudo nano etc/pacman.d/hooks/mirrorupgrade.hook
```
```
[Trigger]
Operation = Upgrade
Type = Package
Target = pacman-mirrorlist

[Action]
Description = Updating pacman-mirrorlist with reflector and removing pacnew...
When = PostTransaction
Depends = reflector
Exec = /bin/sh -c 'systemctl start reflector.service; [ -f /etc/pacman.d/mirrorlist.pacnew ] && rm /etc/pacman.d/mirrorlist.pacnew'
```

#### Tear Free & Free Sync
TearFree and Free Sync will prevent screen tearing and flicker. Create the file `/etc/X11/xorg.conf.d/20-amdgpu.conf` and insert the following:
```
sudo nano /etc/X11/xorg.conf.d/20-amdgpu.conf
```
```
Section "Device"
     Identifier "AMD"
     Driver "amdgpu"
     Option "TearFree" "true"
     Option "VariableRefresh" "true"
EndSection
```
#### Power Saving
Power saving features can be disabled to improve network latency issues. Since I use an `Intel AX200` chipset on desktop, I will create the file `/etc/modprobe.d/iwlmvm.conf` and disable power saving state:

```
sudo touch /etc/modprobe.d/iwlmvm.conf
```
Edit the newly created file:
```
options iwlmvm power_scheme=1
```
**Note: Laptop users will want to keep the power saving features enabled to preserve battery.**

### Optional

These are just a collection of optional tweaaks that the majority of users may not need or use.

### Install Missing Firmware
Although the missing firmware is not needed in most cases, you can install them to remove the warning messages when regenerating the `initramfs`:

```
sudo pacman -S aic94xx-firmware wd719x-firmware upd72020x-fw
```
### Disable Secondary Login Screen
If using multiple monitors, the secondary display can be disabled for aesthetics.

First we need to find the name of the display ports connected to the system:

```
xrandr | grep ' connected'
```
The output says the connected displays are `DisplayPort-1` and `DisplayPort-2`.
```
DisplayPort-1 connected primary 1920x1080+0+0 
DisplayPort-2 connected 1920x1080+1920+0 
```
As we can see, my secondary monitor port is `DisplayPort-2`.
Once we know the names of the connected ports, create the file `/usr/share/sddm/scripts/Xsetup`:
```
sudo nano /usr/share/sddm/scripts/Xsetup
```
Edit the file and append `DisplayPort-2` with the name of the secondary display port corresponding to the output from `xrandr`.

```
#!/bin/sh
# Xsetup - run as root before the login dialog appears

xrandr --output DisplayPort-2 --off
```
Finally, edit `/etc/sddm.conf` to point to the script we set up above:

```
[X11]
# Xsetup script path
# A script to execute when starting the display server
DisplayCommand=/usr/share/sddm/scripts/Xsetup
```


