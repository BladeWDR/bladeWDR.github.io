+++
title = "Geoip Blocking in Ubuntu 22.04 using iptables"
date = "2023-04-10T20:55:05-04:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["ubuntu", "iptables", "jammy", "firewall"]
keywords = ["ubuntu", "iptables", "jammy", "firewall"]
description = "How to enable Geo-IP blocking in Ubuntu 22.04 using iptables"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++


# Blocking connections based on geolocation in Ubuntu 22.04, using iptables and xtables-addons

### Steps:

First note: all of these steps assume you're logged in as root. Make sure you give it the 'ol `sudo su` if you're not already root.

### 1. First, update your system.

`apt-get update && apt-get upgrade`

### 2. Now install some dependencies.

`apt-get install libxtables-dev xtables-addons-common libtext-csv-xs-perl pkg-config`

Note: on older versions of Ubuntu, substitute `libxtables-dev` for `iptables-dev`.

### 3. Download xtables addon, untar and configure. Replace

Get the latest version of the xtables-addons package from here https://inai.de/files/xtables-addons/

```
wget https://inai.de/files/xtables-addons/xtables-addons-3.23.tar.xz
tar xf xtables-addons-3.23.tar.xz
cd xtables-addons-3.23
./configure
make
make install
```

### 4. Download and build DB-IP definitions.

```
cd geoip
./xt_geoip_dl
mkdir /usr/share/xt_geoip
./xt_geoip_build -D /usr/share/xt_geoip *.csv
```

Okay, we're ready to use the geoip matching in iptables.

### 5. Allow or deny traffic based on geolocation! 

My favorite way to do this is to have the default DENY policy, and whitelist only the countries I want to be able to access my servers. In my case, the US.

`iptables -I INPUT -m geoip --src-cc US -j ACCEPT`

Add a command at the top of your INPUT chain to allow responses from outgoing connections. (only required if you want to set your default policy to DROP)

This is because when you set the default policy to DROP, even the responses from outgoing connections that YOU made will get dropped.

`iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`

I'd make sure you set a rule to allow your ssh connection before changing the default policy (to avoid getting kicked out and not being able to get back in.)

`iptables -I INPUT -p tcp --dport 22 -m geoip --src-cc US -j ACCEPT`

Set the default policy

`iptables --policy INPUT DROP`

### 6. Install the iptables-persistent package to make your iptables rules survive reboots.

`sudo apt install iptables-persistent`

Don't forget that if you make a change later on, you'll need to save again!

`netfilter-persistent save`