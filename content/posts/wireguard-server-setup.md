+++
title = "Setting up a Wireguard VPN server on Ubuntu 22.04"
date = "2023-06-27T14:39:45-04:00"
author = "Blade"
authorTwitter = "BladeWDR" #do not include @
cover = ""
tags = ["linux", "wireguard", "ubuntu", "vpn"]
keywords = ["linux", "wireguard", "vpn"]
description = "How to set up your own wireguard server on Linux."
showFullContent = false
readingTime = true 
hideComments = false
color = "" #color from the theme settings
+++

I recently went through the process of setting up my own Wireguard VPN server up in a cloud server.

You can do this on any Ubuntu server with internet access, as long as you have the ability to open ports on the firewall, but I do recommend setting this up at a cloud hosting provider such as Linode or Digital Ocean.

The lowest tier of server will do - Wireguard is a pretty light VPN to run. I'm using a Linode "Nanode" plan, which is one shared CPU core and 2GB of RAM. Plenty enough for our purposes.

I won't go over how to create the server and connect to it over SSH, as there are plenty of guides on that topic already. Come back when you're logged into the server over SSH.

##### 1. First things first - updates! 

`sudo apt update && sudo apt dist-upgrade`

You can reboot at this step, or wait, since we'll need to reboot in a minute anyway.

##### 2. Edit `/etc/sysctl.conf` and enable ipv4 forwarding

This is needed for Wireguard to be able to pass the traffic on and act as an intermediary between you and the sites you're trying to route to through the VPN.

1. `sudo vim /etc/sysctl.conf` or `sudo nano /etc/sysctl.conf`
2. Uncomment the line that says *net.ipv4.ip_forward=1*
3. Reboot if you have pending kernel updates or run `sysctl -p`

##### 3. Allow port for the Wireguard server in your firewall of choice

Once your server is up, log back in. Now we'll need to open the port that Wireguard will be listening on.

I use port 444 but the default is 51820.

You can use either one - 444 is in the well-known ports range, but it's for a pager service that I hadn't heard of until I googled it. I think I'm safe.

Here's how to open the port with iptables:

`sudo iptables -I INPUT 1 -p udp --dport 444 -j ACCEPT`

Install the iptables-persistent package to make the rule survive reboots.

`sudo apt install iptables-persistent`

Save your rules. (You'll need to redo this every time you make a change to your iptables rules.)

`netfilter-persistent save`

Here's how to do the same thing with ufw:

`sudo ufw allow 444`

> ***Remember that if you have an upstream firewall (i.e. your router at home - you'll need to forward the ports there too!***

##### 4. Generate the keys for the Wireguard server. 

```
cd /etc/wireguard
sudo wg genkey | tee privatekey | wg pubkey > publickey
```

This will generate 2 files in /etc/wireguard.

You can check their contents by running the `cat` command.

`cat /etc/wireguard/privatekey`

`cat /etc/wireguard/publickey`

Remember to keep your private key *private*. It's like a password.

Keep these handy, we'll need them for the next part.

##### 5. Create config file for the server.

Pick an address range for your wireguard tunnel. It should be a private IP address range, but it can be anything you want.

I chose 10.200.0.0/24.

I've provided an example configuration below.

Open `/etc/wireguard/wg0.conf` with your favorite editor and paste it in. 

Remember to enter the private key you generated here earlier, and watch out for trailing whitespaces on the end of the key!

Example server config:

```
[Interface]  
Address = 10.200.0.1/24  
SaveConfig = true  
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; iptables -A FORWARD -o %i -j ACCEPT  
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; iptables -D FORWARD -o %i -j ACCEPT  
ListenPort = 444  
PrivateKey = privatekeyhere
```

##### 6. Bring up the Wireguard tunnel

`wg-quick up wg0`

##### 7. Create client config

On mobile and windows generally the private and public keys will be created for you. On Linux, just re-run the same commands you used to generate your server keys.

For cleanliness you can create a `/etc/wireguard/clients` subdirectory to hold all of your keys. 

Example Client config:

```
[Interface]
PrivateKey = clientprivatekey
Address = 10.200.0.2/24
DNS = 9.9.9.9    

[Peer]
PublicKey = serverpublickey
AllowedIPs = 0.0.0.0/0 #can change this depending on if you want full tunnel or split tunnel.
Endpoint = public-ip-of-server:444
```

For the `Endpoint` field you can either use the public IP of the server, or a DNS name that you've created for it. Remember to specify the port.

For the `PublicKey` field - this is the public key that you generated earlier on the *server*.

For `DNS` it can be any DNS server you like. I usually set my Pi-Hole servers at home as my DNS on the tunnel to provide adblocking.
For this example, I've just set it to use Quad9.

For `AllowedIPs` - set this to 0.0.0.0/0 only if you want to route *all* traffic through the VPN tunnel. If you only want to use this as a remote access VPN, then leave it as only the Wireguard subnet. 

`AllowedIPs = 10.200.0.0/24`

##### 8. Configure the server to bring up wireguard's interface on boot 

On the server, set the tunnel to come up automatically after a reboot.

`systemctl enable wg-quick@wg0`

##### 9. Add your new wireguard peer

Now it's time to add the client that you set up as a peer.

Wireguard uses public and private key pairs to authenticate peers, so you'll need the public key of your chosen client to enter below.

`wg set wg0 peer publickeyhere allowed-ips 10.200.0.2`


Once you have this set, it's a good idea to restart wireguard to make sure it knows about the new peer.

`sudo wg-quick down wg0`

`sudo wg-quick up wg0`

Now, you should be able to see it handshake.

`sudo wg show`

If you don't see a handshake happening - try sending some traffic across the tunnel.

`ping 10.200.0.2`

That should be it! If you set this up as a full tunnel, try going to a site such as [dnsleaktest.com](https://dnsleaktest.com/) and make sure it shows the IP of the wireguard server rather than your actual public IP.

If you set it up as a split tunnel, see if you can access resources on the other side of the tunnel.

##### 10. OPTIONAL - set up your router as a Wireguard peer! 

If you'd like to use your wireguard server to route traffic back to your local network, it's pretty simple to do so with firewalls like PfSense and OpnSense, that have Wireguard support baked in.

I'm using OpnSense, so I'll provide those steps here.

**Steps for setting up opnsense as a client**

1. Install the `os-wireguard-go` plugin. You can find it under System > Firmware > Plugins.
2. Enable wireguard. VPN > Wireguard.
3. Configure the remote server as an Endpoint. The PSK field is optional for more security and can be left blank. Enter a name, the public key for your wireguard server, and the endpoint address and port.
4. Configure the "Local" server. Generate key pairs for it, pick an IP in your Wireguard subnet. Add the remote endpoint as a peer. `wg set wg0 peer publickeyhere allowed-ips 10.200.0.3`

Next we'll need to add routes to tell Wireguard where to find your local subnets.

##### Adding routes

In the wireguard server config (/etc/wireguard/wg0.conf in this case):

Add each subnet you want to go through the VPN tunnel to the allowed IP list for that peer. 

**Example**


```
[Interface]
Address = 10.200.0.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; iptables -D FORWARD -o %i -j ACCEPT
ListenPort = 444
PrivateKey = privatekeyhere 

#config for opnsense peer
[Peer]
PublicKey = publickeyhere 
AllowedIPs = 10.200.0.3/32, 192.168.1.0/24 #192.168.1.0/24 being a subnet on the local network for this peer.
```

Test it out! If you're not using the full tunnel option on your client (0.0.0.0/0 for AllowedIPs) you'll need to add the local subnet to your allowed IPs list there, too.


