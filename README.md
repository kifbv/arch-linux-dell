Notes to myself on how to install Arch Linux.

Instructions below mostly from the [Arch Linux Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide/).

Tested on Dell XPS13 (9360).

## Preparation
Prepare a bootable USB stick with the Arch image on it:

1. download the latest Arch Linux ISO image and its signature from the [Arch Linux Downloads](https://archlinux.org/download/) page
1. verify the image integrity: `gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig`
1. format a USB stick (FAT format, MBR partition table) or use something like [ventoy](https://www.ventoy.net/en/index.html)
  - formatted stick: copy the ISO image to the stick: `cp path/to/archlinux.iso /dev/disk/by-id/usb-xxxx`
  - ventoy: copy the ISO image to the stick: `cp path/to/archlinux.iso path/to/usb/stick`
1. reboot the computer, enter the BIOS, and disable 'safe boot' (if on Linux with systemd: run `systemctl reboot --firmware-setup` should present you with the boot menu directly)
1. reboot and select Arch Linux in the boot menu (Dell XPS13: press F12 to get the boot menu)

## Pre-installation
Note: in case of SQUASHFS errors during installation, it probably means that the USB device is not good so try another one.

### Set the console keyboard layout
The default is `us` but for e.g. a `uk` layout:
1. list the available layouts: 

    `localectl list-keymaps` or `ls /usr/share/kbd/keymaps/**/*.map.gz`

2. load the uk layout:

    `loadkeys uk`

~Load a more relevant font: `setfont eurlatgr`~


### Connect to the internet
~Run `wifi-menu`, or refer to [Network configuration](https://wiki.archlinux.org/index.php/Network_configuration) or [Wireless network configuration](https://wiki.archlinux.org/index.php/Wireless_network_configuration) if that doesn't work.~
add /etc/iwd/main.conf with the following for DHCP:
[network]
EnableNetworkConfiguration=true
ip link to check the network interface is listed and up
iwctl to get an interactive prompt; then device list; turn on if necessary with device <device> set-property Powered on (same with adapter if necessary); scan with station <device> scan; list available networks with station <device> get-networks; connect with station <device> connect <SSID>
ping -4 archlinux.org to verify connection ok


### Update the system clock
`timedatectl set-ntp true`

### Optimize logical sector size (for NVMe drives)


### Partition the disk
Check the disk with `lsblk` or `fdisk -l` and take a note of the device name (`/dev/nvme0n1` on the Dell XPS13)

1. Use the GPT disklabel:

   `parted -a optimal /dev/nvme0n1 mklabel gpt`

1. Create a small boot partition:

   `parted -a optimal -- /dev/nvme0n1 mkpart ESP fat32 1MiB 513MiB set 1 boot on`

1. Create a system partition using the rest of the disk:

   `parted -a optimal -- /dev/nvme0n1 mkpart ext4 513MiB 100%`

Persistent block device naming for fstab is achieved by using UUID of the disk.

### Encrypt the Linux partition
Use [LVM on LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS).

1. Create the encrypted container on the **system** partition (choose a good passphrase):

   `cryptsetup luksFormat /dev/nvme0n1p2`

1. Open the container (name it 'luks'):

   `cryptsetup open /dev/nvme0n1p2 luks`

1. Create a Physical Volume on top of the opened LUKS container:

   `pvcreate /dev/mapper/luks`

1. Create a Volume Group and add the Physical Volume to it (name it `main`):

   `vgcreate main /dev/mapper/luks`

1. Create Logical Volumes on that Volume Group (50GiB for `/` and 100GiB for `/home`, no swap YOLO):

   `lvcreate -L50g main -n root`

   `lvcreate -L100g main --name home`

### Format the filesystem on each Logical Volume:
See https://wiki.archlinux.org/index.php/Ext4#Enabling_metadata_checksums and maybe http://www.askapache.com/optimize/super-speed-secrets/
   `modprobe crc32c_intel`
   `mkfs.ext4 -O metadata_csum,64bit /dev/mapper/main-root`
   `mkfs.ext4 -O metadata_csum,64bit /dev/mapper/main-home`

### Mount the file systems
1. mount `/`:

   `mount -o defaults,journal_checksum /dev/mapper/main-root /mnt`

1. create mountpoints for `/boot` and `/home`:

   `mkdir /mnt/{boot,home}`

1. mount `/boot`:

   `mount -o defaults,journal_checksum /dev/nvme0n1p1 /mnt/boot`

1. mount `/home`:

   `mount -o defaults,journal_checksum /dev/mapper/main-home /mnt/home`


## Installation
### Install the base packages
Install the [base](https://www.archlinux.org/groups/x86_64/base/) and [base-devel](https://www.archlinux.org/groups/x86_64/base-devel/) package groups as well as some packages for wireless network configuration, Intel micro-code updates and git:

   `pacstrap /mnt base base-devel dialog dhcpcd wpa_actiond wpa_supplicant intel-ucode git`
   `pacstrap /mnt base base-devel linux linux-firmware intel-ucode iwd lvm2`
 sway foot waybar wofi vim brave
## Configure the system
### Fstab
Create the `fstab` file:

   `genfstab -U /mnt >> /mnt/etc/fstab`

### Change root into the target system to continue the installation:
`arch-chroot /mnt`

### Time settings
Set the timezone:

   `ln -fs /usr/share/zoneinfo/Europe/London /etc/localtime`

Set the hardware clock (from the system clock but keep it UTC):

   `hwclock --systohc --utc`

### Locale
Uncomment required localisations in `/etc/locale.gen` and regenerate them with `locale-gen`.

Set the `LANG` variable in `/etc/locale.conf` e.g. `LANG=en_GB.UTF-8`

Make console keyboard mappings and font changes permanent `/etc/vconsole.conf`:

```Shell
FONT=eurlatgr
KEYMAP=us
```

### Hostname
1. Create the hostname file:

   `echo <hostname> > /etc/hostname`

1. Add a matching entry to `/etc/hosts`:

   `echo 127.0.1.1 <hostname>.localdomain <hostname> >> /etc/hosts`

### Network configuration
1. Configure wireless with `wifi-menu`.
1. Configure wireless to start at boot and automatically connect to known networks:

   `systemctl enable netctl-auto@<interface_name>.service`

(use tab completion to figure out the interface name).

### Initramfs
Edit `/etc/mkinitcpio.conf` with:

  - `MODULES="EXT4"`
  - `HOOKS="autodetect systemd block sd-vconsole sd-encrypt sd-lvm2 fsck keyboard"`

Run `mkinitcpio -p linux` to generate the initramfs image as `/boot/initramfs-linux.img`

(see [mkinitcpio#hooks](https://wiki.archlinux.org/index.php/Mkinitcpio#HOOKS))

Setup systemd-boot:

   `bootctl --path=/boot install`

### Bootloader
Create a bootloader entry:

```Shell
UUID=$(cryptsetup luksUUID /dev/nvmen0n1p2)
cat <<EOF>/boot/loader/entries/arch.conf
title     Arch Linux
linux     /vmlinuz-linux
initrd    /intel-ucode.img
initrd    /initramfs-linux.img
options   luks.uuid=$UUID luks.name=$UUID=luks root=/dev/mapper/main-root rw
EOF
```

Point the bootloader to the default entry (see [Basic Configuration](https://wiki.archlinux.org/index.php/systemd-boot#Basic_configuration)):

```Shell
cat <<EOF>/boot/loader/loader.conf
default  arch
editor   0
EOF
```

### Users
Set the root password with `passwd`

Add a regular user:

```Shell
useradd -m -g users -G wheel -s /bin/bash franck
passwd franck
```

Run `visudo` and add the following to have my user in the sudoers file :

```Shell
# options
Defaults editor=/usr/bin/vim, !env_editor
Defaults insults

# full access
franck ALL=(ALL) ALL

# last rule as a safety guard
franck ALL=/usr/sbin/visudo
```
To add a red blinking prompt for root:  
First, copy some skeleton files:  
`cp /etc/skel/.bash_profile /etc/skel/.bashrc /root`  
And then edit `/root/.bashrc` and set `PS1` to:  
`PS1="\e[1m\e[91m\e[5m[\u@\h]\$\e[m\e(b "`

## Reboot

   `exit`

   `reboot`

## System configuration

Add the following repo to `/etc/pacman.conf` for easy installation of `vivaldi`:
```Shell
[herecura]
Server = https://repo.herecura.be/herecura/x86_64
```
ensure that the following options are set:
```Shell
HoldPkg = pacman glibc
Architecture = auto
Color
CheckSpace
VerbosePkgLists
SigLevel = Required DatabaseOptional
LocalFileSigLevel = Optional
```
and upgrade the system with `sudo pacman -Syu` in order to be able to use it.

add tuning instructions e.g. build in tmpfs (https://wiki.archlinux.org/index.php/Makepkg#Improving_compile_times)

### Packages
Install the following package groups:
`i3 xorg xorg-apps xorg-drivers`

Install the packages in `packages.lst` with pacman and pacaur.

### Enable some services
- systemctl enable --now systemd-timesyncd.service# simple SNTP daemon
- systemctl enable --now docker.service
- cp /etc/iptables/empty.rules /etc/iptables/iptables.rules && \
..   systemctl enable --now iptables.service # safety first
(and see [Simple Statefull Firewall](https://wiki.archlinux.org/index.php/Simple_stateful_firewall) for configuration)

### User configuration
Clone all the config files from [github/kifbv/dotfiles](https://github.com/kifbv/dotfiles) and install them with stow

Plus:
- wayland support, prompt (see [here](https://github.com/sapegin/dotfiles/blob/dd063f9c30de7d2234e8accdb5272a5cc0a3388b/includes/bash_prompt.bash) and [here](https://code.mendhak.com/simple-bash-prompt-for-developers-ps1-git/) and [here](https://github.com/sindresorhus/pure) too.
- install [fonts](https://www.programmingfonts.org/#hack)
- finish qmk config

### TODO
Also see https://wiki.archlinux.org/index.php/Sysctl and https://wiki.archlinux.org/index.php/Improving_performance
Fix the layout to be the same as Macbook UK (i.e. use us layout but remap the |\ key next to shift-left to ~`)
Relocate files to tmpfs (+ other tips in https://wiki.archlinux.org/index.php/Improving_performance)

## Arch Linux Docs:
[Installation guide](wiki.archlinux.org/index.php/Installation_guide)
[Improving performance](https://wiki.archlinux.org/index.php/Improving_performance)
[Keyboard shortcuts](https://wiki.archlinux.org/index.php/Keyboard_shortcuts#Virtual_console)
[Keyboard configuration in Xorg](https://wiki.archlinux.org/index.php/Keyboard_configuration_in_Xorg)
[Map scancodes to keycodes](https://wiki.archlinux.org/index.php/Map_scancodes_to_keycodes)
[Extra keyboard keys in Xorg](https://wiki.archlinux.org/index.php/Extra_keyboard_keys_in_Xorg#Mapping_keysyms_to_actions)
[Xbindkeys](https://wiki.archlinux.org/index.php/Xbindkeys)
[LVM](https://wiki.archlinux.org/index.php/LVM)
[Mkinitcpio](https://wiki.archlinux.org/index.php/Mkinitcpio#HOOKS)
[Disk encryption](https://wiki.archlinux.org/index.php/Disk_encryption)
[Ext4](https://wiki.archlinux.org/index.php/Ext4#Enabling_metadata_checksums)
[Wireless Network Configuration](https://wiki.archlinux.org/index.php/Wireless_network_configuration)

Broadcom driver for the MacBook
[](https://wireless.wiki.kernel.org/en/users/Drivers/b43#devicefirmware)
[](http://www.lwfinger.com/b43-firmware/)

