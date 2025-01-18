---
layout: post
title:  "Portable ext HDD disconnects randomly on Proxmox"
date:   2025-01-18 19:45:00 +0800
permalink: /blog/:title
---
# Portable external Seagate HDD disconnects randomly on Proxmox (an in-progress analysis)

Recently, I added a 4TB [Seagate Ultra Touch HDD](https://www.seagate.com/sg/en/products/external-hard-drives/ultra-touch-external-drives/?bvstate=pg:2/ct:r)to my Proxmox server, as it was running out of storage for my media server.
> <img src="/assets/images/2025-01-18-Random-HDD-Disconnect-Proxmox/ultratouch.jpg" style="width: 300px; object-fit: cover;"/>
>

I created a ZFS pool on the HDD and mounted it to my media server VM. Everything worked fine initially, but I noticed that the HDD would randomly disconnect and reconnect. This became problematic because the ZFS pool would fault, and any container relying on the pool would lose access.

It was impossible to run `zpool clear` as well since the command reports that there were I/O errors present. The only temporary solution was to reboot the Proxmox server to fix the zpool, which took around 5-10mins (much longer than a normal reboot).

## The problem

Upon checking the dmesg logs with `dmesg | grep usb`, I observed the following:

```
[3481117.907192] usb 2-2: USB disconnect, device number 2        <---- HERE
[3481119.187728] usb 2-2: new SuperSpeed USB device number 3 using xhci_hcd
[3481119.200484] usb 2-2: New USB device found, idVendor=0bc2, idProduct=2065, bcdDevice=20.04
[3481119.200493] usb 2-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[3481119.200496] usb 2-2: Product: Ultra Touch
[3481119.200498] usb 2-2: Manufacturer: Seagate
[3481119.200500] usb 2-2: SerialNumber: 00000000NADD0ZSK
[3482648.308921] usb 2-2: USB disconnect, device number 3        <---- HERE
[3482649.582732] usb 2-2: new SuperSpeed USB device number 4 using xhci_hcd
[3482649.595471] usb 2-2: New USB device found, idVendor=0bc2, idProduct=2065, bcdDevice=20.04
[3482649.595480] usb 2-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[3482649.595483] usb 2-2: Product: Ultra Touch
[3482649.595485] usb 2-2: Manufacturer: Seagate
[3482649.595487] usb 2-2: SerialNumber: 00000000NADD0ZSK
```
As you can see, the disconnect happens, and then the HDD reconnects almost immediately. 

This eventually causes issues further down the line because the zfs pool does not expect a disconnect and faults with a I/O error. Sometimes, the device would reconnect to a different device number, such as from `/dev/sda` to `/dev/sdb`.

Here is another instance of the disconnect happening:
```
[Sat Jan 18 05:24:36 2025] usb 2-2: USB disconnect, device number 2
[Sat Jan 18 05:24:36 2025] sd 0:0:0:0: [sda] Synchronizing SCSI cache
[Sat Jan 18 05:24:36 2025] zio pool=hdd_pool vdev=/dev/disk/by-id/usb-Seagate_Ultra_Touch_HDD_00000000NADD0ZSK-0:0-part1 error=5 type=1 offset=270336 size=8192 flags=721601
[Sat Jan 18 05:24:36 2025] WARNING: Pool 'hdd_pool' has encountered an uncorrectable I/O failure and has been suspended.

[Sat Jan 18 05:24:36 2025] sd 0:0:0:0: [sda] Synchronize Cache(10) failed: Result: hostbyte=DID_ERROR driverbyte=DRIVER_OK
[Sat Jan 18 05:24:38 2025] usb 2-2: new SuperSpeed USB device number 3 using xhci_hcd
[Sat Jan 18 05:24:38 2025] usb 2-2: New USB device found, idVendor=0bc2, idProduct=2065, bcdDevice=20.04
[Sat Jan 18 05:24:38 2025] usb 2-2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[Sat Jan 18 05:24:38 2025] usb 2-2: Product: Ultra Touch
[Sat Jan 18 05:24:38 2025] usb 2-2: Manufacturer: Seagate
[Sat Jan 18 05:24:38 2025] usb 2-2: SerialNumber: 00000000NADD0ZSK
```

I confirmed there were no relevant logs before/after the disconnect by running `dmesg -T | grep -B 5 -A 5 usb`. 

## Possible Causes

### USB autosuspend feature
Various sources suggest that the issue could stem from the Linux kernel's USB autosuspend feature, which suspends USB devices after a period of inactivity to save power:
> - https://danielbrennand.com/blog/proxmox-fix-usb-disconnect/
> - https://www.reddit.com/r/Proxmox/comments/13xwtst/random_interruptions_of_usb_connections/
> - https://forum.proxmox.com/threads/proxmox-6-0-usb-devices-disconnect-and-reconnect-randomly.59507/


For my system, the following files were set to these values
- `/sys/bus/usb/devices/2-2/power/autosuspend`: `2`
- `/sys/bus/usb/devices/2-2/power/control`: `on`
- `sys/module/usbcore/parameters/autosuspend` : `2`

> `device/2-2` is the Seagate HDD.

Although the [Linux kernel documentation](`https://www.kernel.org/doc/Documentation/usb/power-management.txt`) says that `control` should prevent autosuspend, it seems that the feature is still active somehow. 

**Daniel Brennand** recommends adding `usbcore.autosuspend=-1` to the kernel boot parameters in `/etc/default/grub` followed by running `update-grub`. However, since my Proxmox does not boot from a grub bootloader, I instead updated the kernel boot parameters in `/etc/kernel/cmdline` in Proxmox[^1].

Indeed, after modifying the kernel boot parameters, the autosuspend feature was disabled and the following files reported the following values:

- `/sys/bus/usb/devices/2-2/power/autosuspend`: `-1`
- `/sys/bus/usb/devices/2-2/power/control`: `on`
- `sys/module/usbcore/parameters/autosuspend` : `-1`


## What else?

Despite disabling autosuspend, the HDD disconnected again after a few hours (at 5 AM). I suspect the HDD itself initiates the disconnection due to inactivity, as "power efficiency" is a key feature advertised on the product page.

<img src="/assets/images/2025-01-18-Random-HDD-Disconnect-Proxmox/image-1.png" style="width: 300px; object-fit: cover;"/>


For now, I have settled on a inelegant workaround by running a cron job every 5 minutes. This reads a random block from the HDD to keep it active. 

> `*/5 * * * * dd if=/dev/sda bs=4096 skip=$RANDOM count=1 status=none of=/dev/null`

Will this prevent disconnection? I don't know. I will report back in a month or so if it disconnects again.

In the meantime, I’d appreciate any suggestions from my readers on how to tackle this problem effectively. You can contact me using my details on my github page.


[^1]: https://forum.proxmox.com/threads/grub-parameters-when-using-proxmox-boot-tool-refresh.118649/
