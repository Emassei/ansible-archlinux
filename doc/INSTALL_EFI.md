# Arch installation guide covering the following topics
* Full disk encryption including /boot
* EFI boot and GRUB
* LVM on LUKS
* Minimal system configuration including intel-ucode update

## Table of contents
1. Create bootable install medium
2. Create disk layout
3. Install base system & minimal configuration
4. Install and configure bootloader

### Disk partition layout:
```
+---------------+----------------+----------------+----------------+
|ESP partition: |Boot partition: |Volume 1:       |Volume 2:       |
|               |                |                |                |
|/boot/efi      |/boot           |root            |swap            |
|               |                |                |                |
|               |                |/dev/vg0/root   |/dev/vg0/swap   |
|/dev/sda1      |/dev/sda2       +----------------+----------------+
|unencrypted    |LUKS encrypted  |/dev/sda3 encrypted LVM on LUKS  |
+---------------+----------------+---------------------------------+
```
## 1. Create a bootable install medium

Get the latest iso and checksums from a fast mirror. 
```bash
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/archlinux-$(date +%Y.%m.%d)-x86_64.iso archlinux.iso
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/md5sums.txt
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/sha1sums.txt
```

Validate the downloads.
```bash
$ md5sum --check md5sums.txt
$ sha1sum --check sha1sums.txt
```

Create a bootable usb flash drive, make sure /dev/sdX corresponds to the usb drive.
```bash
$ dd if=archlinux.iso of=/dev/sdX bs=1M status=progress && sync
```

Boot and check your internet connection, fix if necessary.
```bash
$ ping google.com
```

Enable network time synchronization.
```bash
$ timedatectl set-ntp true
```

Check if the time got synchronized.
```bash
$ timedatectl status
```

## 2. Create disk layout
Create partitions according to the partitioning scheme above. Use a gpt
partition table.
```bash
$ gdisk /dev/sda
```

The partition table should look like the following example.
```
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI System
   2         1050624         5244927   2.0 GiB     8300  Linux filesystem
   3         5244928       976773133   463.3 GiB   8E00  Linux LVM
```

Create a filesystem on the EFI partition.
```bash
$ mkfs.fat -F32 /dev/sda1
```

Create an encrypted /boot container and format it. For convenience we will set a temporary passphrase, nevertheless it should be a good one
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda2
$ cryptsetup luksFormat /dev/sda2
$ cryptsetup open /dev/sda2 cryptboot
$ mkfs.ext2 /dev/mapper/cryptboot
```

Create an encrypted container containing an the logical volumes /root and swap. Again set a temporary safe passphrase.
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda3
$ cryptsetup open /dev/sda3 cryptlvma
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
$ lvcreate -L 16G vg0 -n swap # This should be at least the size of your RAM if you want hybernation to work
$ lvcreate -l 100%FREE vg0 -n root
$ mkfs.ext4 /dev/mapper/vg0-root
$ mkswap /dev/mapper/vg0-swap
```

Mount everything on the live system.
```bash
$ mkdir /mnt/boot
$ mkdir /mnt/boot/efi
$ mount /dev/mapper/vg0-root /mnt
$ mount /dev/mapper/cryptboot /mnt/boot
$ mount /dev/sda1 /mnt/boot/efi
```

Activate the swap partition.
```bash
$ swapon /dev/mapper/vg0-swap
```

Check all the filesystems.
```bash
$ lsblk
```

If the output looks like this you're good to go.
```
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
loop0            7:0    0 347.9M  1 loop  /run/archiso/sfs/airootfs
sdb              8:32   1   3.8G  0 disk  
├─sdb2           8:34   1    40M  0 part  
└─sdb1           8:33   1   797M  0 part  /run/archiso/bootmnt
sda              8:0    0 931.5G  0 disk  
├─sda2           8:2    0   200M  0 part  
│ └─cryptboot  254:0    0   198M  0 crypt /mnt/boot
├─sda3           8:3    0   800G  0 part  
│ └─cryptlvm   254:1    0   800G  0 crypt 
│   ├─vg0-swap 254:2    0    16G  0 lvm   [SWAP]
│   └─vg0-root 254:3    0   784G  0 lvm   /mnt
└─sda1           8:1    0   512M  0 part  /mnt/boot/efi
```

## 3. Install base system & minimal configuration

Install the base system and some further components using pacstrap.
```bash
$ pacstrap /mnt base base-devel grub-efi-x86_64 vim git efibootmgr linux lvm2
```

Generate fstab with UUID representation.
```bash
$ genfstab -pU /mnt >> /mnt/etc/fstab
```

Chroot into your new base system.
```bash
$ arch-chroot /mnt
```

Set timezone and set your hwclock to use utc format.
```bash
$ ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime
$ hwclock --systohc --utc
```

Configure your locales.
```bash
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ echo "de_CH.UTF-8 UTF-8" >> /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

Set a hostname, keymap and nice console font.
```bash
$ echo "myhostname" > /etc/hostname
$ echo "KEYMAP=de_CH-latin1" >> /etc/vconsole.conf
$ echo "FONT=lat9w-16" >> /etc/vconsole.conf
$ echo "FONT_MAP=8859-1_to_uni" >> /etc/vconsole.conf
```

Set a strong root password.
```bash
$ passwd
```

Change mkinitcpio.conf to support lvm2 and encryption.
```bash
$ sed -i "s/MODULES=.*/MODULES=(ext4)/g" /etc/mkinitcpio.conf
$ sed -i "s/HOOKS=.*/HOOKS=(base udev autodetect modconf keyboard block keymap encrypt lvm2 resume filesystems keyboard fsck shutdown)/g" /etc/mkinitcpio.conf
```

Regenerate the initrd image.
```bash
$ mkinitcpio -p linux
```

## 4. Install and configure bootloader
Change your grub bootloader config.
```bash
$ echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
$ sed -i "s#^GRUB_CMDLINE_LINUX=.*#GRUB_CMDLINE_LINUX=\"cryptdevice=UUID=$(blkid /dev/sda3 -s UUID -o value):lvm resume=/dev/mapper/vg0-swap quiet splash\"#g" /etc/default/grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Install grub to your disk.
```bash
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux
```

Create a random keyfile for passwordless /boot decryption, set its permissions
and add it to the luks container. We're using crypttab for automount.
```bash
$ dd bs=512 count=8 if=/dev/urandom of=/etc/key
$ chmod 400 /etc/key
$ cryptsetup luksAddKey /dev/sda2 /etc/key
$ echo "cryptboot /dev/sda2 /etc/key luks" >> /etc/crypttab
```

Create a random keyfile for passwordless LVM decryption and add it to the luks
container. Tell the initrd where to get the file to decrypt the logical
volumes.
```bash
$ dd bs=512 count=8 if=/dev/urandom of=/keyfile.bin
$ chmod 000 /keyfile.bin
$ cryptsetup luksAddKey /dev/sda3 /keyfile.bin
$ sed -i 's\^FILES=.*\FILES="/keyfile.bin"\g' /etc/mkinitcpio.conf
$ mkinitcpio -p linux
$ chmod 600 /boot/initramfs-linux*
```

Create a new user and set its password.
```bash
$ useradd -m -g users -G wheel $YOUR_USER_NAME
$ passwd $YOUR_USER_NAME
```

Finally uncomment the string `%wheel ALL=(ALL) ALL` in `/etc/sudoers` to allow sudo for users of the group wheel.
```bash
$ vim /etc/sudoers
```

Exit from chroot, unmount system, shutdown, extract flash stick. You made it! Now you have fully encrypted system.
```bash
$ exit
$ umount -R /mnt
$ swapoff -a
$ shutdown now
```
