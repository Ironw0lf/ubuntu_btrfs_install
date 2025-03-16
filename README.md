# ubuntu_btrfs_install
How to install ubuntu 24.04 with BTRFS, timeshift and snapshot access via GRUB.
I was looking for a way to install this on my laptop and it took me a bit of research and trial and error but now it's working!

1. Start the ubuntu live USB and choose "Try ubuntu"
2. Once booted, open a terminal screen to prepare the partitions:
```
#To avoid typing sudo 23 times
sudo su -

#Identify which disk to use (sda, nvme0n1, etc.)
lsblk

# Create new partition using gdisk (example /dev/sda)
gdisk /dev/sda
# n (new partition) → 1 → Enter → +512M → EF00 (EFI)
# n → 2 → Enter → Enter (remaining space) → 8300 (Linux)
# w (write and exit)
```
3. Format partitions:
```
# EFI
mkfs.fat -F32 /dev/sda1

# BTRFS
mkfs.btrfs /dev/sda2
```
4. Create BTRFS sub-volumes:
You can create only @ and @home if you wish, Timeshift will be happy with that!
```
# Mount BTRFS
mount /dev/sda2 /mnt

# create sub-volume
btrfs subvolume cr /mnt/@
sudo btrfs subvolume cr /mnt/@home

# Unmount
umount /mnt
```
5. Mount file system:
```
# Root
mount -o subvol=@ /dev/sda2 /mnt

# Create mounting point
mkdir -p /mnt/{home,var,.snapshots,boot/efi}

# Mount sub-volumes
mount -o subvol=@home /dev/sda2 /mnt/home

# EFI
mount /dev/sda1 /mnt/boot/efi
```
6. Bootstrap ubuntu:
```
apt update
apt install debootstrap
debootstrap noble /mnt
```

7. Prepare chroot:
```
# Mount virtual filesystem
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

# Chroot
chroot /mnt
```
8. Configure system in chroot:
```
# Install essential packages
apt install nano git curl ubuntu-desktop linux-generic btrfs-progs grub-efi-amd64 efibootmgr

# Generate fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) / btrfs defaults,subvol=@ 0 0" >> /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) /home btrfs defaults,subvol=@home 0 0" >> /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda1) /boot/efi vfat umask=0077 0 1" >> /etc/fstab

# Install GRUB
grub-install /dev/sda
update-grub

# Update initramfs
update-initramfs -u -k all

# Define root password
passwd

# Create user
adduser votre_nom_utilisateur
usermod -aG sudo votre_nom_utilisateur

#set hostname
echo "your-hostname" > /etc/hostname
echo "127.0.0.1 your-hostname" >> /etc/hosts
```
9. Exit and reboot:
```
exit
umount -R /mnt
reboot
```
10. Configure Timeshift
11. Install grub-btrfs https://github.com/Antynea/grub-btrfs

You should now have ubuntu using BTRFS with timeshift and access to snapshot via grub. It will be very barebone and you should go into the software updater to add the repository you want!
 
