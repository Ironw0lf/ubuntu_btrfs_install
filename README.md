# ubuntu_btrfs_install
How to install ubuntu 23.04 with BTRFS, timeshift and snapshot access via GRUB.
I was looking for a way to install this on my laptop and it took me a bit of research and trial and error but now it's working!

1. Start the ubuntu live USB and choose "Try ubuntu"
2. Once booted, open a terminal screen to prepare the partitions:
```
#Identify which disk to use (sda, nvme0n1, etc.)
lsblk

# Create new partition using gdisk (example /dev/sda)
sudo gdisk /dev/sda
# n (new partition) → 1 → Enter → +512M → EF00 (EFI)
# n → 2 → Enter → Enter (remaining space) → 8300 (Linux)
# w (write and exit)
```
3. Format partitions:
```
# EFI
sudo mkfs.fat -F32 /dev/sda1

# BTRFS
sudo mkfs.btrfs /dev/sda2
```
4. Create BTRFS sub-volumes:
```
# Mount BTRFS
sudo mount /dev/sda2 /mnt

# create sub-volume, you can create only @ and @home if you wish
sudo btrfs subvolume create /mnt/@
sudo btrfs subvolume create /mnt/@home
sudo btrfs subvolume create /mnt/@var
sudo btrfs subvolume create /mnt/@snapshots

# Unmount
sudo umount /mnt
```
5. Mount file system:
```
# Root
sudo mount -o subvol=@ /dev/sda2 /mnt

# Create mounting point
sudo mkdir -p /mnt/{home,var,.snapshots,boot/efi}

# Mount sub-volumes
sudo mount -o subvol=@home /dev/sda2 /mnt/home
sudo mount -o subvol=@var /dev/sda2 /mnt/var
sudo mount -o subvol=@snapshots /dev/sda2 /mnt/.snapshots

# EFI
sudo mount /dev/sda1 /mnt/boot/efi
```
6. Bootstrap ubuntu:
```
sudo apt update
sudo apt install debootstrap
sudo debootstrap noble /mnt
```

7. Prepare chroot:
```
# Mount virtual filesystem
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys

# Chroot
sudo chroot /mnt
```
8. Configure system in chroot:
```
# Install essential packages
apt install ubuntu-desktop linux-generic btrfs-progs grub-efi-amd64 efibootmgr

# Generate fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) / btrfs defaults,subvol=@ 0 0" >> /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) /home btrfs defaults,subvol=@home 0 0" >> /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) /var btrfs defaults,subvol=@var 0 0" >> /etc/fstab
echo "UUID=$(blkid -s UUID -o value /dev/sda2) /.snapshots btrfs defaults,subvol=@snapshots 0 0" >> /etc/fstab
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
```
9. Exit and reboot:
```
exit
sudo umount -R /mnt
sudo reboot
```
10. In ubuntu desktop install Timeshift
11. Install grub-btrfs https://github.com/Antynea/grub-btrfs

You should now have ubuntu using BTRFS with timeshift and access to snapshot via grub.
 
