# Pck-linux-0.0.1-Test-.iso

> **BETA / TEST BUILD — READ BEFORE USING**
>
> This ISO is still in beta/testing. It has not been extensively tested
> across hardware, and things may break. If you want to try it, **run it
> in a virtual machine (VMware, VirtualBox, GNOME Boxes, QEMU, etc.)** —
> don't install it on real hardware you care about.
>
> Don't say you weren't warned.

Pck Linux is a minimal, Debian-based Linux distribution built with `live-build`.
No desktop environment is installed by default — you choose what goes on top.

Built on Debian 13 (trixie).

## Features

- Minimal base install (sudo, network-manager + iwd, nano, curl, and the essentials)
- No installer script — install manually following [INSTALL.md](INSTALL.md),
  Arch-Wiki style, with full control over every step
- Supports both BIOS and UEFI, optional LUKS disk encryption, optional swap
- Custom branding, no bloat, no desktop environment forced on you

## Installing

1. Download the latest `.iso` from [Releases](../../releases)
2. Boot it in a VM (see warning above) or on spare hardware
3. Follow [INSTALL.md](INSTALL.md) for the full manual installation walkthrough

The guide is also baked into the live system itself at `/root/INSTALL.md`.

Installing will erase data on whatever disk you target. Read each step
before running it.

## Building from source

Requires a Debian/Debian-based host with `live-build` installed.

```bash
sudo apt install live-build
git clone https://github.com/YOUR_USERNAME/pcklinux.git
cd pcklinux
lb config --distribution trixie \
  --archive-areas "main contrib non-free non-free-firmware" \
  --iso-volume "Pck Linux" \
  --iso-application "Pck Linux"
lb build
```

The resulting `live-image-amd64.hybrid.iso` is your bootable image.

## License

Pck Linux's own original work (configuration, branding, documentation) is
licensed under the GNU General Public License v2.0 — see [LICENSE](LICENSE).

Pck Linux is built on top of Debian and the Linux kernel, which retain their
own respective licenses. See [NOTICE](NOTICE) for full attribution and
license details for upstream components.

## Disclaimer

This is a personal/hobby project, currently in beta. Provided as-is, with
no warranty. Not affiliated with or endorsed by the Debian Project.
