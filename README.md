# Arch Linux Installation Guide

## Introduction
The purpose of this guide is to install and configure Arch Linux with KDE Plasma desktop enviroment. We will go through how I set up my system from beginning to end.

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
  * [Post Installation](#post-installation)
    * [Add user](#users)
    * [Sudo Command](#sudoers)
  * [User Login](#login-as-user)
    * [Display Server & GPU Drivers](#xorg--gpu-drivers)
    * [Multilib Repository](#enable-multilib-repository)
    * [Display Manager (SDDM)](#display-manager---sddm)
    * [Audio & Bluetooth](#audio-utilities--bluetooth)
    * [Personal Applications](#personal-applications)
  * [Extras](#extras)
    * [Final Thoughts](#final-thoughts)
    * [Maintenance & Performance](#maintenance--performance-tuning)
    * [Tweaks](#tweaks)
    
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
```
### Bootloader Configuration
Create and configure `/boot/loader/loader.conf`:

```
default  arch.conf
timeout  4
console-mode max
editor   no
```
Create a loader configuration file `/boot/loader/entries/arch.conf`:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=[UUID] quiet splash rw
```
Create and configure the fallback file `/boot/loader/entries/arch-fallback.conf`:
```
title   Arch Linux (fallback initramfs)
linux   /vmlinuz-linux
initrd  /amd-ucode
initrd  /initramfs-linux-fallback.img
options root="UUID=[UUID] rw
```
Additionally, install a secondary backup kernel and create the configuration file like the example above. ( e.g. for LTS kernel, `/boot/loader/entries/arch-lts.conf` and `/boot/loader/entries/arch-lts-fallback.conf`. 

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
## Post Installation

### Login as ROOT

### Users
Create a user and append `username` with your name:
```
useradd -m -G wheel -s /bin/bash username
```
Set the user password:
```
passwd username
```
Ensure you are connected to the internet before doing these steps.

### Sudoers
Edit the sudoers file and uncomment this line:
```
EDITOR=nano visudo
```
Uncomment the `wheel` group to allow members of group wheel to execute any command
```
%wheel ALL=(ALL) ALL
```
### Logout ROOT

```
exit
```
## Login as USER

### Update the system
Before installing any other packages, update the system:
```
sudo pacman -Syu
```

### Xorg & GPU Drivers
A display server is needed to process and manage the GUI. Install the `xorg` package and the Xorg display drivers:
```
sudo pacman -S xorg xf86-video-your gpu type]
```
+ For AMD GPUs, install `xf86-video-amdgpu`
+ For NVIDIA GPUs, install `nvidia and `nvidia-settings`. 
+ For legacy Radeon GPUs, install `xf86-video-ati`.
+ For Intel iGPU install `xf86-video-intel`
### Enable Multilib Repository
Multilib includes 32-bit software and library applications. (e.g. Steam).

Edit `etc/pacman.conf` and uncomment these lines to enable the `Mulilib` repo:

```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```
### MESA Libraries (32 Bit Support)
Optionally, to enable 32 bit application support install the `lib32-mesa` package:
```
sudo pacman -S lib32-mesa
```
### Vulkan API
Vulkan is a 3D graphics API backend used by games and video applications. For Vulkan support, install the `vulkan-radeon` package:
```
sudo pacman -S vulkan-radeon
```
Optionally, install `lib32-vulkan-radeon` for 32-bit application support:
```
sudo pacman -S lib32-vulkan-radeon
```
### Hardware Video Acceleration
To enable hardware acceleration install `libva-mesa-driver`,  `lib32-libva-mesa-driver` for VA-API and `mesa-vdpau` and `lib32-mesa-vdpau` for VDPAU:
```
sudo pacman -S libva-mesa-driver
sudo pacman -S lib32-libva-mesa-driver
sudo pacman -S mesa-vdpau
sudo pacman -S lib32-mesa-vdpau
```
### Early KMS Loading
To load the GPU driver modules early, edit `/etc/mkinitcpio.conf` and add `[your-driver-module]` to the kernel MODULES:
+ For AMD add `amdgpu`.
+ For NVIDIA add `nvidia`, 'nvidia_modeset`, `nvidia_uvm` and `nvidia_drm`.
+ For legacy AMD Radeon, add `radeon`.
+ For Intel iGPU, add `i915`.
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
### Display Manager (SDDM)
Install and enable SDDM:
```
sudo pacman -S sddm
systemctl enable sddm.service
```
### KDE Applications
These are the core KDE applications that I will install for my setup:

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

```
sudo pacman -S plasma konsole dolphin ark kate kcalc spectacle partitionmanager
```
### Audio Utilities & Bluetooth
To configure audio and enable Bluetooth:
```
sudo pacman -S alsa-utils bluez bluez-utils
```
Packages | Description
--------- | ----------
alsa-utils | This includes (among other utilities) the `alsamixer` and `amixer` utilities.
bluez | Provides the Bluetooth protocol stack.
bluez-utils | Provides the `bluetoothctl` utility.

### Personal Applications
This is a list of independent applications that I personally use on my system:
```
sudo pacman -S firefox steam mpv spotify git neofetch
```
Packages | Description
--------- | ----------
firefox | Mozilla Firefox Web Browser.
steam| A digital video game distribution service.
mpv | Video player
spotify| Proprietary music streaming service. 
git | Github command-line utility tools.
timeshift | System restore/snapshot utility 
neofetch | Neofetch is a command-line system information tool.

## Final Thoughts

Everything required has now been installed and the system should now be ready to use. Below are some optional extra applications and tweaks that you can make to enhance user experience and performance.  

## Extras

### Install AUR Helper
Yet Another Yogurt - An AUR Helper

Yay is a pacman wrapper for managing, installing and updating packages.

Install `yay`:
```
git clone https://aur.archlinux.org/yay.git
cd yay/
makepkg -si
```
### PipeWire
By default Arch uses PulseAudio as an audio server. Pipewire is newer and provides better functionality. 
Install and enable PipeWire:
```
sudo pacman -S pipewire
sudo pacman -S pipewire-pulse
```
## Security
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
## Maintenance & Performance Tuning

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
### Update Mirrorlist
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
### Power Saving
Power saving features can be disabled to improve network latency issues. Since I use an `Intel AX200` chipset on desktop, I will create the file `/etc/modprobe.d/iwlmvm.conf` and disable power saving state:

```
sudo touch /etc/modprobe.d/iwlmvm.conf
```
Edit the newly created file:
```
options iwlmvm power_scheme=1
```
**Note: Laptop users will want to keep the power saving features enabled to preserve battery.**

## Tweaks

Let's make some system tweaks for better optimisation.

### Tear Free & Free Sync
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

## Optional

These are just a collection of optional tweaks that the majority of users may not need or use.

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

#
