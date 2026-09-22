# Pck Linux Installation Guide (PckGuide)

This guide walks you through installing Pck Linux by hand, step by step —
no installer script involved. It assumes you've booted the Pck Linux live
ISO and are at a root/sudo shell.

**This process will erase data on the target disk.** Double-check every
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
sudo lsblk
```

Find your target disk's device name (e.g. `/dev/sda`, `/dev/nvme0n1`).
**Note it carefully.** Every command below uses `/dev/sda` as an example —
replace it with your actual disk name every time you see it.

## 3. Wipe the disk

This clears out any old partition table, bootloader, or filesystem
signatures so you're working with a genuinely clean disk. Replace
`/dev/sda` with your actual disk:

```bash
sudo dd if=/dev/zero of=/dev/sda bs=1M count=100 status=progress
```

This zeroes out the first 100MB of the disk — enough to destroy the
partition table and any old bootloader. There is no undo. Confirm you
have the right disk before running this.

## 4. Partition the disk

Launch cfdisk on your target disk — it's a visual, menu-driven partitioner
(arrow keys to move, Enter to select, no need to memorize commands).
Replace `/dev/sda` with your actual disk name:

```bash
sudo cfdisk /dev/sda
```

If the disk has no partition table yet, cfdisk will ask you to pick a
label type first: choose **gpt** for UEFI, or **dos** for BIOS/legacy.

### If you're on UEFI:
1. Highlight **Free space**, select **[ New ]**, press Enter
2. Type `512M` for the size, press Enter — this creates your first
   partition (the EFI System Partition)
3. With that partition highlighted, select **[ Type ]** and choose
   **EFI System** from the list
4. Highlight the remaining **Free space**, select **[ New ]**, press
   Enter, then press Enter again to use all remaining space — this is
   your root partition
5. Select **[ Write ]**, type `yes` to confirm, then select **[ Quit ]**

### If you're on BIOS/legacy:
1. Highlight **Free space**, select **[ New ]**, press Enter
2. Type `512M` for the size, press Enter — this is your boot partition
3. With that partition highlighted, select **[ Bootable ]** to flag it
4. Highlight the remaining **Free space**, select **[ New ]**, press
   Enter, then press Enter again to use all remaining space — this is
   your root partition
5. Select **[ Write ]**, type `yes` to confirm, then select **[ Quit ]**

Your partitions will now be named things like `/dev/sda1` and `/dev/sda2`
(or `/dev/nvme0n1p1`/`/dev/nvme0n1p2` on NVMe drives — note the extra `p`
before the number). Confirm the exact names with:

```bash
lsblk
```

**Throughout the rest of this guide:**
- Wherever you see `/dev/sda`, use your actual disk
- Wherever you see `/dev/sda1`, use your actual boot/EFI partition
- Wherever you see `/dev/sda2`, use your actual root partition

## 5. Format the boot partition

```bash
# UEFI:
sudo mkfs.fat -F32 /dev/sda1

# BIOS:
sudo mkfs.ext4 -F /dev/sda1
```

## 6. (Optional) Encrypt the root partition with LUKS

Skip this whole section if you don't want disk encryption.

```bash
sudo cryptsetup luksFormat /dev/sda2
sudo cryptsetup luksOpen /dev/sda2 pcklinux-root
```

You'll be prompted to set and then confirm a passphrase. Once open, your
decrypted device is available at `/dev/mapper/pcklinux-root` — use that
as your root device in the next step instead of `/dev/sda2` directly.

## 7. Format the root partition

```bash
# If NOT encrypted, format the partition directly:
sudo mkfs.ext4 -F /dev/sda2

# If encrypted (from step 6), format the opened mapper device instead:
sudo mkfs.ext4 -F /dev/mapper/pcklinux-root
```

**From here on:** if you did NOT encrypt, use `/dev/sda2` as your root
device in the steps below. If you DID encrypt, use
`/dev/mapper/pcklinux-root` instead, everywhere you see a root device
referenced.

## 8. Mount everything

```bash
sudo mkdir -p /mnt/pcklinux
sudo mount /dev/sda2 /mnt/pcklinux   # or /dev/mapper/pcklinux-root if encrypted

# UEFI:
sudo mkdir -p /mnt/pcklinux/boot/efi
sudo mount /dev/sda1 /mnt/pcklinux/boot/efi

# BIOS:
sudo mkdir -p /mnt/pcklinux/boot
sudo mount /dev/sda1 /mnt/pcklinux/boot
```

## 9. Copy the live system to disk

This clones the currently running live system onto your new disk:

```bash
sudo rsync -aHAXx --info=progress2 --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/mnt --exclude=/media --exclude=/tmp --exclude=/run --exclude=/lost+found / /mnt/pcklinux/
```

```bash
sudo mkdir -p /mnt/pcklinux/{proc,sys,dev,run,tmp}
```

This takes a few minutes — let it finish completely.

## 10. (Optional) Set up swap

```bash
sudo fallocate -l 2G /mnt/pcklinux/swapfile   # adjust size as you like
sudo chmod 600 /mnt/pcklinux/swapfile
sudo mkswap /mnt/pcklinux/swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /mnt/pcklinux/etc/fstab
```

## 11. Add fstab entries for root and boot (UUID-based)

Device names like `/dev/sda1` can shift between boots (extra drives
plugged in, etc.) — UUIDs don't change, so real systems reference
partitions by UUID in `/etc/fstab` rather than by device name. Look up
your partitions' UUIDs:

```bash
sudo blkid /dev/sda1
sudo blkid /dev/sda2
```

Each prints a line containing `UUID="...."`. Copy each one, then add
the entries by hand (replace `PASTE-ROOT-UUID` and `PASTE-BOOT-UUID`
with what you copied):

```bash
echo "UUID=PASTE-ROOT-UUID / ext4 errors=remount-ro 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab
```

```bash
# UEFI:
echo "UUID=PASTE-BOOT-UUID /boot/efi vfat umask=0077 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab

# BIOS:
echo "UUID=PASTE-BOOT-UUID /boot ext4 defaults 0 2" | sudo tee -a /mnt/pcklinux/etc/fstab
```

If you encrypted root with LUKS, use `/dev/mapper/pcklinux-root` in
place of `/dev/sda2` when running `blkid` above — the root entry then
doesn't need a UUID at all, since the mapper name is already stable:

```bash
echo "/dev/mapper/pcklinux-root / ext4 errors=remount-ro 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab
```

## 12. Set hostname

```bash
echo "pck-linux" | sudo tee /mnt/pcklinux/etc/hostname   # pick any hostname you like
sudo sed -i "s/127.0.1.1.*/127.0.1.1\tpck-linux/" /mnt/pcklinux/etc/hosts
```

## 13. Enter the new system (chroot)

```bash
sudo mount --bind /dev /mnt/pcklinux/dev
sudo mount --bind /proc /mnt/pcklinux/proc
sudo mount --bind /sys /mnt/pcklinux/sys
sudo chroot /mnt/pcklinux /bin/bash
```

**Everything from here until "exit" runs inside the new system.**

Note: you're already root inside the chroot at this point (that's what
`sudo chroot` drops you into), so the commands in steps 14–20 below
don't need `sudo` in front of them — you already have full access.

## 14. Set timezone

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime   # e.g. Asia/Manila
echo "Region/City" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata
```

## 15. Set keyboard layout

```bash
sed -i 's/^XKBLAYOUT=.*/XKBLAYOUT="us"/' /etc/default/keyboard   # change "us" as needed
dpkg-reconfigure -f noninteractive keyboard-configuration
```

## 16. Create your user account

```bash
useradd -m -s /bin/bash yourusername
passwd yourusername
usermod -aG sudo yourusername
```

## 17. (Optional) Enable/disable automatic time sync

```bash
systemctl enable systemd-timesyncd    # to enable
# or
systemctl disable systemd-timesyncd   # to disable
```

## 18. (Optional) Install an audio server

```bash
apt-get update

# Pick ONE:
apt-get install --no-install-recommends alsa-utils            # ALSA only
apt-get install --no-install-recommends pipewire pipewire-pulse wireplumber   # PipeWire
apt-get install --no-install-recommends pulseaudio             # PulseAudio
```

## 19. Install the bootloader

If you encrypted root (step 6), first:

```bash
echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
blkid -s UUID -o value /dev/sda2
```

Copy the UUID it prints, then run (replacing `PASTE-UUID-HERE`):

```bash
echo "pcklinux-root UUID=PASTE-UUID-HERE none luks" >> /etc/crypttab
```

Then install GRUB:

```bash
# UEFI:
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Pck Linux" --removable

# BIOS (use the whole disk, e.g. /dev/sda — NOT a partition like /dev/sda1):
grub-install /dev/sda

update-initramfs -u -k all
update-grub
```

## 20. (Optional) Install any extra packages you want

```bash
apt-get update
apt-get install your-package-names-here
```

## 21. Exit and clean up

```bash
exit   # leave the chroot

sudo umount /mnt/pcklinux/dev /mnt/pcklinux/proc /mnt/pcklinux/sys

# UEFI:
sudo umount /mnt/pcklinux/boot/efi
# BIOS:
sudo umount /mnt/pcklinux/boot

sudo umount /mnt/pcklinux

# If you used encryption:
sudo cryptsetup luksClose pcklinux-root
```

## 22. Reboot

```bash
sudo reboot
```

Remove the installation media when prompted. You should boot straight
into your new Pck Linux install.

---

## Installing a desktop environment (optional, after first boot)

Pck Linux ships with no desktop environment or display manager by
design. This section covers adding them after you've booted into your
installed system and logged in.

### 1. Install a display manager

A display manager shows the graphical login screen and starts your
session. Pick one:

```bash
sudo apt update

# LightDM (lightweight, simple):
sudo apt install --no-install-recommends lightdm

# SDDM (KDE's default, also lightweight):
sudo apt install --no-install-recommends sddm

# GDM (GNOME's default, heavier):
sudo apt install --no-install-recommends gdm3
```

You'll need a display server too if you don't already have one:

```bash
sudo apt install --no-install-recommends xserver-xorg
```

### 2. Install a desktop environment

Pick one (or install more than one — you can choose between them at
the login screen):

```bash
# XFCE (lightweight):
sudo apt install --no-install-recommends xfce4

# KDE Plasma (full-featured):
sudo apt install --no-install-recommends kde-plasma-desktop

# GNOME (full-featured):
sudo apt install --no-install-recommends gnome-core

# A minimal window manager instead of a full DE (very lightweight):
sudo apt install --no-install-recommends openbox
```

### 3. Enable the display manager

```bash
sudo systemctl enable lightdm    # or sddm / gdm3, matching what you installed
```

### 4. Reboot

```bash
sudo reboot
```

You should now land on a graphical login screen instead of a text
console. Log in with the user account you created during install.

### Notes

- `--no-install-recommends` keeps installs lean, matching Pck Linux's
  minimal philosophy — drop it if you'd rather get the fuller default
  package set for whichever DE you pick.
- If you skipped audio setup during install (or picked "none"), see
  the Audio step earlier in this guide to add PipeWire, PulseAudio, or
  ALSA now — most desktop environments expect one of these to be
  present for sound to work.
- A browser (e.g. Firefox) and other GUI apps aren't included either —
  install them the same way, with plain `apt install`.

---

## A note on device names used in this guide

This guide uses `/dev/sda` (disk), `/dev/sda1` (boot/EFI partition), and
`/dev/sda2` (root partition) as examples throughout. Substitute your own
actual device names wherever you see these — check with `lsblk` at any
point if you're unsure. NVMe drives look like `/dev/nvme0n1`,
`/dev/nvme0n1p1`, `/dev/nvme0n1p2` instead.

If anything goes wrong partway through, you can safely start over from
step 4 — just make sure to unmount anything you'd already mounted first
(`sudo umount -R /mnt/pcklinux` usually handles this).
