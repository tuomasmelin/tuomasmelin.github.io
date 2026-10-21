---
title: "TIL: When the USB stick won't boot, boot the installer from the hard drive (Garage Door Project, Part 4/5)"
date: 2026-07-14
description: 'For a phone-friendly UI I wanted Home Assistant, and the Pi 1 is far too weak for it. Turning an old laptop into a server included a genuinely clever workaround.'
series: "Opening my garage door with my phone"
tags: [linux, antix, grub, docker, home-assistant, til]
---

**Series:** Opening my garage door with my phone — Part 4 of 5
**Learned:** 8–14 July 2026

I could now open the door with `curl`. For a phone-friendly UI I wanted **Home Assistant**, and the Pi 1 is far too weak for it. What I *did* have was an old Fujitsu Siemens laptop (1.8 GHz CPU, 2 GB RAM) dual-booting Windows Vista and Lubuntu 18.04. This post is about turning it into a server, which included a genuinely clever workaround.

## 1. Fixing Wi-Fi on an old Lubuntu

The Wi-Fi card wouldn't connect: `nmcli device` showed it as `unavailable`, and the GUI said **"device not managed"**.

Debugging path:

```bash
lspci -knn | grep -A3 Network   # Atheros AR242x, but no "Kernel driver in use"
sudo modprobe ath5k             # load the driver
dmesg | grep -i ath             # driver happy, card named wls1
rfkill list                     # nothing blocked
```

**What I learned:** the driver was fine; **Netplan** was configured so that NetworkManager didn't manage the card. The fix was a three-line file in `/etc/netplan/`:

```yaml
network:
  version: 2
  renderer: NetworkManager
```

```bash
sudo netplan apply
```

…and Wi-Fi networks appeared.

## 2. Lubuntu 18.04 is end-of-life, so pick a server OS

- `cat /etc/os-release` showed **18.04.5**, out of support since April 2021 (Lubuntu flavours only got three years).
- For an always-on box, a **headless** (no desktop) install leaves far more RAM for the actual work.
- **2 GB RAM is enough for Home Assistant** if you skip heavy extras (video processing, big databases).
- On low-end hardware, run **Home Assistant in Docker** rather than Home Assistant OS in a virtual machine.
- An **SSD** instead of an old spinning disk is the cheapest speed-up you can buy.

I wiped Windows Vista too. Nothing on it was needed.

## 3. The USB boot wall

This is where things went sideways. Every installer stick failed:

| Attempt | Result |
| --- | --- |
| Debian netinst via Startup Disk Creator | "This is not a bootable disk" |
| Debian, written with `dd` | same |
| Full Debian DVD, different stick | same |
| **Ventoy** (MBR) + ISO | Black screen, blinking cursor |
| Ventoy GPT for UEFI | Not even listed |
| antiX via Ventoy, then via `dd` | Black screen, blinking cursor |

Things I learned along the way:

- The BIOS was an old **PhoenixBIOS**: no UEFI, no Secure Boot, no CSM toggle. It only boots **MBR/legacy** media, so GPT/UEFI sticks are pointless.
- Ventoy's data partition is **exFAT**; an old Lubuntu needs `sudo apt install exfat-fuse exfat-utils` before you can copy ISOs onto it.
- `umount`, not `unmount`. (Yes, really.)

## 4. The workaround: boot the installer from the hard drive

The machine booted its *existing* GRUB from the hard drive perfectly. So instead of fighting the BIOS, I let GRUB start the antiX live system **straight from the disk**:

```bash
# 1. Mount the ISO and copy the kernel + initrd into /boot
sudo mkdir -p /mnt/iso
sudo mount -o loop ~/Downloads/antiX-*.iso /mnt/iso
sudo cp /mnt/iso/antix/vmlinuz   /boot/vmlinuz-antix
sudo cp /mnt/iso/antix/initrd.gz /boot/initrd-antix.gz

# 2. Copy the live filesystem folder to the root of the disk
sudo cp -r /mnt/iso/antix /
sudo umount /mnt/iso
```

Then add a menu entry to `/etc/grub.d/40_custom`:

```
menuentry "Install antiX from hard disk" {
    insmod ext2
    set root=(hd0,5)
    linux  /boot/vmlinuz-antix from=hd antiX=all
    initrd /boot/initrd-antix.gz
}
```

```bash
sudo update-grub
```

Each error on the way taught something:

| Error | Meaning / fix |
| --- | --- |
| `no such partition` / `you need to load the kernel first` | GRUB couldn't find the partition. `df /boot` said `/dev/sda5`; with `insmod ext2`, `(hd0,5)` worked. |
| `file /initrd-antix.gz not found` → kernel panic | The `initrd` line was missing the `/boot/` prefix. |
| `Could not find file antix/linuxfs` | The live system needs the whole `antix/` folder copied to the disk root. |

Then: **antiX booted to a desktop.** 🎉

## 5. Installing antiX from a live system on the same disk

The graphical installer showed an **empty partition list**, because the live system was running from `/dev/sda5` and the installer locked the disk.

What worked:

1. In **GParted**: delete the Windows partitions, create one **ext4** partition in the free space, and click the green ✓ to apply.
2. Skip the graphical installer and run **`cli-installer`** in a terminal.
3. Answer: no repartitioning, root = the new partition, no separate `/home`, and install **GRUB to the MBR** (legacy BIOS).

## 6. Docker on a system without systemd

antiX doesn't use systemd, which tripped up nearly every guide:

- `docker.io` from the repos didn't give me a working `docker` command. Following **Docker's official Debian instructions** did.
- `sudo service docker start` → `docker: unrecognized service`. Try `sudo /etc/init.d/docker start`, or start `dockerd` directly.
- `sudo docker run hello-world` is the "is it alive?" test.
- `docker: invalid reference format` = a **copy-pasted multi-line command** with broken `\` line breaks. Put it on one line:

```bash
sudo docker run -d --name homeassistant --privileged --restart=unless-stopped \
  -m 1G --memory-swap 1G \
  -v ~/homeassistant:/config \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable
```

More things I learned:

- `-m 1G --memory-swap 1G` caps Home Assistant's memory so it can't starve the whole laptop. Changing it means `docker stop`, `docker rm` and `docker run` again; your config survives in `~/homeassistant`.
- `--restart=unless-stopped` brings the container back **if the Docker daemon starts at boot**. On a SysV-style init that's `sudo update-rc.d docker defaults`.
- Files created by the container are owned by root: `sudo chown -R $USER:$USER ~/homeassistant` fixes "Permission denied" when editing `configuration.yaml`.
- Home Assistant felt slow because I used it **in a browser on the laptop itself**. Use it from your phone or another computer and let the laptop just serve.
- The lightest antiX desktops are IceWM (the default), Fluxbox and JWM.

## Takeaways

- "Unmanaged" Wi-Fi on Ubuntu-family systems: check Netplan's `renderer`.
- Old BIOS + modern boot media = pain. If the machine can boot its own disk, **let GRUB boot the installer** from it.
- Not every distro uses systemd. Look for `service`, `/etc/init.d` and `update-rc.d`.
- Run Home Assistant in Docker with a memory cap on small machines, and use it from another device.

**Next up:** finally connecting the real garage door, and reaching it from outside the house.
