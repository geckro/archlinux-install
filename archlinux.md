# 1. EFI, Network, Timezone
## 1) Verify that you're in EFI  
`ls /sys/firmware/efi/efivars`  
## 2) Connect to the internet in live installation  
use `iwctl` ([more info](https://wiki.archlinux.org/title/Iwd#iwctl))  
check via `ping archlinux.org`  
## 3) System clock
`timedatectl set-ntp true`

# 2. Disk partitioning & Set-up
## 1) Check drive
`lsblk`
## 2) Partition disks
`gdisk /dev/DRIVE` x z y y  
`cgdisk /dev/DRIVE`
- EFI Partition should be 512MiB /dev/*1
   - EF00 boot
- Swap should be 12GiB /dev/*2
  - 8200 swap
- Remainder is the root /dev/*3
  - 8300 root
## 3) Formatting partitions
`mkfs.fat -F 32 /dev/*1`  
`mkswap /dev/*2`  
`mkfs.ext4 /dev/*3`
## 4) Mounting partitions
`mkdir /mnt/boot`  
`mount /dev/*1 /mnt/boot`  
`swapon /dev/*2`  
`mount /dev/*3 /mnt`

# 3. Install Linux
backup mirrorlist `cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`  
`pacman -Sy` & `pacman -S pacman-contrib`  
`rankmirrors -n 5 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist`  
Initial packages: `pacstrap -i /mnt base base-devel linux-headers linux linux-firmware networkmanager`

# 4. Configure Linux & chroot
`genfstab -U /mnt >> /mnt/etc/fstab`  
Chroot into system: `arch-chroot /mnt`
`pacman -S sudo bash-completion nano`  
`ln -sf /usr/share/zoneinfo/Australia/Perth /etc/localtime`  
`hwclock --systohc --utc`  
`nano /etc/locale.gen` uncomment `en_US.UTF-8 UTF-8`  
`locale-gen`  
`echo LANG=en_US.UTF-8 > /etc/locale.conf`  
`export LANG=en_US.UTF-8`    
`nano /etc/hostname` archlinux  

# 5. User
`passwd` set root password
`useradd -m -g users -G wheel,storage,power -s /bin/bash db`  
`passwd db` set password  
`EDITOR=nano visudo` uncomment %wheel ALL=(ALL) ALL  
add "Defaults rootpw" to end of file  

# 6. pacman stuff, apps
`systemctl enable fstrim.timer`  
`nano /etc/pacman.conf` enable [multilib]  
`pacman -Sy`  
`pacman -S amd-ucode`

# 7. bootloader & network
`mount -t efivarfs efivarfs /sys/firmware/efi/efivars/` checks if mount is mounted efi mount efi mounted mount efi  
`bootctl install`  
`nano /boot/loader/entries/default.conf`  
title Arch Linux  
linux /vmlinuz-linux  
initrd /amd-ucode.img  
initrd /initramfs-linux.img  
`echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/*3) rw" >> /boot/loader/entries/default.conf`  
`pacman -S dhcpcd bluez`  
`systemctl enable dhcpcd@wlan0.service`  
`systemctl enable NetworkManager.service`

# 8. haha nvidia lmaoooo

`pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings`  
`nano /etc/mkinitcpio.conf`  MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)  
`nano /boot/loader/entries/default.conf` options root=***** rw [TYPE THIS >>>] nvidia-drm.modeset=1  
`mkdir /etc/pacman.d/hooks`  
`nano /etc/pacman.d/hooks/nvidia`  
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

# 9. reboot
`exit`  
`reboot`  
  
`systemctl --failed` (FIX STUFF ig)  
`sudo pacman -S mesa xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm`  
`sudo pacman -S plasma sddm`  
`sudo systemctl enable sddm.service`  
`sudo pacman -S  konsole chromium keepassxc discord steam dolphin yakuake ark transmission-qt neofetch bashtop gimp cmatrix libreoffice-still kate gwenview mpv xdg-user-dirs wine winetricks kcalc spectacle p7zip adobe-source-han-sans-jp-fonts adobe-source-han-serif-jp-fonts jre-openjdk nano-syntax-highlighting noto-fonts-emoji pcsx2`  
`reboot`

# 10. post setup
Download yay snapshot from https://aur.archlinux.org/packages/yay/  
`tar -xzvf yay.tar.gz`  
`cd yay`  
`makepkg -csi`  
`yay -S vscodium-bin multimc5`