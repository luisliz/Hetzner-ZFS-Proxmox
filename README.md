# Hetzner-ZFS-Proxmox
Instructions for installing proxmox in hetzner with zfs.  
Enter Rescue Mode:

### 1.1. Log into the Hetzner Robot Web Interface.

### 1.2. Select your server, go to the "Rescue" tab, select "Linux" as the operating system and click "Activate rescue system". You will receive an email with the rescue system's credentials.

### 1.3. Restart the server to enter the rescue system.

SSH into Rescue System:

### 2.1. SSH into the server using the provided credentials in the email.

Setup the Partition Scheme for NVMe and Hard drives:

It's recommended to use the NVMe drives for the ZFS rpool (OS and VM images) and the hard drives for a secondary pool for data storage.

Follow these steps for the NVMe drives:

### 3.1. Use fdisk to create a GPT partition table and a single Linux partition on each NVMe drive. For example, for the first NVMe drive:
use `lsblk` if you don't know your drives. 

```bash
fdisk /dev/nvme0n1
````
Follow the prompts to create a new GPT table (g command) and a new partition (n command). Repeat the process for the second NVMe drive.

#### Create a ZFS mirrored pool using these partitions:

```bash
zpool create -f -o ashift=12 -O atime=off -O canmount=off -O compression=lz4 -O normalization=formD -O mountpoint=/ -R /mnt rpool mirror /dev/nvme0n1p1 /dev/nvme1n1p1
```

Do similar steps for the hard drives but the pool should be created without the -R /mnt and -O mountpoint=/ options.
I like to do this to avoid boot errors 
```bash
zpool create -f -o ashift=12 -O atime=off -O canmount=off -O compression=lz4 -O normalization=formD data-pool mirror /dev/sda /dev/sdb
```

#### Install Proxmox:

Download and install Proxmox onto the ZFS rpool:

```bash
apt-get update
apt-get install -y debootstrap gdisk zfs-initramfs
debootstrap --arch amd64 buster /mnt
```

Mount the system folders into the chroot environment:
```bash
mount --rbind /dev /mnt/dev
mount --rbind /proc /mnt/proc
mount --rbind /sys /mnt/sys
```
Chroot into the environment:

```bash
chroot /mnt /bin/bash --login
```

Set up your /etc/apt/sources.list:

```bash
echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

echo "deb http://deb.debian.org/debian bookworm main" >> /etc/apt/sources.list
echo "deb-src http://deb.debian.org/debian bookworm main" >> /etc/apt/sources.list

echo "deb http://deb.debian.org/debian-security/ bookworm-security main" >> /etc/apt/sources.list
echo "deb-src http://deb.debian.org/debian-security/ bookworm-security main" >> /etc/apt/sources.list

echo "deb http://deb.debian.org/debian bookworm-updates main" >> /etc/apt/sources.list
echo "deb-src http://deb.debian.org/debian bookworm-updates main" >> /etc/apt/sources.list
```

Update repo and system 
```bash
apt update && apt full-upgrade
apt install pve-kernel-6.2
```

Install proxmox 
```bash
apt update
apt dist-upgrade
apt install proxmox-ve postfix open-iscsi
```

Grub stuff 
```bash
grub-probe /boot/grub

grub-install /dev/nvme0n1
grub-install /dev/nvme1n1
update-grub
```

Exit & Reboot
```bash 
exit && reboot
```
