
# 1. EFI, Network, Timezone

## 1) Verify that you're in EFI
`efivar -l`  
if a bunch of files appear, you're booted into **UEFI**. If no files show, you're booted into **BIOS**.

## 2) Connect to the internet in live installation
use `iwctl` ([more info](https://wiki.archlinux.org/title/Iwd#iwctl))  
```
[iwd]# device list
[iwd]# station name scan
[iwd]# station name get-networks
[iwd]# station name connect [SSID]
[iwd]# Passphrase: [password]
```
check via `ping archlinux.org`

## 3) System clock
`timedatectl set-ntp true`  
`timedatectl status`  
IF timedatectl is still finicky, try `timedatectl set-timezone {COUNTRY}/{CITY}`

# 2. Disk partitioning & Set-up

## 1) Check drive
`lsblk` [See what drive you're going to install Arch Linux to]

## 2) Partition disks
**NOTE**: /dev/sda will be used as the example.  
`gdisk /dev/sda`  
```
x   expert mode  
z   zap drive  
y   yes  
y   yes  
```
`cgdisk /dev/sda`  

*If warned about damaged GPT when entering cgdisk, just ignore.*  
```
- EFI Partition -
SIZE = 512MiB
DISK = /dev/sda1
GUID = efi
NAME = boot
- SWAP Partition -
SIZE = 33 GiB
DISK = /dev/sda2
GUID = swap
NAME = swap
- ROOT Partition -
SIZE = 35 GiB
DISK = /dev/sda3
GUID = linux_fs
NAME = root
- HOME Partition -
SIZE = Remainder
DISK = /dev/sda4
GUID = linux_fs
NAME = home

[ Write ]
Yes
[ Quit ]
```
## 3) Formatting partitions
`mkfs.fat -F32 /dev/sda1`  
`mkswap /dev/sda2`  
`swapon /dev/sda2`  
`mkfs.ext4 /dev/sda3`  
`mkfs.ext4 /dev/sda4`  

## 4) Mounting partitions
`mkdir /mnt/boot`  
`mkdir /mnt/home`  
`mount /dev/sda1 /mnt/boot`  
`mount /dev/sda3 /mnt`  
`mount /dev/sda4 /mnt/home`  

# 3. Install Linux
`cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`  [backup Mirrorlist]  
`reflector --country {COUNTRY} --protocol https --latest 5 --sort rate --save /etc/pacman.d/mirrorlist`  

`pacstrap -K /mnt base base-devel linux-headers linux linux-firmware networkmanager bash-completion nano linux-lts linux-lts-headers` [Install base linux kernel + LTS in addition + other needed packages]

# 4. Configure Linux & chroot
`genfstab -U /mnt >> /mnt/etc/fstab`  
`arch-chroot /mnt /bin/bash`  
`ln -sf /usr/share/zoneinfo/{COUNTRY}/{CITY} /etc/localtime`  
`hwclock --systohc`  
`nano /etc/locale.gen`  
```
en_{COUNTRY_CODE}.UTF-8 UTF-8  
```
`locale-gen`  
`locale > /etc/locale.conf`  
`echo "{HOSTNAME}" > /etc/hostname`
`nano /etc/hosts`  
```
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    {HOSTNAME}.localdomain	  {HOSTNAME}
```

# 5. User
`passwd` [set root password]  
`useradd -m -g users -G wheel,storage,power {USERNAME}`  
`passwd {USERNAME}` [set password]  
`EDITOR=nano visudo`  
```
%wheel ALL=(ALL) ALL  
Defaults rootpw         {EOF}
```  

# 6. pacman stuff, apps
`systemctl enable fstrim.timer`  

`nano /etc/pacman.conf`  
```
enable [multilib]  
Color  
VerbosePkgLists  
ParallelDownloads = 5  
ILoveCandy
```
`pacman -Sy`  
`pacman -S amd-ucode`

# 7. bootloader & network
`mount -t efivarfs efivarfs /sys/firmware/efi/efivars/`  
`bootctl install`  
`nano /boot/loader/entries/default.conf`  
```
title add_title_here  
linux /vmlinuz-linux  
initrd /amd-ucode.img  
initrd /initramfs-linux.img  
options root=PARTUUID=?????? rw nvidia_drm.modeset=1
```
`systemctl enable NetworkManager.service`

# 8. NVIDIA drivers

`pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings`  
`nano /etc/mkinitcpio.conf` 
```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```
`mkdir /etc/pacman.d/hooks`  
`nano /etc/pacman.d/hooks/nvidia`  
```
[Trigger]  
Operation=Install  
Operation=Upgrade  
Operation=Remove  
Type=Package  
Target=nvidia  
  
[Action]  
Depends=mkinitcpio  
When=PostTransaction  
Exec=/usr/bin/mkinitcpio -P  
```
`mkinitcpio -P`
# 9. Finalization
`exit`  
`reboot`  

# 10. Post Install

`systemctl --failed` [Check if any systemd services have failed on boot] 

## 1) Enable and configure GNOME 
`sudo pacman -S gnome gnome-tweaks`  
`systemctl enable gdm.service`  

## 2) Download other packages
`sudo pacman -S neofetch bashtop gimp cmatrix libreoffice-fresh celluloid xdg-user-dirs p7zip rhythmbox gnome-keyring gnome-terminal adobe-source-han-sans-jp-fonts adobe-source-han-sans-cn-fonts adobe-source-han-sans-kr-fonts nano-syntax-highlighting pcsx2 git flatpak firefox bluez bluez-utils ttf-ibm-plex ttf-iosevka-nerd zsh zsh-completion fragments`  
`systemctl enable bluetooth.service`  

## 3) AUR and Flatpak setup 
`git clone https://aur.archlinux.org/packages/paru`  
`cd yay`  
`makepkg`  
`paru -S betterdiscordctl visual-studio-code-bin vmware-workstation`  
`flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo`  
`flatpak install flathub com.discordapp.Discord`  
`flatpak install flathub com.valvesoftware.Steam`  
`flatpak install flathub com.github.tchx84.Flatseal`  
`reboot`

