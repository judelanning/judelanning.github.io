---
title: "Homelab Pt 1 - Deploying OPNsense"
date: 2024-5-26 00:00
categories: ["Homelab", "Networking"]

img_path: /assets/img/2024-05-26-homelab-pt-1-deploying-opnsense
image:
    path: thumbnail.jpg
    alt: Rosario Beach, Washington
---
## Introduction
This is the first entry in, what I plan to be, a long-running series of blog posts documenting my efforts to build out my security-focused home network. My aim is to develop my security engineering skills by developing a comprehensive security ecosystem that parallels a proper enterprise environment. Throughout this journey, I'll be using open-source solutions when available for the goal at hand.

In this post, I'll be deploying OPNsense. OPNsense is an open-source, FreeBSD-based firewall and routing tool that will replace my SOHO router to allow connectivity for my devices to the internet and one another (if needed). Using OPNsense instead of a traditional consumer SOHO router will give me much more insight into what is occurring on my network, granular security controls, and increased reliability. 

## Why OPNsense?
Of course, OPNsense isn't the only solution that exists in it's area - there are Ubiquiti, Cisco, and Juniper products and the list goes on. So why OPNsense then? As mentioned before, I am looking to use open-source solutions when possible. This really narrows the list down to two major players: pfSense Community Edition and OPNsense. OPNsense is a fork of pfSense created in 2014. You can view the reasons Deciso, the company behind OPNsense, decided to fork pfSense at [this](https://docs.opnsense.org/history/thefork.html) page. Despite being a fork of pfSense, OPNsense only shares about 10% of the legacy code base with pfSense and has truly evolved into its own platform. 

The following components of the products themselves were key in deciding with OPNsense over pfSense:
* Weekly security updates for all components
* Has a REST API
* I prefer the OPNsense UI over pfSense UI

However, the behavior of Netgate, the company behind pfSense, also played a large part in my decision. There have been numerous examples of *interesting* actions taken by Netgate to say the least:
* Netgate setup a parody website mocking OPNsense using the opnsense.com domain. OPNsense was forced to appeal to the WIPO to get ownership of this domain. The parody website was archived by the Wayback Machine and can be viewed [here](https://web.archive.org/web/20160314132836/http://www.opnsense.com/).
* Publicly attacking a Wireguard dev via a [blog post](https://www.netgate.com/blog/painful-lessons-learned-in-security-and-community) and [mailing lists](https://lists.zx2c4.com/pipermail/wireguard/2021-March/006499.html).
* Since announcing pfSense+, a paid & close-source version of pfSense, pfSense CE has seen a decline in both major and point release frequency.

Those are just a couple examples in a sea of questionable behavior from Netgate, and that behavior is a big reason why I decided on OPNsense instead.

## Bare-metal vs Virtualized
OPNsense can be ran either virtualized or bare-metal. As I do have a Proxmox instance within my environment, virtualization is possible but I will be sticking with bare-metal for the simple reason that there are less points of failure. For example, if OPNsense is virtualized and my hypervisor goes offline for whatever reason (updates, crashes, resource over-allocation, etc.) then my network goes offline with it. With OPNsense running on bare metal there are simply less points of failure as there are less components involved, which in turn makes troubleshooting easier when failures do occur.

For my hardware choice, I went with a ProtectCLI Vault that has 8GB DDR3 RAM, 128 GB mSata SSD, and 4x 1GB ports. While not beefy by any means, this ProtectCLI vault will be more than enough for my homelab needs.


Outside of the firewall hardware, I've also got a TP-LINK EAP225 wireless access point and TP-LINK managed switch to allow for the connection of devices to OPNsense.

## ISP Modem Configuration
Before moving forward with installing OPNsense, I first need to change my ISP's modem to bridge mode. This will allow the modem to pass it's public IP to OPNsense, which it will use as its WAN interface. For the C6500XK, this was done in WAN Settings-> WAN Settings->ISP Protocol.
![](1.png){: .normal }    

## Installing OPNsense
After downloading the OPNsense ISO file from the OPNsense site and flashing it to a USB, I am now ready to install OPNsense. Booting from the USB on the ProtectCLI vault, I can see the system has booted into the live demo running off the USB. Since I want to install OPNsense, I'll authenticate as the installer user with the default credentials to begin installation.

The first step in the installation process is choosing the file system OPNsense will use. Here I'll choose UFS.
![](19.png){: .normal }    

Next I'll select the drive OPNsense should be installed on. Since the ProtectCLI Vault only has one internal drive, I'll be using that one.  
![](20.png){: .normal }    

After that I am prompted for the swap partition size - I'll leave it at the default 8 GB.
![](21.png){: .normal }    

Finally, I am prompted to confirm the changes and change the default password. After making these changes and waiting a few minutes, OPNsense is now up and running! 

## Configuring OPNsense
With the default OPNsense settings, none of the interfaces are configured as I'd like them so I'll do that now. For the time being, I'll simply have my WAN and LAN interfaces. To set those, I'll use the "Assign Interfaces" option and select the proper ports. 

```bash
igb0 -> WAN
igb1 -> LAN
igb2 -> None
igb3 -> None
```

I won't assign the other 2 ports on the device as they are not needed for a simple, flat network. 

After saving these changes and rebooting OPNsense, I can now see the default LAN IPv4 range and the public IP on my WAN interface.  
![](22.png){: .normal }    

Since I do plan on segmenting my network with VLANs in my next post, I'd prefer to not use the default IPv4 range. Instead, I plan on having various VLANs using the 10.0.0.0/8 range. As I don't expect to have anywhere near 255 hosts per VLAN, I'll keep these VLANs to /24 each. For the default LAN, I'll use 10.0.1.1/24. Using the "Set interface IP address" option, I'll make the following changes to the default LAN interface:
```bash
LAN CIDR: 10.0.1.1/24
DHCP Enabled
DHCP Start Address: 10.0.1.100
DHCP End Address: 10.0.1.255
```

Having the DHCP pool contain 10.0.1.100-255 will allow me to have a reserved space of 99 IPs (10.0.1.1-99) for any static IPs when needed.  

Now with the interfaces correctly assigned and configured, I can now plug my PC directly into the LAN interface to access the OPNsense web interface. When accessing the web interface, I am given a warning because OPNsense uses a self-signed certificate. As this isn't an issue, I'll proceed.  
![](6.png){: .normal }    

I am now met with the authentication page. Here I will authenticate using the credentials I configured earlier in the setup process.  
![](7.png){: .normal }  

Once authenticated, I am now brought to the setup wizard to finish the basic OPNsense installation. On the first page, I'll simply add 1.1.1.1 and 8.8.8.8 as my DNS servers. In a future post, I do plan to setup PiHole+Unbound as a recursive DNS server and ad-blocker so I'll need to change these settings once that is implemented.  
![](9.png){: .normal }  

For the time server, I'll leave it on UTC and the default NTP server.  
![](10.png){: .normal }  

On the WAN interface settings, I'll leave DHCP on as this is what my ISP uses to provided me with an IP. I'll also be sure enabling blocking of RDC1918 and bogon networks as they should never communicate with my WAN interface.  
![](11.png){: .normal }  
![](12.png){: .normal }  

Next, the LAN interface settings I configured were carried over. I'll simply confirm these settings and conclude the setup wizard.  
![](13.png){: .normal }  

As you may have noticed, I only thing I really changed with the setup wizard was adding DNS servers. That being said, my PC should now be able to resolve domain names and have full internet access. After confirming this with an nslookup and speed test, everything appears to be in order!  
![](15.png){: .normal }  
![](16.png){: .normal }  

## Configuring Switch & Access Point
Now that OPNsense is fully configured with the default network, all I need to do is configure the systems that will allow my devices to join that network.  

For the TP-Link switch I am using, I'll connect my PC directly to it and manually change my PC's IP address to something within the default range of the switch. After switching my PC's IP to 192.168.0.100, I can access the switch's web interface at 192.168.0.1. On the web interface, I'll assign the switch's IP address to 10.0.1.2, inside the same range that OPNsense is operating it's default LAN on.  
![](5.png){: .normal }  

Now, all I need to do is assign the TP-Link access point a proper IP address. After repeating the same steps as above, I'll set the AP's IP to 10.0.1.3, also within the same 10.0.1.1/24 that OPNsense is using for the default LAN.  
![](17.png){: .normal }  

Finally, I'll create a network on the AP that my wireless devices can use to connect to the LAN.  
![](18.png){: .normal }  

After connecting to this network with a wireless device, it can indeed reach the internet and gets the expected speeds.  
![](23.JPEG){: .normal }  

## Closing Thoughts
Now with OPNsense, a switch, and an access point configured I now have a simple, flat network up and running. Although this setup is as barebones as it gets, I do plan to make it much more advanced and secure in future posts by implementing VLANs for segmentation, IDS/IPS monitoring, and many more controls to ensure a comprehensive, layered defense. As it stands, my network matches the following layout:  
![](23.png){: .normal }  

Thanks for taking the time to read this post - feel free to reach out via [LinkedIn](https://www.linkedin.com/in/judelanning/) if you have any questions for me!
![human_badge](badge.svg){: .normal }{: width="125" }