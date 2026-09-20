# Installing Pck Linux (Manual Guide)

This guide walks you through installing Pck Linux by hand, step by step —
no installer script involved. It assumes you've booted the Pck Linux live
ISO and are at a root/sudo shell.

⚠️ **This process will erase data on the target disk.** Double-check every
device name before running destructive commands.

---

## 1. Connect to the network (if needed)

If you're on Wi-Fi and need internet access during install (for optional
packages later):

```bash
sudo nmtui
```

Use it to activate a connection, then exit back to the shell. Ethernet
usually just works via DHCP with no setup needed.

## 2. Identify your target disk

```bash
lsblk
fdisk -l
```

Find your target disk's device name (e.g. `/dev/sda`, `/dev/nvme0n1`).
**Note it carefully — everything below uses `$TARGET` as a placeholder.**

```bash
TARGET=/dev/sdX   # replace sdX with your actual disk
```

## 3. Partition the disk

Launch fdisk on your target disk:

```bash
fdisk "$TARGET"
```

### If you're on UEFI:
1. `g` — create a new empty GPT partition table
2. `n` then Enter, Enter, `+512M` — create a 512MB first partition (this
   will be your EFI System Partition)
3. `t` then `1` — set that first partition's type to "EFI System"
4. `n` then Enter, Enter, Enter — create a second partition using the
   remaining space (this will be root)
5. `w` — write changes and exit

### If you're on BIOS/legacy:
1. `o` — create a new empty DOS/MBR partition table
2. `n`, `p`, `1`, Enter, `+512M` — create a 512MB first partition (boot)
3. `a` — mark it bootable
4. `n`, `p`, `2`, Enter, Enter — create a second partition using the
   remaining space (root)
5. `w` — write changes and exit

Your partitions will now be named things like `${TARGET}1` and `${TARGET}2`
(or `${TARGET}p1`/`${TARGET}p2` on NVMe drives). Set variables for clarity:

```bash
BOOT_PART=/dev/sdX1   # adjust to match your actual partitions
ROOT_PART=/dev/sdX2
```

## 4. Format the boot partition

```bash
# UEFI:
mkfs.fat -F32 "$BOOT_PART"

# BIOS:
mkfs.ext4 -F "$BOOT_PART"
```

## 5. (Optional) Encrypt the root partition with LUKS

Skip this whole section if you don't want disk encryption.

```bash
cryptsetup luksFormat "$ROOT_PART"
cryptsetup luksOpen "$ROOT_PART" pcklinux-root
```

You'll be prompted to set and then confirm a passphrase. Once open, your
decrypted device is available at `/dev/mapper/pcklinux-root` — use that
as your root device in the next step instead of `$ROOT_PART` directly.

## 6. Format the root partition

```bash
# If NOT encrypted:
mkfs.ext4 -F "$ROOT_PART"
ROOT_DEV="$ROOT_PART"

# If encrypted (from step 6):
mkfs.ext4 -F /dev/mapper/pcklinux-root
ROOT_DEV=/dev/mapper/pcklinux-root
```

## 7. Mount everything

```bash
mkdir -p /mnt/pcklinux
mount "$ROOT_DEV" /mnt/pcklinux

# UEFI:
mkdir -p /mnt/pcklinux/boot/efi
mount "$BOOT_PART" /mnt/pcklinux/boot/efi

# BIOS:
mkdir -p /mnt/pcklinux/boot
mount "$BOOT_PART" /mnt/pcklinux/boot
```

## 8. Copy the live system to disk

This clones the currently running live system onto your new disk:

```bash
rsync -aHAXx --info=progress2 \
    --exclude=/proc --exclude=/sys --exclude=/dev \
    --exclude=/mnt --exclude=/media --exclude=/tmp \
    --exclude=/run --exclude=/lost+found \
    / /mnt/pcklinux/

mkdir -p /mnt/pcklinux/{proc,sys,dev,run,tmp}
```

This takes a few minutes — let it finish completely.

## 9. (Optional) Set up swap

```bash
fallocate -l 2G /mnt/pcklinux/swapfile   # adjust size as you like
chmod 600 /mnt/pcklinux/swapfile
mkswap /mnt/pcklinux/swapfile
echo "/swapfile none swap sw 0 0" >> /mnt/pcklinux/etc/fstab
```

## 10. Set hostname

```bash
echo "pck-linux" > /mnt/pcklinux/etc/hostname   # pick any hostname you like
sed -i "s/127.0.1.1.*/127.0.1.1\tpck-linux/" /mnt/pcklinux/etc/hosts
```

## 11. Enter the new system (chroot)

```bash
mount --bind /dev /mnt/pcklinux/dev
mount --bind /proc /mnt/pcklinux/proc
mount --bind /sys /mnt/pcklinux/sys
chroot /mnt/pcklinux /bin/bash
```

**Everything from here until "exit" runs inside the new system.**

## 12. Set timezone

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime   # e.g. Asia/Manila
echo "Region/City" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata
```

## 13. Set keyboard layout

```bash
sed -i 's/^XKBLAYOUT=.*/XKBLAYOUT="us"/' /etc/default/keyboard   # change "us" as needed
dpkg-reconfigure -f noninteractive keyboard-configuration
```

## 14. Create your user account

```bash
useradd -m -s /bin/bash yourusername
passwd yourusername
usermod -aG sudo yourusername
```

## 15. (Optional) Enable/disable automatic time sync

```bash
systemctl enable systemd-timesyncd    # to enable
# or
systemctl disable systemd-timesyncd   # to disable
```

## 16. (Optional) Install an audio server

```bash
apt-get update

# Pick ONE:
apt-get install --no-install-recommends alsa-utils            # ALSA only
apt-get install --no-install-recommends pipewire pipewire-pulse wireplumber   # PipeWire
apt-get install --no-install-recommends pulseaudio             # PulseAudio
```

## 17. Install the bootloader

If you encrypted root (step 5), first:

```bash
echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
UUID=$(blkid -s UUID -o value "$ROOT_PART")
echo "pcklinux-root UUID=${UUID} none luks" >> /etc/crypttab
```

Then install GRUB:

```bash
# UEFI:
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Pck Linux" --removable

# BIOS:
grub-install "$TARGET"   # the whole disk, e.g. /dev/sda — NOT a partition

update-initramfs -u -k all
update-grub
```

## 18. (Optional) Install any extra packages you want

```bash
apt-get update
apt-get install your-package-names-here
```

## 19. Exit and clean up

```bash
exit   # leave the chroot

umount /mnt/pcklinux/dev /mnt/pcklinux/proc /mnt/pcklinux/sys

# UEFI:
umount /mnt/pcklinux/boot/efi
# BIOS:
umount /mnt/pcklinux/boot

umount /mnt/pcklinux

# If you used encryption:
cryptsetup luksClose pcklinux-root
```

## 20. Reboot

```bash
reboot
```

Remove the installation media when prompted. You should boot straight
into your new Pck Linux install.

---

## Quick reference: variables used in this guide

| Variable | Meaning |
|---|---|
| `$TARGET` | The whole disk (e.g. `/dev/sda`) |
| `$BOOT_PART` | The boot/EFI partition (e.g. `/dev/sda1`) |
| `$ROOT_PART` | The root partition (e.g. `/dev/sda2`) |
| `$ROOT_DEV` | The actual device to format/mount as root — same as `$ROOT_PART`, or `/dev/mapper/pcklinux-root` if encrypted |

If anything goes wrong partway through, you can safely start over from
step 3 — just make sure to unmount anything you'd already mounted first
(`umount -R /mnt/pcklinux` usually handles this).
