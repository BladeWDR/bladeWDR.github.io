+++
title = "Shut down your laptop automatically when clamshelled using systemd"
date = "2023-06-23T21:11:41-04:00"
author = "Blade"
authorTwitter = "bladewdr" #do not include @
cover = ""
tags = ["tutorial", "linux", "systemd", "logind.conf"]
keywords = ["", ""]
description = "Learn how you can use systemd to automatically shut down your Linux laptop when you close the lid"
showFullContent = false
readingTime = true 
hideComments = false
color = "" #color from the theme settings
+++


#### Set laptop to shut down automatically on lid close in Linux

This applies specifically to Linux systems running systemd - though that covers most distros currently available. I'm doing this on Ubuntu 23.04.

Some background - I currently work at a very small MSP in New York State as general IT tech / system admin / network engineer / floor polisher. As a part of my job I'm constantly in and out from different job sites, often requiring the use of my laptop.

I constantly run into the issue, as I'm sure many of you have, of my laptop simply having a dead or nearly dead battery after pulling it out of my go bag.

Frustratingly, most if not all manufacturers in recent years have not properly implemented sleep functionality, resulting in more battery drain while suspended.

I decided to look into how I could fix this problem by having my laptop automatically shut down when I clamshell it.

Unfortunately, GNOME as of time of writing does not have power management features built in to accomplish this task, so we must venture into the terminal.

To accomplish this task, we will be overriding the systemd-logind service configuration file `/etc/logind.conf`. You could also edit the file directly, but it's better practice to use an override, so that an update does not erase your changes by overwriting the file.

#### Create a new file using your text editor choice called `/usr/lib/systemd/logind.conf.d/overrides.conf`

    `sudo vim /usr/lib/systemd/logind.conf.d/overrides.conf`
  
#### Edit the file to contain these lines. The second line is optional.

	
    HandleLidSwitch=poweroff
    HandleLidSwitchExternalPower=ignore
    
    
##### Restart systemd-logind.service.  

> Note that the following will log you out!


```
sudo systemctl restart systemd-logind
```

#### Try it out

Log back in and clamshell your laptop. Did it shut down?


#### Troubleshooting

If, like me, you run into problems where your laptop is not following the behavior that you set, here are some basic troubleshooting steps.

1. Check for syntax errors. These files are case sensitive, did you spell things correct, do you have the correct case?
2. And the one that I personally ran up against - other programs inhibiting `handle-lid-switch`.
    * Run `systemd-inhibit --list`. Look for any programs listing `What` as `handle-lid-switch` or something similar to it. 
    * In my case, it was GNOME tweaks. I had set a preference in earlier attempts to resolve this problem to not suspend when clamshelling the laptop, and it was taking priority over my settings for systemd-logind.