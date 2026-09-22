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

The kernel may still be holding onto old filesystem signatures and
partition-table metadata from before the wipe. Clear those explicitly,
then tell the kernel to reread the (now-empty) partition table:

```bash
sudo wipefs -a /dev/sda
sudo partprobe /dev/sda
```

Verify the disk is genuinely empty before moving on:

```bash
sudo lsblk -f /dev/sda
```

You should see the disk with no partitions and no filesystem type
listed underneath it.

## 4. Partition the disk

Launch cfdisk on your target disk — it's a visual, menu-driven partitioner
(arrow keys to move, Enter to select, no need to memorize commands).
Replace `/dev/sda` with your actual disk name:

```bash
sudo cfdisk /dev/sda
```

cfdisk will ask you to pick a label type first. **This choice matters —
pick wrong and later steps won't work:**

- **UEFI system → choose `gpt`**
- **BIOS/legacy system → choose `dos`** (this is what gives you the
  `[Bootable]` option later — `gpt` does not have it)

If you're not sure which one you are, check now:

```bash
if [ -d /sys/firmware/efi ]; then echo "You are on UEFI - use gpt"; else echo "You are on BIOS - use dos"; fi
```

### If you're on UEFI (gpt):
1. Highlight **Free space**, select **[ New ]**, press Enter
2. Type `512M` for the size, press Enter — this creates your first
   partition (the EFI System Partition)
3. With that partition highlighted, select **[ Type ]** and choose
   **EFI System** from the list
4. Highlight the remaining **Free space**, select **[ New ]**, press
   Enter, then press Enter again to use all remaining space — this is
   your root partition
5. Select **[ Write ]**, type `yes` to confirm, then select **[ Quit ]**

### If you're on BIOS/legacy (dos):
1. Highlight **Free space**, select **[ New ]**, press Enter
2. Type `512M` for the size, press Enter — this is your boot partition
3. With that partition highlighted, select **[ Bootable ]** to flag it
4. Highlight the remaining **Free space**, select **[ New ]**, press
   Enter, then press Enter again to use all remaining space — this is
   your root partition
5. Select **[ Write ]**, type `yes` to confirm, then select **[ Quit ]**

Your partitions will now be named things like `/dev/sda1` and `/dev/sda2`
(or `/dev/nvme0n1p1`/`/dev/nvme0n1p2` on NVMe drives — note the extra `p`
before the number). Confirm the exact names and that both partitions
exist with the correct type:

```bash
sudo lsblk -f /dev/sda
```

**Throughout the rest of this guide:**
- Wherever you see `/dev/sda`, use your actual disk
- Wherever you see `/dev/sda1`, use your actual boot/EFI partition
- Wherever you see `/dev/sda2`, use your actual root partition

## 5. Format the boot partition

```bash
# UEFI:
sudo mkfs.fat -F32 -n PCKBOOT /dev/sda1

# BIOS:
sudo mkfs.ext4 -F -b 4096 -m 1 -L pckboot -E lazy_itable_init=0,lazy_journal_init=0 /dev/sda1
```

`-b 4096` sets an explicit block size instead of trusting the default.
`-m 1` reserves only 1% of the partition for root (the ext4 default of
5% is meant for large root filesystems on old spinning disks — wasteful
here). `-E lazy_itable_init=0,lazy_journal_init=0` forces mkfs to fully
initialize the filesystem now instead of lazily in the background,
so you know it's completely ready before continuing.

Look up this partition's UUID now — you'll need it twice (once to
mount it in a moment, once again later for fstab):

```bash
sudo blkid /dev/sda1
```

Write down the `UUID="...."` value it prints.

## 6. (Optional) Encrypt the root partition with LUKS

Skip this whole section if you don't want disk encryption.

```bash
sudo cryptsetup luksFormat /dev/sda2
sudo cryptsetup luksOpen /dev/sda2 pcklinux-root
```

You'll be prompted to set and then confirm a passphrase. Verify the
mapping is actually active before continuing:

```bash
sudo cryptsetup status pcklinux-root
```

This should report the mapping as active, along with the cipher and
key size in use. Your decrypted device is available at
`/dev/mapper/pcklinux-root` — use that as your root device in the next
step instead of `/dev/sda2` directly.

## 7. Format the root partition

```bash
# If NOT encrypted, format the partition directly:
sudo mkfs.ext4 -F -b 4096 -m 1 -L pcklinux-root -E lazy_itable_init=0,lazy_journal_init=0 /dev/sda2

# If encrypted (from step 6), format the opened mapper device instead:
sudo mkfs.ext4 -F -b 4096 -m 1 -L pcklinux-root -E lazy_itable_init=0,lazy_journal_init=0 /dev/mapper/pcklinux-root
```

**From here on:** if you did NOT encrypt, use `/dev/sda2` as your root
device in the steps below. If you DID encrypt, use
`/dev/mapper/pcklinux-root` instead, everywhere you see a root device
referenced.

Look up the root device's UUID now too, same as you did for boot
(skip this if you're using the LUKS mapper — that has a stable name
already and doesn't need a UUID for mounting):

```bash
sudo blkid /dev/sda2
```

Write down that `UUID="...."` value as well.

## 8. Mount everything

Mount by UUID rather than device path — this exercises the same
addressing you'll commit to permanently in `/etc/fstab` later, so
you're proving it works before you rely on it. Replace
`PASTE-ROOT-UUID` and `PASTE-BOOT-UUID` with the values you wrote down
in steps 5 and 7:

```bash
sudo mkdir -p /mnt/pcklinux
sudo mount UUID=PASTE-ROOT-UUID /mnt/pcklinux   # or: sudo mount /dev/mapper/pcklinux-root /mnt/pcklinux if encrypted

# UEFI:
sudo mkdir -p /mnt/pcklinux/boot/efi
sudo mount UUID=PASTE-BOOT-UUID /mnt/pcklinux/boot/efi

# BIOS:
sudo mkdir -p /mnt/pcklinux/boot
sudo mount UUID=PASTE-BOOT-UUID /mnt/pcklinux/boot
```

Confirm both are actually mounted where you expect:

```bash
findmnt /mnt/pcklinux
findmnt /mnt/pcklinux/boot   # or /mnt/pcklinux/boot/efi on UEFI
```

## 9. Copy the live system to disk

This clones the currently running live system onto your new disk using
two `tar` invocations piped together — one packs the live filesystem,
the other unpacks it straight into your new root partition:

```bash
sudo bash -c 'cd / && tar --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/mnt --exclude=/media --exclude=/tmp --exclude=/run --exclude=/lost+found -cpf - . | (cd /mnt/pcklinux && tar -xpf -)'
```

`-c` creates an archive, `-p` preserves permissions/ownership, `-f -`
means "write to stdout" (on the packing side) or "read from stdin" (on
the unpacking side) instead of an actual archive file — that's what
lets the two `tar` processes talk directly to each other through the
pipe, with nothing ever touching disk as an intermediate `.tar` file.

```bash
sudo mkdir -p /mnt/pcklinux/{proc,sys,dev,run,tmp}
```

This takes a few minutes — let it finish completely. Once done, spot
check that the copy actually landed:

```bash
ls /mnt/pcklinux/etc/os-release
```

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
partitions by UUID in `/etc/fstab` rather than by device name. Reuse
the UUIDs you wrote down in steps 5 and 7 (replace `PASTE-ROOT-UUID`
and `PASTE-BOOT-UUID` below):

```bash
echo "UUID=PASTE-ROOT-UUID / ext4 errors=remount-ro 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab
```

```bash
# UEFI:
echo "UUID=PASTE-BOOT-UUID /boot/efi vfat umask=0077 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab

# BIOS:
echo "UUID=PASTE-BOOT-UUID /boot ext4 defaults 0 2" | sudo tee -a /mnt/pcklinux/etc/fstab
```

If you encrypted root with LUKS, the root entry uses the stable mapper
name instead of a UUID:

```bash
echo "/dev/mapper/pcklinux-root / ext4 errors=remount-ro 0 1" | sudo tee -a /mnt/pcklinux/etc/fstab
```

Check the file actually looks right before moving on:

```bash
cat /mnt/pcklinux/etc/fstab
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

Verify GRUB actually found your root filesystem correctly before
moving on:

```bash
grub-probe /
```

This should print `ext4` (or `cryptodisk` first, then `ext4`, if
encrypted) with no errors. If it errors out, GRUB won't boot correctly
— fix this before rebooting.

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

Confirm everything actually unmounted cleanly:

```bash
findmnt | grep pcklinux
```

This should print nothing. If it prints anything, something's still
mounted — find it and unmount it before rebooting.

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
