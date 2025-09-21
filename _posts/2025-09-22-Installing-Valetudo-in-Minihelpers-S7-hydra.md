---
layout: post
title:  "Installing Valetudo in Minihelpers S7 hydra"
date:   2025-09-22 12:00:00 +0800
permalink: /blog/:title
---

## Backstory

I have a robot vacuum cleaner that stopped performing its scheduled cleaning tasks. The cause? - The manufacturer's cloud service was no longer available.

The robot's model is the **Minihelpers S7 Hydra**. Minihelpers is most likely is likely a Singaporean / Asian brand that sells OEM rebranded vacuums (much like what Prism is for its monitors and SecretLab for its chairs). It was getting some hype in Hardwarezone forums back when I bought it in 2020 on Qoo10.

The S7 Hydra got very little use when it stopped working. I was staying on campus and my parents did not know how to operate it's app and I did not want to set up the daily cleaning lest no one at home knew how to troubleshoot it. But recently, I decided to just set up the robot for daily scheduled cleaning and let the people at home adapt to its schedule instead. 

To manage the robot, you had to download the Minihelpers app and create an account. Then, you would login with the account to manage the robot. When the robot stopped performing its scheduled tasks and I couldn't log in to manage the robot, this was when I realized that the manufacturer's cloud service was no longer available.

And indeed, it appears that Minihelpers has gone out of business? The last I checked its [Shopee store front](https://shopee.sg/minihelpers#product_list), it was active 2 months ago! 

Given that the robo vacuum had very little use until recently, it would be such a waste to replace a perfectly fine device just because the OEM went out of business. I decided to see if I could install [Valetudo](https://valetudo.cloud/), an open-source firmware for robot vacuums that allows local control without the need for cloud services.

## Compatibility with Valetudo

Valetudo works on the Minihelpers S7 Hydra. The robot vacuum is essentially a `3irobotix CRL-200S` inside, with a `RTL8821CS` WiFi module. I confirmed this by looking at the firmware under `/lib/modules/wifi/` and seeing the `8821cs.ko` module.

As mentioned by Valetudo's supported robot page, the `3irobotix CRL-200S` is actually made by `3irobotix` and is sold under many different brands, including: Xiaomi Vacuum-Mop P, Mijia STYJ02YM, Viomi SE, Conga 3290 and more. 

The pictures and similarity of the `3irobotix CRL-200S` clones to the Minihelpers S7 Hydra are striking. From the [rooting guide of the CRL-200S](https://github.com/Hypfer/valetudo-crl200s-root), the location of the USB port, screws, and wheels are very similar to the Minihelpers S7 Hydra. Furthermore, this explains how a relatively no-name brand like Minihelpers can have the manufactering expertise to create a robot vacuum cleaner under its own name.

With this knowledge, the installation process is as straightforward as following Valetudo's guide on installing Valetudo on the `3irobotix CRL-200S` clones. I'll leave a abridged version of the steps I took here, just to leave my thoughts and augment what is already known and available online.

## Installation

### Step 1: Prepare the necessary tools and files
- `adb` shell
- [dustbuilder](https://builder.dontvacuum.me/) firmware mentioned in step 5 of valetudo's guide. You want to prepare this beforehand since it may take a while to get the firmware.
- micro USB cable

### Step 2: Connect the robot and gain adb access
- The S7 Hydra appears to disable adb access by default. As documented in the Valetudo guide, you'll need to follow the steps to enable adb access. This involves running a `enable-adb.sh` script provided by the Valutudo guide.
- In my case, I connected the robot to my laptop and noticed the following behavior:
  - When the robot was powered off, connecting the USB caused the robot to occasionally boot into FEL(?) mode. We can see this in the `dmesg` log that a new USB device is connected. We can also occasionally see the device show up in `lsusb`.

  - To get adb access, run the script, and then power on the robot by pressing the power button. The script should be able to detect the brief moment when the robot boots up and gain adb access. If you miss the moment, just follow the steps mentioned in the script, or power off the robot and try again.
- Once you have adb access, you can run `adb shell` to get a shell on the robot.

### Step 3: Gain root access (skipped)
- I skipped this step since there was no password when I did this.

### Step 4: Take backups
- Follow the Valetudo guide to take a image of the the robot's nand partition. In my case, there was `nanda` till `nandi`. `cat` the `/proc/partitions` file to see the partitions available.
- I have no idea how to restore from this backup, and the Valetudo's guide doesn't seem helpful in this regard. I apologise for this.
- At this point, I guess I have to point out that you should prepare to lose your robot if something goes wrong.

### Step 5: Install dustbuilder firmware and install Valetudo
- Again, follow the Valetudo guide. 
- From how I understand it, the dustbuilder firmware converts the robot to a `Viomi SE6` clone, which is supported by Valetudo. The subsquent steps involve installing the Valetudo service which gives the robot its local web interface.

### Step 6: Enable 8821cs wifi module.
- From the Valetudo guide, it explicitly mentions additional model-specific steps for some of the models with the RTL8821CS wifi module. Again, I can confirm that the S7 Hydra falls under this category, so I followed the steps to enable the wifi module.

### Step 7. Start using Valetudo
- After running `reboot`, press and hold the power + home button on the S7 Hydra until the robot informs you that WiFi has been reset.
- After doing so, you can connect to the robot's WiFi AP (or more colloquially, the robot's hotspot). After doing so, follow the Valetudo guide to add your home wifi.
- For me, the robot refused to display/scan for Wifi networks at this point. I manually provisioned my SSID and password in the robot's web interface, and it connected to my home wifi without issues.

### Step 8: Using Valetudo
- When I started using Valetudo, I found out that the robot's map was still present from the previous Minihelpers app, which was a nice surprise. I could see the rooms and no-go zones that I had set up previously.

