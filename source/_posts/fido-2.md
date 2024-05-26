---
title: Setting up FIDO2 on Linux (2) - Decrypt LUKS with FIDO2
date: 2024-5-25 20:00:00
tags: [fido, fedora, linux, security]
categories: Work
thumbnail: "https://i.imgur.com/7OialCz.jpeg"
---

# Prerequisites

Fedora is used as the base system for this guide. The same steps can be followed on other distributions.

Fedora comes with systemd-cryptsetup by default, which is a utility to manage encrypted block devices. We will use this utility to easily add a FIDO2 key as a mean to unlock the LUKS filesystem. You can also rely on your computer's TPM chip instead of a FIDO2 key, but this guide will focus on the latter. The steps are similar for both methods though.

Note that systemd-cryptsetup only supports LUKS2 and Fedora have been using LUKS2 by default since Fedora 30. If you have an older LUKS1 filesystem, you can upgrade it to LUKS2.

If you did a full disk encryption during installation, you are ready to start with the steps below. If you wish to encrypt an existing Fedora disk, you can follow our previous guide on [Encrypt an Existing Fedora Disk with LUKS](/2024/04/30/luks/).

Learn more about LUKS and FIDO2 in the [previous post](/2024/05/20/fido-1/).

# Identify the LUKS Partition

You will need the device path of the LUKS partition to add the FIDO2 key. Run the following command to list all block devices:

```bash
lsblk -f
```

The LUKS partition will have a `crypto_LUKS` filesystem type. In this example, the LUKS partition is `/dev/nvme0n1p6` since I use dual-boot with Windows on the same drive.

# Dracut Configuration

Dracut is the tool used to generate the initramfs image in Fedora. We need to add the FIDO2 module to the initramfs image to be able to unlock the LUKS partition with a FIDO2 key.

```bash
echo "add_dracutmodules+=\" fido2 \"" | sudo tee /etc/dracut.conf.d/fido2.conf
```

Now, enroll the FIDO2 key to the LUKS partition as an additional unlock method:

```bash
sudo systemd-cryptenroll --fido2-device auto /dev/nvmen1p3
```

By default PIN (if set up) and presence (touch) are the requested means for use.

Next, update crypttab and append `fido2-device=auto` to the options of the LUKS partition:

```bash
sudo systemd-cryptenroll --fido2-device auto /dev/nvmen1p3
```

Finally, regenerate the initramfs image:

```bash
sudo dracut -f
```

Know that the prompt for the LUKS password will be the same as before, but you will have to use the FIDO2 key PIN there instead and touch the key.

**Preview**

![](https://sysguides.com/wp-content/uploads/2021/11/03-13-install-fedora-35-with-luks-full-disk-encryption-luks2-passphrase.webp)


This post relies on the Fedora Magazine article [here](https://fedoramagazine.org/use-systemd-cryptenroll-with-fido-u2f-or-tpm2-to-decrypt-your-disk/)