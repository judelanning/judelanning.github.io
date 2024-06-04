---
title: "Homelab Pt 2 - Network Segmentation with OPNsense and VLANs"
date: 2024-5-30 00:00
categories: ["Homelab", "Networking"]

img_path: /assets/img/2024-05-30-homelab-pt-2-network-segmentation-with-opnsense-and-vlans
image:
    path: thumbnail.JPEG
    alt: Hoh Rainforest, Washington
toc: true
---
## Introduction
Today, I'll walk through the process of segmenting the network I setup in my last post. I'll be doing this by using OPNsense to create usage-specific VLANs. This will allow me to limit how & what devices within my network communicate to other devices. For example, if I don't want my IoT devices to communicate with my personal devices, that can be done through VLANs. I'll start with the default position of not allowing any inter-VLAN traffic, except allowing the Management VLAN (10.0.1.1/24) to communicate with all VLANs as it is needed for OPNsense to function correctly.

Once complete, my network will be divided into the following VLANs:  
* Management (10.0.1.1/24) - Used by network management devices (OPNsense, switches, access points, etc.)
* Security Tools (10.0.10.1/24) - Used by hosted security tools (EDR, SIEM, SOAR, Bitwarden, etc.)
* Personal Devices (10.0.20.1/24) - Used by personal devices (Phones, laptops, PCs, etc.)
* IoT (10.0.30.1/24) - Used by IoT devices (Smart lights, networked cameras, smart TVs, etc.)
* Applications (10.0.40.1/24) - Used by hosted applications (Plex, Gitlab, Portainer, etc.)
* Work (10.0.50.1/24) - Used by work devices
* Guest (10.0.60.1/24) - Used by guest devices

![](1.png){: .normal }

Of course, as the network continues to grow and evolve, I may need to allow certain inter-VLAN traffic. For example, I plan to setup PiHole & Unbound as an ad-blocking, recursive DNS server. Once that is deployed, the endpoints on my network will need to connect to the DNS server which would be in the Management VLAN. By default this would be blocked, so I would need to add firewall rules to each VLAN that *explicitly* allow communication to the DNS server on UDP/TCP 53. 

Now let's jump into actually getting this segmentation created!

## VLANs in OPNsense
To create the VLANs, I'll head to OPNsense->Interfaces->Other Types->VLAN and click the "+" button to add a new VLAN. Here I'll be using the third octet of each VLAN's CIDR for its name and VLAN tag. 

Here is a table showing the information I'll enter for each VLAN:  
![](3.png){: .normal }

Using this information, I'll create each VLAN. Here is an example of the information I'll enter to create the security tools VLAN:  
![](2.png){: .normal }

For now, I'll leave the priority for each VLAN at the default. I also plan on using igb1 as the parent interface for all VLANs. After creating all VLANs, I can now see that they are all present and ready to be assigned to their respective interfaces interface.  
![](4.png){: .normal }

Heading over to OPNsense->Interfaces->Assignments I can see all the VLAN interfaces that need to be assigned, but they cannot be assigned as they are not initiated (shown by the red plug, I will get an error if I try to assign them in this state).  
![](5.png){: .normal }

I'll simply restart OPNsense to initialize these interfaces. After a reboot,  I can see they are green and ready to be assigned.  
![](6.png){: .normal }

I'll go through and assign each VLAN to an interface by going through each VLAN and clicking "add".  
![](8.png){: .normal }

Now that each VLAN has an interface created and assigned to it, I can go through and configure each interface with the proper settings. I'll only show the configuration of the Security Tools VLAN, but the settings for the rest are exactly the same just with their different CIDR ranges.

To configure the Security Tools VLAN I'll head to OPNsense->Interfaces->SECURITYTOOLS. By default new interfaces aren't enabled so there isn't much to configure.  
![](9.png){: .normal }

After checking the "Enable Interface" box, there are now a plethora of options to configure. For the time being, I'll only assign a static IPv4 range to this interface. As mentioned above, the range for the Security Tools VLAN will be 10.0.10.1/24 so I'll enter that and save the changes.
![](10.png){: .normal }

Next, I'll need to setup DHCP for the SECURITYTOOLS interface to allow IPs to be assigned to new endpoints on the interface. To do this, I'll head to OPNsense->Services->DHCPv4->SECURITYTOOLS. For the DHCP address pool I'll use 10.0.10.100-254 so there are just under 100 reserved IPs at the beginning of the range in-case they are needed. I'll also use 8.8.8.8 and 1.1.1.1 as the DNS servers, but I do plan on hosting Pi-Hole & Unbound in the future so these will be changed when that is deployed.  
![](11.png){: .normal }

Finally, the last step I need to do on the OPNsense side is setup firewall rules to allow traffic as needed. As mentioned above, by default I'll start with the position of allowing traffic to the internet but not inter-VLAN traffic. With the SECURITYTOOLS interface being newly created, there are no firewall rules and any traffic will simply be blocked by OPNsense.  
![](12.png){: .normal }

Since firewall rules are evaluated top to bottom I plan on having the following 3 rules to meet the requirement mentioned above:
1. Allow traffic to 10.0.10.1/24 (Allows intra-VLAN traffic)
2. Block traffic to bogon networks (Blocks traffic to all 10/8, 192.168/16, etc.)
3. Allow traffic to anywhere

There are ways of writing rules that could get this done in less rules, but to keep it simple I broke the default position into 3 different rules that each have an explicit, simple purpose. These 3 rules will allow traffic to hosts within the same VLAN, block traffic to hosts in other VLANs, and allow access to the internet (in that order). Because these rules are stateful by default, I don't need to worry about any rules in the other direction currently.

Once created, those 3 rules can be seen here:  
![](13.png){: .normal }

After repeating the process above for all other VLANs, the work is done in OPNsense and now I simply need to configure my switch and access point to tag VLAN traffic correctly. 

## Switch VLAN Configuration
Over on the TP-Link switch, I'll head to VLAN->802.1Q VLAN and set the proper settings. For the VLANs where I want wireless endpoints connecting, I'll tag port 1 (where the access point is connected) and port 5 (where OPNsense is connected) with the respective VLAN tag. For the VLANs where I expect wired traffic I'll tag port 5 (where OPNsense is connected) and whatever port the endpoints will be connected to. As of now, all my other wired endpoints will be hosted in Proxmox, which is connected on port 8. Currently, I don't plan on having both wireless and wired endpoints reside in the same VLAN, so I don't need to worry about tagging ports for both scenarios in one VLAN. After entering the information, these are the settings:  
![](14.png){: .normal }

Now, any wired endpoints that are connected and have the proper VLAN tag configured will be assigned to their respective VLAN. All that's left to do is create the wireless networks on the access point to allow endpoints to connect via WiFi.

## Access Point VLAN Configuration
Similar to the interface configuration in OPNsense, I'll only show setting up one wireless network but the steps taken for the other networks are the exact same except different VLAN tags.

To create a wireless network I'll head to Access Point->Wireless->Wireless Settings. Here I'll configure the wireless network:  
![](15.png){: .normal }

Once the network is created, I'll head to Access Point->Wireless->VLAN, enable VLAN for the network I just created, and enter the VLAN tag for the IoT VLAN.  
![](16.png){: .normal }

To ensure everything is working as expected I've connected a few devices to the IoT wireless network. On one device, I can see it was assigned an IP within the VLAN range (10.0.30.103) and has the correct DNS servers. This means the VLAN tagging and DHCP settings on the IOT interface are all working correctly. 
![](17.png){: .normal }

Pinging another device on the IoT VLAN, I do get a reply as expected.  
![](18.png){: .normal }

However, when I try to ping an internal device outside of the IoT VLAN, I am unable to (also as expected).
![](19.png){: .normal }

Finally, I am able to communicate to the internet.  
![](20.png){: .normal }

With the firewall rules, DHCP server, and VLAN tagging all in working order the VLAN is now correctly configured.  

## Closing Thoughts 
With VLANs now configured, I can isolate devices I may not inherently trust on my network from communicating with trusted devices, limit the spread of a potential malware infection in my network, and management my network on a much more granular level. With the VLANs in-place, my network follows this layout:  
![](1.png){: .normal }

In my next post, I plan on deploying PiHole & Unbound as an ad-blocking, recursive DNS server to enhance the privacy and security of users on my network.

Thanks for taking the time to read this post - feel free to reach out via [LinkedIn](https://www.linkedin.com/in/judelanning/) if you have any questions for me!