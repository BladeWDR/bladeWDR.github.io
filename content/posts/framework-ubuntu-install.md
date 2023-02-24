---
title: "Framework Ubuntu Install"

date: 2023-02-23T22:15:57-05:00
url: /framework-ubuntu-install/
image: images/2020-thumbs/framework-ubuntu-install.jpg
categories:
  - Linux
tags:
  - Ubuntu
  - Pop_OS
  - Framework
draft: false
---
<!--more-->

# How to configure Framework laptop for Ubuntu based OS
Created: 2023-02-23 21:57

The below is stolen from the official guide provided by framework on their website - cited below.

```
# This is for 12th Gen ONLY.

# Copy the entire text block below, paste into the terminal, press enter and type your

user's password as prompted.

# This will:

# - Update your Ubuntu install's packages.

# - Install the recommended OEM kernel.

# - Workaround needed to get the best suspend battery life for SSD power drain.

# - Disable the ALS sensor so that your brightness keys work.

# - Enable improved fractional scaling support for Ubuntu's GNOME environment using Wayland.

# - Enable headset mic input.

## *****COPY AND PASTE THIS CODE BELOW*****

sudo apt update && sudo apt upgrade -y && sudo apt-get install linux-oem-22.04 && echo "options snd-hda-intel model=dell-headset-multi" | sudo tee -a /etc/modprobe.d/alsa-base.conf && gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']" && sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT.*/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash module_blacklist=hid_sensor_hub nvme.noacpi=1"/g' /etc/default/grub && sudo update-grub &&

echo "[connection]" | sudo tee /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf && echo "wifi.powersave = 2" | sudo tee -a /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf

## *****COPY AND PASTE THIS CODE ABOVE*****

## then press enter key, password, reboot.

# If you would rather enter the commands individually instead:

## Updating packages.

sudo apt update && sudo apt upgrade -y

## Install the recommended OEM kernel.

sudo apt install linux-oem-22.04

## Enable headset mic input.

echo "options snd-hda-intel model=dell-headset-multi" | sudo tee -a /etc/modprobe.d/alsa-base.conf

## Enable improved fractional scaling support for Ubuntu's GNOME environment using Wayland.

gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"

## Disable the ALS sensor so that your brightness keys work, 12th gen only.

sudo gedit /etc/default/grub

## Append the following to the GRUB_CMDLINE_LINUX_DEFAULT="quiet splash section.

GRUB_CMDLINE_LINUX_DEFAULT="quiet splash module_blacklist=hid_sensor_hub"

## Workaround needed to get the best suspend battery life for SSD power drain.

sudo gedit /etc/default/grub

## Append the following to the GRUB_CMDLINE_LINUX_DEFAULT="quiet splash section.

GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvme.noacpi=1"

# Sudo with your fingerprints.

## To run sudo in a terminal with the fingerprint reader, you need to run this command in a terminal and follow the prompts.

sudo pam-auth-update

## Also, if you've previously enrolled fingerprints in Windows or another Linux distro, you may find that fingerprint enrollment errors until you manually force clear the stored fingerprints.

https://knowledgebase.frame.work/en_us/fingerprint-enrollment-rkG6YP7xF

## Additional ways to extend battery life can be found at this link: https://community.frame.work/t/linux-battery-life-tuning/6665.
```

#### For distros other than Ubuntu you may also need to disable panel screen refresh
Only use this if you run into freezing / stuttering issues.
I ran into this on Pop_OS 22.04 specifically.

`sudo kernelstub -o "i915.enable_psr=0"`


## Links 
https://guides.frame.work/Guide/Ubuntu+22.04+LTS+Installation+on+the+Framework+Laptop/109
https://www.reddit.com/r/framework/comments/10zeoyu/anyone_experiencing_random_freezing_on_pop_os_2204/

