# Arch Linux Installation Guide

## Introduction
The purpose of this guide is to install and configure Arch Linux on a `UEFI` system. It will be setup with the following parameters:
+ UEFI
+ systemd-boot
+ NetworkManager
+ Xorg
+ KDE Plasma

**Note:** This is not intended to be a universal guide so installation procedures and configuration may differ based on individual setup.

## Pre installation
Before installation, make sure to:
+ Read the [official wiki](https://wiki.archlinux.org/title/installation_guide).
+ Aquire the ISO [image](https://archlinux.org/download/).
+ Verify the ISO signature
+ Boot the live enviroment

## Set Keyboard Layout:
The default keyboard layout is US. The layout can be changed with `loadkeys`:

UK:
```
loadkeys uk
```
## Verify the boot mode
Verify the boot mode andlist the efivars directory: 
```
ls /sys/firmware/efi/efivars
```
The output should display the directory without error if the system is using UEFI. If not, it may be using Legacy/CSM.
# Update System Clock
Synchronise the system clock:
```
timedatectl set-ntp true
```

## Network Connection

### Ethernet
Firstly, check the network interface is up:
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
ip link set enp0s3 up
```
### Wi Fi
Connect to Wi Fi interface using iwctl:
```
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect SSID
```
### Testing Network Connection:
```
ping archlinux.org
```
The output should contain the following if connected:
```
64 bytes from archlinux.org (95.217.163.246): icmp_seq=2 ttl=49 time=54.1 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=3 ttl=49 time=50.1 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=4 ttl=49 time=50.0 ms
```

 
## Partitioning
Recommended partition scheme:

| Partition     | Size             | Type            |
|:-------------:|:----------------:|:---------------:|
| `/boot`         | 512M           | EFI System      |
| `swap`          | 4GB            | Linux Swap      |
| `/`             | Remaining space| Linux filesystem| 

### Create partitions:
List the disks to find the device name:
```
fdisk -l
```

The output shows the disk name is `/dev/sda`:
```
Disk /dev/sda: 64 GiB, 68719476736 bytes, 134217728 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: A7B0EEB0-2290-4E4B-AD46-F0EFC0B1CBE0
```
Be sure to check the size of the disk to know which drive you want to install the OS on.

Partition the disk using `cfdisk`:
```
cfdisk /dev/sda
```
+ Select [New]
+ Partition size: +512M
+ Select [Type]>EFI System  

Select Free space
New>Partition size: 4G
Select [Type]>Linux Swap


Select Free space>New>Partition size: Remaining space
Select [Type]>Linux filesystem

Select [Write]: Type "yes" to write partitions to disk.
```
## Formatting
```
mkfs.fat -F 32 /dev/sda1 
mkswap /dev/sda2 
swapon /dev/sda2
mkfs.btrfs /dev/sda3
```
### Mounting the file systems

Mount the `root` partition:

```
mount /dev/sda3 /mnt
```
Mount the `/boot` partition:
```
mount /dev/sda1 /mnt/boot
```
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
## Install the base packages
To install the base system packages:
```
pacstrap /mnt base base-devel linux linux-firmware 
```
You may want to install any additional packages within pacstrap such as:
`sudo`, `vim/nano`, `git`, `rsync` and `base-devel`.

**Note: You may wish to substitute `nano` with your text editor of choice e.g `vim`.**
## Configuring the system

### Fstab:
To generate the fstab file which will be used to mount the filesystems:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
Check that the fstab file was generated properly and the entries are correct:
```
cat /mnt/etc/fstab
```
If the fstab file does not contain the entries, regenerate the file.
### Chroot 
To change root into the system:
```
arch-chroot /mnt
```
### Timezone
Set the timezone:
```
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```
**Note: Available timezones can be listed with `timedatectl list-timezones`**.
### HWClock
Set the HWClock:
```
hwclock --systohc
```
This will generate `/etc/adjtime`.
### Localisation
Edit /etc/locale.gen and uncomment `en_US.UTF-8 UTF-8` and any other needed locales. e.g `en_GB.UTF-8 UTF-8`.

Then generate the locales:
```
locale-gen
```
Set the LANG variable:
```
echo LANG=en_GB.UTF-8 > /etc/locale.conf
```
Set the console keyboard layout:
```
echo KEYMAP=uk > /etc/vconsole.conf
```
## Network configuration
Set the hostname e.g `arch`:
```
echo arch > /etc/hostname
```
Configure the `hosts` file:
```
echo 127.0.0.1  localhost >> /etc/hosts
echo ::1        localhost >> /etc/hosts                   
echo 127.0.1.1  localhost.localdomain   arch >> /etc/hosts
```
**Note: Append `arch` with the hostname you have set.**
### Root password
Set the root user password:
```
passwd
```
### Install Bootloader
A bootloader needs to be installed for system initialisation. For this configuration GRUB will be the default option:
```
pacman -S grub efibootmgr
```
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```
Then generate the GRUB confifuration file:
```
grub-mkconfig -o /boot/grub.grub.cfg
```
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
### Network Manager
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
umount -a /mnt
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
### Install Snapper
```
pacman -S snapper
```
## Post Install Configuration
Ensure you are connected to the internet before doing these steps. 
### Sudoers
Edit the sudoers file and uncomment this line:
```
EDITOR=nano visudo
```
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```
### Multilib

Multilib support can be enabled by editing `/etc/pacman.conf` and uncommenting this line:
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```
```
### Update the system
Before installing any other packages, update the system:
```
sudo pacman -Syu
```

### Install AUR Helper
Git clone yay and build the package:
```
git clone https://aur.archlinux.org/yay.git
cd yay/
makepkg -si
```
### Install & configure Snapper wrappers
Install `snapper-gui-git` and `snap-pac-grub` which will allow you to view snapshots on GRUB menu and manage them with a GUI.
```
yay -S snapper-gui-git
yay -S snap-pac-grub
```
### Backup boot partition hook
Create a hook file with the following contents:
```
sudo touch /etc/pacman.d/hooks/50-bootbackup.hook
```
```
[Trigger]
Operation = Upgrade
Operation = Install
Operation = Remove
Type = Path
Target = usr/lib/modules/*/vmlinuz

[Action]
Depends = rsync
Description = Backing up /boot...
When = PostTransaction
Exec = /usr/bin/rsync -a --delete /boot /.bootbackup
```
This will ensure the `boot` partition is snapshotted on kernel upgrade.

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
MODULES=(btrfs amdgpu)
```
Don't forget to regenerate the initramfs:
```
sudo mkinitcpio -p linux
```
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
## Extras

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
sudo ufw limit ssh```
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






