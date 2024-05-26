---
title: Encrypt an Existing Fedora Disk with LUKS
date: 2024-4-30 20:00:00
tags: [fedora, linux, security]
categories: Work
thumbnail: "https://i.imgur.com/3AXCKT5.jpeg"
---

# Prerequisites

This guide assumes you have a default Fedora installation where you forgot to check the "Encrypt my data" box during installation, or perhaps if you have a secondary disk that you want to encrypt, let it be another home partition or an external drive. A manual encryption process post-installation is also useful if you want to set up a FIDO 2 key to unlock your LUKS filesystem. We will cover this topic on a separate upcoming post.

Having the following btrfs-based partition layout:

- Root partition
  - Btrfs subvolume 'root' mounted at /
  - Btrfs subvolume 'home' mounted at /home
- Boot partition mounted at /boot
- EFI partition mounted at /boot/efi for UEFI systems

Following this process may lead to data loss if not done correctly. Make sure to have the following:

- A full backup of your data before proceeding
- Cryptsetup installed on your system, can be installed with `sudo dnf install cryptsetup`
- At least 100 MB of free space
- A live USB of Fedora or any other Linux distribution

# Encrypting the Disk

## UUID and Kernel Version from Fedora Installation

After making sure you have the prerequisites, we start by taking note of the root partition UUID and kernel version.

Run the following command to get the UUID of the root partition:

```bash
lsblk -f
```

It might be useful to remember which partition is mounted at /boot and /boot/efi as well in case you have dual-boot systems.

Now, run the following command to get the kernel version:

```bash
uname -r
```

## Reboot into Live USB

### Mount the Root Partition in the Live USB

We first, make sure we are able to locate and identify the device of the root partition with:

```bash
blkid --uuid <root_partition_uuid>
```

![](https://i.imgur.com/EjXiIgA.png)

To avoind any troubles, run a check on the filesystem:

```bash
btrfs check <device>
```

![](https://i.imgur.com/qGkq4ed.png)

Once done, mount the root partition to /mnt:

```bash
mount /dev/<root_partition_device> /mnt
```

### Shrinking the Root Partition

We need to shrink the root partition to make space for the LUKS partition. We will shrink it by at least 32 MB:

```bash
btrfs filesystem resize -32M /mnt
```

When done, unmount the root partition to proceed with the encryption:

```bash
umount /mnt
```

### In-place LUKS Encryption of the Root Partition

We will now encrypt the root partition in-place. Run the following command to encrypt the root partition:

```bash
cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/<device>
```

When prompted, enter a passphrase for the root partition. This process may take a while depending on the size of the partition.

![](https://i.imgur.com/9rn13sr.png)

![](https://i.imgur.com/bDqZH0f.png)

When completed, the root partition will have a different UUID. Run the following command to get the new UUID and note it down.

```bash
lsblk -f
```

You will also notice the filesystem type of the root partition has changed to crypto_LUKS.

### Resizing the mapped filesystem to use the full partition

Open the partition and provide the passphrase when prompted:

```bash
cryptsetup open /dev/<device> system
```

Mount the mapped filesystem:

```bash
mount /dev/mapper/system /mnt
```

Resize the filesystem to use the full partition:

```bash
btrfs filesystem resize max /mnt
```

Unmount the mapped filesystem:

```bash
umount /mnt
```

![](https://i.imgur.com/169zxLr.png)

Don't close the partition yet and follow the next steps.

### GRUB Configuration

To be able to boot into the encrypted root partition, we need to update the GRUB configuration.

Mount the root subvolume to /mnt:

```bash
mount -t btrfs -o "noatime,subvol=root,compress=zstd:1" /dev/mapper/system /mnt
```

The noatime option is optional but recommended for SSDs to reduce the number of writes. The compress=zstd:1 option is also optional but recommended for better compression.

If know which devices are mounted at /boot and /boot/efi, mount them as well, otherwise the vfat partition is usually the EFI partition and the ext4 partition is usually the boot partition:

```bash
lsblk -f
mount /dev/<boot_partition_device> /mnt/boot
mount /dev/<efi_partition_device> /mnt/boot/efi
```

![](https://i.imgur.com/eR9weRT.png)

Now, bind the pseudo filesystems:

```bash
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /run /mnt/run
mount --bind /sys /mnt/sys
```

To open a shell within the filesystem environment, run the following command:

```bash
chroot /mnt /bin/bash
```

Chroot stands for change root and it changes the root directory for the current running process and its children to the specified directory.

Open the GRUB configuration file with your preferred text editor:

```bash
vi /etc/default/grub
```

Modify the line for kernel parameters to include the root new UUID identified as LUKS partition, and to temporarily disable the SELinux enforcing mode as shown below:

```bash
GRUB_CMDLINE_LINUX="[other params] rd.luks.uuid=<LUKS partition UUID> enforcing=0"
```

![](https://i.imgur.com/plMgrf3.png)

Now, to avoid trouble with SELinux, we need to relabel the filesystem:

```bash
touch /.autorelabel
```

On the next boot, the SELinux subsystem will detect this file, and then relabel all of the files on that system with the correct SELinux contexts, which may take a while.

Rebuild the GRUB configuration:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

If you have UEFI, run the following command as well:

```bash
grub2-mkconfig -o /etc/grub2-efi.cfg
```

Regenerate the initramfs to ensure cryptsetup is included, use the kernel version noted earlier:

```bash
dracut --kver <kernel version> --force
```

Exit the chroot environment:

```bash
exit
```

![](https://i.imgur.com/kiy88sm.png)

### Unmount and Reboot

Unmount the filesystems in reverse order:

![](https://i.imgur.com/JV947QM.png)

Close the LUKS partition:

```bash
cryptsetup close system
```

Reboot into the existing Fedora installation:

```bash
shutdown -r now
```

You will be prompted to enter the passphrase for the LUKS partition. If you have followed the steps correctly, you should be able to boot into the encrypted root partition.

![](https://sysguides.com/wp-content/uploads/2021/11/03-13-install-fedora-35-with-luks-full-disk-encryption-luks2-passphrase.webp)

## Post-Encryption

Reenable SELinux enforcing mode in the GRUB configuration file `etc/default/grub` by removing the `enforcing=0` parameter from the `GRUB_CMDLINE_LINUX` line. Save the file and rebuild the GRUB configuration the same way we did before:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

If you have UEFI, run the following command as well:

```bash
grub2-mkconfig -o /etc/grub2-efi.cfg
```

And we will need to relabel SELinux contexts again with:

```bash
touch /.autorelabel
```

Reboot and login to your system. You have successfully encrypted your root partition with LUKS.

Stay tuned for the next post on how to set up a FIDO 2 key to unlock your LUKS filesystem.

This post is my application of the answer provided by cam-rod on Unix Stack Exchange. The original answer can be found [here](https://unix.stackexchange.com/questions/710367/how-to-encrypt-an-existing-disk-on-fedora-without-formatting). Which heavily derives from maxschelpzig [answer](https://unix.stackexchange.com/a/584275/533888) and the [Arch Wiki](https://wiki.archlinux.org/title/dm-crypt/Device_encryption#Encrypt_an_existing_unencrypted_file_system). I also want to point out that the tool `luksipc` is no longer working and the `cryptsetup reencrypt` command is the only way to do an in-place encryption of a partition.
