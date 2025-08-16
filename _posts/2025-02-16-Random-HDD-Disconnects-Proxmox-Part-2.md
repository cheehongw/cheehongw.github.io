---
layout: post
title:  "Portable external HDD disconnects randomly on Proxmox - Follow up"
date:   2025-02-16 17:45:00 +0800
permalink: /blog/:title
---

Following up on my previous post on [random HDD disconnects on Proxmox](/blog/Random-HDD-Disconnect-Proxmox), I seem to have found a solution to the issue.

To recap, the issues was that my USB connected HDD would randomly disconnect and reconnect at random intervals. This was problematic for all sorts of reasons. For one, any zfs pool on the HDD would fault, necessitating a reboot. I am sure you can imagine how bad it is for your storage device to be disconnected mid-write/read.

The disconnects were entirely random, and an inspection of the `dmesg` logs showed absolutely no relevant logs before or after the disconnects.

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


In the previous blogpost and after, I tried the following solutions to resolve the disconnections, which did not work:
1. Disabling USB autosuspend (because I suspected it to be a power-saving issue)
2. Using a cronjob to constantly read the drive (to prevent it from sleeping)
3. Using a cronjob to constanly write to the drive (a even more aggressive approach to prevent sleeping)
4. Blacklisting `uas` (to force the drive to use `usb-storage` instead)

None of the above solutions worked. I still observed disconnect messages in the `dmesg` logs. In fact, for #3, constantly writing to the drive made the drive heat up considerably.


I also tried a different approach, which was to embrace the intermittent nature of USB storage. I decided ZFS on a usb device was a bad idea, and adopted a ext4, which was more tolerant of disconnects. Alas, the disconnects would remain and it still caused considerable issues. For example, if a mountpoint was configured between the drive and any LXCs, the LXCs would encounter an I/O error when trying to access the mountpoint after the drive reconnects, necessitating a reboot of the LXC to clear the fault.

At this point, I was going down a rabbit hole of "hot-swappable" USB solutions to drive this approach further. 

My fringe idea was to set up a NFS share on the Proxmox server, then somehow let my LXCs access the storage via NFS. I thought this might work since Network file sharing is inherently "unreliable" by nature and storage devices mounted via this approach might have built in tolerances or protocols to handle such unreliability, which seemed applicable to my usecase.

Alas, I might have found a solution to my problem - which is to use a different USB port 😂. Previously, the connection was 

```
USB-C (computer)  <---->   USB-C (HDD)
``` 
Now, it is:

```
USB-A 3.2 (computer)  <---->   USB-C (HDD)
```

I have no idea why this works, but it does. I have not observed any disconnects since I made the change.