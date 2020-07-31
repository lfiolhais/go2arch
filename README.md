# Configure Install Environment

Set keyboard layout: `loadkeys i386/qwerty/uk`

Reorder mirror list. Put your country's mirrors first: `vim /etc/pacman.d/mirrorlist`

If you use WiFi, set it up. Setup WiFi: `wifi-menu`

If you are in a HiDPI display increase the font size: `setfont latarcyrheb-sun32`

# Drive Partitioning

| Number |  Start  |   End   |  Size  | File System | Name |   Flags   |
|:------:|:-------:|:-------:|:------:|:-----------:|:----:|:---------:|
|    1   |  1049KB |  538MB  |  537MB |    fat32    | boot | boot, esp |
|    2   |  538MB  |  8730MB | 8192MB |  linux-swap | swap |           |
|    3   |  8730MB | 10778MB |   2GB  |     ext4    |  sos |           |
|    4   | 10778MB |   100%  |        |     ext4    | root |           |

```
# parted
    mktable gpt
    mkpart ESP fat32 1049kB 538MB
    set 1 boot on
    name 1 boot
    mkpart primary linux-swap 538MB 8730MB
    name 2 swap
    mkpart primary ext4 8730MB 10778MB
    name 3 sos
    mkpart primary ext4 10778MB 100%
    name 4 root
# mkfs.vfat -F32 /dev/sda1
# mkswap /dev/sda2
# mkfs.ext4 /dev/sda3
# mkfs.ext4 /dev/sda4
```

# Configuring SOS system

## Mounting partitions

```
mount /dev/sda3 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
```

## Installing base system

```
# pacstrap /mnt base linux networkmanager broadcom-wl wpa_supplicant dialog vim dhcpcd
# genfstab -U -p /mnt >> /mnt/etc/fstab
# vim /mnt/etc/fstab
```

Make sure that the line of the ext4 partition ends with a “2”, the swap partition’s line ends
with a “0”, and the boot partition’s line ends with a “1”. This configures the partition checking on
boot.

If you have an SSD (which the mac does), you can edit the ext4 partition options to look
like this: `rw,relatime,data=ordered,discard`

## Configuring base system

```
# arch-chroot /mnt
chroot# vim /etc/locale.gen # (uncomment en_GB.UTF-8 UTF-8)
chroot# locale-gen
chroot# echo LANG=en_GB.UTF-8 > /etc/locale.conf
chroot# export LANG=en_GB.UTF-8
chroot# vim /etc/vconsole.conf
    KEYMAP=uk
    FONT=latarcyrheb-sun32
chroot# rm /etc/localtime
chroot# ln -s /usr/share/zoneinfo/Europe/Lisbon /etc/localtime
chroot# hwclock --systohc --utc
chroot# /etc/modprobe.d/broadcom-wl.conf # (Delete line: brcmfmac)
chroot# vim /etc/modules-load.d/broadcom-wl.conf
    wl
    brcmfmac
chroot# vim /etc/modules-load.d/applemac.conf
    coretemp
    applesmc
chroot# echo HOSTNAME > /etc/hostname
chroot# vim /etc/hosts
    127.0.0.1 localhost.localdomain localhost HOSTNAME-SOS
    ::1 localhost.localdomain localhost HOSTNAME-SOS
chroot# passwd # (Set root password)
chroot# pacman -S dosfstools
chroot# bootctl --path=/boot install
chroot# vim /etc/mkinitcpio.d/linux.preset
    Delete the lines:
    default_image=”/boot/initramfs-linux.img”
    fallback_image=”/boot/initramfs-linux-fallback.img”
    Replace with:
    default_image=”/boot/initramfs-sos.img”
    fallback_image=”/boot/initramfs-sos-fallback.img”
chroot# mkinitcpio -p linux
chroot# vim /boot/loader/entries/sos.conf
    title Arch Linux SOS
    linux /vmlinuz-linux
    initrd /initramfs-sos.img
    options root=/dev/sda3 rw elevator=deadline quiet splash resume=/dev/sda2 nmi_watchdog=0
chroot# exit
# umount /mnt/boot
# umount /mnt
```

# Configuring main system

## Mounting partitions

```
mount /dev/sda4 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
```

## Installing base system

```
# pacstrap /mnt base base-devel broadcom-wl wpa_supplicant dialog vim dhcpcd linux linux-firmware awesome lightdm-gtk-greeter lightdm
# genfstab -U -p /mnt >> /mnt/etc/fstab
# vim /mnt/etc/fstab
```

Make sure that the line of the ext4 partition ends with a “2”, the swap partition’s line ends
with a “0”, and the boot partition’s line ends with a “1”. This configures the partition checking on
boot.

If you have an SSD (which the mac does), you can edit the ext4 partition options to look
like this: rw,relatime,data=ordered,discard

## Configuring base system

```
# arch-chroot /mnt
chroot# vim /etc/locale.gen # (uncomment en_GB.UTF-8 UTF-8)
chroot# locale-gen
chroot# echo LANG=en_GB.UTF-8 > /etc/locale.conf
chroot# export LANG=en_GB.UTF-8
chroot# vim /etc/vconsole.conf
    KEYMAP=uk
    FONT=latarcyrheb-sun32
chroot# rm /etc/localtime
chroot# ln -s /usr/share/zoneinfo/Europe/Lisbon /etc/localtime
chroot# hwclock --systohc --utc
chroot# /etc/modprobe.d/broadcom-wl.conf # (Delete line: brcmfmac)
chroot# vim /etc/modules-load.d/broadcom-wl.conf
    wl
    brcmfmac
chroot# vim /etc/modules-load.d/applemac.conf
    coretemp
    applesmc
chroot# echo HOSTNAME > /etc/hostname
chroot# vim /etc/hosts
    127.0.0.1 localhost.localdomain localhost HOSTNAME
    ::1 localhost.localdomain localhost HOSTNAME
chroot# passwd
    Set root password
chroot# vim /boot/loader/entries/arch.conf
    title Arch Linux
    linux /vmlinuz-linux
    initrd /initramfs-linux.img
    options root=/dev/sda4 rw elevator=deadline quiet splash resume=/dev/sda2 nmi_watchdog=0
chroot# echo “default arch” > /boot/loader/loader.conf
# exit
# umount /mnt/boot
# umount /mnt
# reboot
```

## Configuring Basic User

### Basic user management

```
# useradd -m -g users -G wheel -s /bin/bash USERNAME
# pacman -S sudo
# visudo
    Uncomment %wheel ALL=(ALL) ALL
# passwd USERNAME
# passwd -l root
# reboot
```

### Graphical interface

```
$ sudo pacman -S xorg-server xorg-randr xf86-video-intel
$ sudo vim /etc/lightdm.conf
    Replace
    #greeter-session=example-gtk-lightdm
    with
    greeter-session=lightdm-gtk-greeter
$ sudo systemctl enable lightdm.service
$ sudo pacman -S awesome
$ sudo pacman -S network-manager-applet cbatticon rxvt-unicode
$ vim .Xresources
    Xft.dpi: 144
    Xft.autohint: 0
    Xft.lcdfilter: lcddefault
    Xft.hintstyle: hintfull
    Xft.hinting: 1
    Xft.antialias: 1
    Xft.rgba: rgb
    URxvt*.depth: 32
    URxvt*font: xft:DejaVu Sans Mono:size=12
    URxvt*scrollBar: false
    ! special
    ! --- ~/.Xresources
    ------------------------------------------------------------
    !
    ------------------------------------------------------------------------------
    ! --- generated with 4bit Terminal Color Scheme Designer
    -----------------------
    !
    ------------------------------------------------------------------------------
    ! --- http://ciembor.github.com/4bit
    -------------------------------------------
    !
    ------------------------------------------------------------------------------! --- special colors ---
    URxvt*background: #0b1d2e
    URxvt*foreground: #d9e6f2
    ! --- standard colors ---
    ! black
    URxvt*color0: #0b1d2e
    ! bright_black
    URxvt*color8: #2e4051
    ! red
    URxvt*color1: #da49ab
    ! bright_red
    URxvt*color9: #da9ed5
    ! green
    URxvt*color2: #88ec5a
    ! bright_green
    URxvt*color10: #b3ecaf
    ! yellow
    URxvt*color3: #da9a5a
    ! bright_yellow
    URxvt*color11: #dac5af
    ! blue
    URxvt*color4: #379afc
    ! bright_blue
    URxvt*color12: #8cc5fc
    ! magenta
    URxvt*color5: #8849fc
    ! bright_magenta
    URxvt*color13: #b39efc! cyan
    URxvt*color6: #37ecab
    ! bright_cyan
    URxvt*color14: #8cecd5
    ! white
    URxvt*color7: #baccdd
    ! bright_white
    URxvt*color15: #daecfc
    !
    ------------------------------------------------------------------------------
    ! --- end of terminal colors section
    -------------------------------------------
    !
    ------------------------------------------------------------------------------
$ vim ~/.profile
    xrdb -merge ~/.Xresources
    nm-applet &
    cbatticon &
$ mkdir -p ~/.config/awesome
$ cp /etc/xdg/awesome/rc.lua ~/.config/awesome/
$ vim ~/.config/awesome/rc.lua
    Swap
    terminal = “xterm”
    for
    terminal = “urxvt”
```
