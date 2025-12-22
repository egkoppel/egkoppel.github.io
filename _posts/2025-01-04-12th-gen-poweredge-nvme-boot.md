---
layout: post
title: "NVMe booting on 12th gen Poweredge"
---
For anyone who just wants the solution without the story, skip to [here](#What if the firmware just had drivers?).

---

For around three years I've been running a Proxmox server on an R720, for random projects like hosting this exact site.
(Also it's just fun to gauge people's reactions when you show them a photo of a big 2U server in your room.)
However, for around the past two years and eleven months, the boot disk setup has been extremely cursed.
I have the internal H710 mini reflahsed into IT mode, which is passed through to a TrueNAS VM, so none of the front bays are useable for a Proxmox boot drive.
Instead I found two internal SATA ports and a nice proprietary power connector (designed for the front panel optical and tape drives) that I decided could be put to use.
So I probed the power connector to figure out the pinout, cut the end off a SATA power connector, replaced it with DuPont connectors, and carefully forced them onto the right pins.
The SSDs were taped into the server above the PSU bays, and it's been working like that ever since.

Except recently it started getting full, and I wanted to actually hook up the optical drive, and it all just felt a bit janky.
The easiest solution to me seemed like installing an M.2 drive in one of the PCIe slots. 
**Apparently**, with the latest firmware version, the R720 was capable of booting directly from NVMe drives (spoiler: it was not).
So I set about installing the new drive, backed up my Proxmox config, and set up Proxmox on the new drive.
At which point I was met with the very unsatisfying
```
UEFI Boot Sources
- Unavailable: proxmox
- ...
```
Some more digging into the firmware revealed that it had not in fact discovered the NVMe drive, and so I'd need to boot from some other device.
Following the many online tutorials I went about installing Clover onto the internal SD card.
Which instantly threw a General Protection Fault.
Welp.
I found a Reddit post saying that only `r5122` and earlier versions worked on the R720s, but even so it still seemed a bit clunky having Clover and GRUB both installed, and only one of them configured by Proxmox.
For some unknown reason, I just wanted it to work with GRUB.
So what if I made it all GRUBs problem?
Cue figuring out how to `chroot` into Proxmox, transferring the EFI system partition over to the SD card, and messing with `fstab` (allegedly not supposed to be called "f-stab") to use the new ESP, to be promptly dropped into the GRUB rescue shell.
Because apparently GRUB uses the firmware drivers too, and there's no nice GRUB module I could just offload that problem too.

# What if the firmware just had drivers?

Clover is able to detect the drives because it bundles its own drivers (as far as I understand they're just the TianoCore EDK II reference drivers), and those drivers are just off the shelf UEFI `efi` files.
So I put a copy of the `r5122` Clover NVMe driver (`EFI/CLOVER/drivers/off/UEFI/Other/NvmExpressDxe.efi`) along with an EFI shell on the SD card, and booted into the shell.
As a sanity check, I ran `load <path to driver>` (tell the firmware to load the driver) followed by `map -r` (rescan and print drives), and like magic I now had an extra drive available!

So then I just needed to have the firmware to automatically load the driver: `bcfg driver add #### <path to driver>`.

The drivers are stored in EFI variables of the form `Driver####`, where `####` is a unique 4 digit hex number to identify the driver.
For me I had no other third party drivers installed, so I used `0`.
The driver was stored on the first drive, at `/EFI/CLOVER/NvmExpressDxe.efi`, so the full command became `bcfg driver add 0 fs0:\EFI\CLOVER\NvmExpressDxe.efi` (note the backslashes and DOS style drive identifier).
Rebooting back into the boot menu showed a lovely new boot entry: `Unknown EFI device`!