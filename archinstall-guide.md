#Install ArchLinux with btrfs and subvolumes without encryption
#Created by EncryptedCicada on 26 Jun 2021
#Info adapted from Arch Wiki and various other sources

#The default keyboard layout/console keymap is US.
#If you don't want to change the default, skip this step.
#If you want to list all available keymaps do:
ls /usr/share/kbd/keymaps/**/*.map.gz
#To change to another keymap append a corresponding file name to 'loadkeys', omitting path and file extension.
#For example, to set a German keyboard layout:
#loadkeys de-latin1

#List drives
fdisk -l

#Select drive where you want to install the system (/dev/sdX where 'X' can be anything. I have 'nvme0n1' in my case instead of 'sdX')
DRIVE=/dev/nvme0n1

#Wipe disk
sgdisk --zap-all $DRIVE

#Create partitions
sgdisk --clear \
        --new=1:0:+3G --typecode=1:ef00 --change-name=1:EFI \
        --new=2:0:+16GiB --typecode=2:8200 --change-name=2:swap \
        --new=3:0:0 --typecode=3:8300 --change-name=3:system \
          $DRIVE

#View and verify the partion table
lsblk -o +PARTLABEL

#Format EFI Partition
mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI

#Make and enable swap
mkswap -L swap /dev/disk/by-partlabel/swap
swapon -L swap

#Format system partition as btrfs and create subvolumes
mkfs.btrfs --label system /dev/disk/by-partlabel/system

o_btrfs=defaults,ssd,noatime,autodefrag,space_cache=v2,discard=async

mount -t btrfs LABEL=system /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@srv
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@.snapshots

umount -R /mnt

mount -o subvol=@,$o_btrfs LABEL=system /mnt

mkdir /mnt/{boot,home,var,root,srv}
mkdir /mnt/var/{log,tmp,cache,cache/pacman,cache/pacman/pkg}

mount -o subvol=@home,$o_btrfs LABEL=system /mnt/home
mount -o subvol=@srv,$o_btrfs LABEL=system /mnt/srv
mount -o subvol=@root,$o_btrfs LABEL=system /mnt/root
mount -o subvol=@log,$o_btrfs LABEL=system /mnt/var/log
mount -o subvol=@tmp,$o_btrfs LABEL=system /mnt/var/tmp
mount -o subvol=@cache,$o_btrfs LABEL=system /mnt/var/cache
mount -o subvol=@pkg,$o_btrfs LABEL=system /mnt/var/cache/pacman/pkg
mount -o subvol=@.snapshots,$o_btrfs LABEL=system /mnt/.snapshots

#Mount boot partition
mount LABEL=EFI /mnt/boot

#Install base system
pacstrap /mnt base linux linux-firmware linux-headers btrfs-progs wpa_supplicant networkmanager mkinitcpio neovim sudo nano git base-devel

#Generate fstab
genfstab -L -p /mnt >> /mnt/etc/fstab
#Check fstab
cat /mnt/etc/fstab

#Chroot into installed system
arch-chroot /mnt

#THE FOLLOWING COMMANDS ARE RUN IN CHROOT!!!

# Turn on NTP synchronization
timedatectl set-ntp true
# Then list timezones and pick one
timedatectl list-timezones
# Finally set your timezone replacing Asia/Kolkata with what you picked
timedatectl set-timezone Asia/Kolkata
ln -s /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
# Set hardware clock from system clock
hwclock --systohc

#Edit /etc/locale.gen and uncomment en_IN, en_GB.UTF8 and other needed locales
#My locales are India specific and fallback locale is UK Eng, please google your own locales
nano /etc/locale.gen
#Generate the locales by running:
locale-gen

#Create the locale.conf (/etc/locale.conf) file, and set the LANG variable accordingly (Paste the following lines excluding '...' making your own edits):
...
LANG=en_IN.UTF-8
LANGUAGE=en_IN:en_GB:en
LC_TIME=en_IN.UTF-8
...

#Set the keyboard layout for console persistent by editing /etc/vconsole.conf as such by doing:
nano /etc/vconsole.conf
...
KEYMAP=us
...

#Set hostname replacing myhostname for your desired hostname (Hostname is the name of your PC on the network)
hostnamectl set-hostname myhostname
#Set hostname and setup /etc/hosts file
echo myhostname > /etc/hostname
# Edit /etc/hosts file
nano /etc/hosts
# Append the following lines
...
127.0.0.1       localhost
::1             localhost
127.0.1.1       myhostname.localhost  myhostname
...

#Enable network connection
systemctl enable NetworkManager

#Enable ntp client
systemctl enable systemd-timesyncd

#Setup root passwd (Process to disable root password at near the EOF)
#passwd
NOTE: Some people including me can skip this step if they don't want to setup a password for root account.
      Some distros like Ubuntu do this by default to prevent newbies from breaking the system and to prevent root exploits.
      This guide still covers steps to provide elevated privilege to the standard user using sudo.


#Add user (replace username with your username)
USER=username
useradd -m -G wheel $USER
usermod -aG libvirt $USER
passwd $USER

#Edit the sudoers file to give access to added user for elevated previlidges (after uncommenting the following line should look like below)
EDITOR=nano visudo
...
%wheel ALL=(ALL) ALL
...

#Install other necessary packages
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber gnome-firmware bpytop cups \
          ttf-liberation noto-fonts noto-fonts-emoji bash-completion amd-ucode curl wget qt5-wayland \
          qt6-wayland glfw-wayland

#Enable CUPS socket detection
systemctl enable cups.socket

#Install graphics drivers (some drivers are specific to amdgpu)
pacman -S mesa vulkan-radeon vulkan-mesa-layers xf86-video-amdgpu libva-mesa-driver mesa-vdpau

# Login as $USER and install paru
su $USER
cd ~/
mkdir {Downloads,Documents,Videos,Pictures,Music,Applications}
cd Downloads
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin
makepkg -si
# After installation successful
paru --gendb
cd ~/Downloads
rm -rf paru-bin
#Use paru to install essential packages from AUR
paru -S android-tools asp bluez-utils bluez brave-bin ccache libvirt edk2-omvf dnsmasq iptables-nft virt-manager \
        flatpak flatseal gdb glib goverlay-bin heroic-games-launcher-bin hunspell-en_gb libreoffice-fresh hyphen-en \
        wireless-regdb jdk-openjdk man-db mangohud-git meld micro mpd neofetch pfetch nerd-fonts-cascadia-code \
        nerd-fonts-fira-code nerd-fonts-jetbrains-mono nerd-fonts-sf-mono obs-studio papirus-icon-theme pdfarranger \
        proton-ge-custom-bin protontricks qt5ct reflector switcheroo-control teams-insiders tangram timeshift timeshift-autosnap \
        uget visual-studio-code-bin wine-stable wine-gecko wine-mono bottles gamemode firewalld
#If Gnome is to be installed
paru -S clapper chrome-gnome-shell extension-manager fractal fragments gdm-settings-git dynamic-wallpaper \
        gnome-software-packagekit-plugin gnome-text-editor networkmanager-openvpn

#Packages specific to my laptop
paru -S dell-g5se-fanctl amd-ucode
sudo systemctl enable dell-g5se-fanctl-sleep.service

#Exit $USER
exit

#OPTIONAL(If you're installing a full-blown desktop environment like Gnome, sometimes GDM doesn't start automatically. For amdgpu users this is the fix)
nano /etc/mkinitcpio.conf
#Edit the line that says MODULES=() and add the following in the beginning just after the bracket:
amdgpu
#The edited line shound look somethng like this:
MODULES=(amdgpu ...)

#Edit the mkinitcpio.conf file like earlier and add make sure it looks like the following
...
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block btrfs filesystems fsck)
...
#Generate
mkinitcpio -P

#Installing systemd-boot bootloader
bootctl --path=/boot install

#Configuring loader
#TIP:A basic loader configuration file is located at /usr/share/systemd/bootctl/loader.conf
nano /boot/loader/loader.conf
# Add the following lines to the file
...
default archlinux.conf
editor no
auto-entries 1
auto-firmware 1
console-mode max
...

#Adding loaders
#TIP:An example entry file is located at /usr/share/systemd/bootctl/arch.conf
cd /boot/loader/entries
#This will create 2 files: archlinux.conf and archlinux-fallback.conf
touch archlinux.conf archlinux-fallback.conf
nano archlinux.conf
# Add the following lines to the file
...
# Created by EncryptedCicada
title Arch Linux
linux /vmlinuz-linux
initrd /amd-ucode.img
initrd /initramfs-linux.img
options root="LABEL=system" rw rootflags=subvol=/@ quiet splash loglevel=3 rd.systemd.show_status=auto vt.global_cursor_default=0
...

nano archlinux-fallback.conf
# Add the following lines to the file
...
# Created by EncryptedCicada
title Arch Linux (fallback initramfs)
linux /vmlinuz-linux
initrd /amd-ucode.img
initrd /initramfs-linux-fallback.img
options root="LABEL=system" rw rootflags=subvol=/@ quiet splash loglevel=3 rd.systemd.show_status=auto vt.global_cursor_default=0
...

#For quiet boot with other bootloaders add the following to the options line in the loader after 'rw'
#quiet splash loglevel=3 rd.systemd.show_status=auto vt.global_cursor_default=0

#To enable automatic updates of systemd-boot bootloader
systemctl enable systemd-boot-update.service

#Enable multilib repo
nano /etc/pacman.conf
#Look for file that says '#[multilib]'
#Uncomment the line and the Server address that follows. It should look like:
...
[multilib]
Include = /etc/pacman.d/mirrorlist
...

Warning: Users need to be certain that their SSD supports TRIM before attempting to use it. Data loss can occur otherwise!
#To verify TRIM support, run:
lsblk --discard
#Check the values of DISC-GRAN (discard granularity) and DISC-MAX (discard max bytes) columns. Non-zero values indicate TRIM support.
#If your disk has trim support then enable fstrim timer
systemctl enable fstrim.timer

#Enable bluetooth, ssh, firewall and libvirt services
systemctl enable bluetooth
systemctl enable sshd
systemctl enable firewalld
systemctl enable libvirtd

#Install WM/DM of choice. Example to install Gnome:
pacman -S gnome
#If you have installed gnome or any other desktop environment do (replace 'gdm' with your desktop specific login manager service):
systemctl enable gdm
#Gnome specific package:
pacman -S geary gnome-connections gnome-mines gnome-sound-recorder gnome-tweaks gnome-usage power-profiles-daemon

#To exit chroot, type: (optionally do ctrl+d)
exit

#Unmount all partitions with
umount -R /mnt

#To reboot into the newly installed system type:
reboot
#Remember to remove the installation medium and then login into the new system with the root account.

#For people who wish to disable password for root account just follow this step:
#sudo usermod -p '!' root
#this sets root to have a disabled password

#Go over to the post-install script