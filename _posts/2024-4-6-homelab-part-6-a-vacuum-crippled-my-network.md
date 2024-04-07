---
title: "Homelab Part 6 - A Vacuum Crippled My Network"
date: 2024-4-6 00:00
categories: ["Homelab", "Networking"]
tags: ["Firewall"]
img_path: /assets/img/2024-4-6-homelab-part-6-a-vacuum-crippled-my-network
---
## Introduction
Earlier this morning, just as my wife and I had finished doing various chores around the house, I heard 4 words echo across the living room that struck fear down my spine.

**"The WiFi isn't working!"**

Of course, I quickly checked the network settings on my phone to verify and sure enough, the various SSIDs I had broadcasting were missing. Next I checked my personal desktop computer, which has a wired connection to my network, and also saw no network connection at all. Gnomenet had seemingly vanished from the time that we started chores to when we wrapped them up.  

> Yes, my network's name is Gnomenet. Yes, all of our guests access my network via the "Gnomenet Guest" SSID. Yes, my wife despises this name. No, the name will not be changed any time soon. 
{: .prompt-tip }  

The next hour or so of troubleshooting would prove to demonstrate the need for a proper server rack with a redundant power source, expose configuration flaws within my homelab, and even conclude with an unlikely villain (Hint: Its *always* DNS). 

## Background
If you haven't seen my previous posts explaining my homelab, its stupid basic. I have a Dell Optiplex 7050 PC handling the virtualization. This machine has been upgraded to 64GB of DDR4 RAM along with a dual-port gigabit network card. This network card is solely used by pfSense, which is virtualized in Proxmox on the same host. From there, pfSense connects to a TP-Link 8-port PoE managed switch. Also connecting to that switch is a TP-Link wireless access points that broadcasts SSIDs for each wireless VLAN, the Proxmox host, and my personal desktop computer. 

This physical setup can be seen here:
![diagram1](1.jpg){: .normal }  

If you noticed the mess of cables occurring under the desk, you may have also noticed that everything in the setup connects to a single power strip. This power strip is of course a single point of failure. If the strip loses power, the Proxmox host loses power as well, and because pfSense is virtualized on Promox, pfSense goes too. All of that to say if the power strip loses power, all devices within my network lose the ability to talk to any other devices, including both internal devices and the internet. You might be able to guess what caused the outage.

## Troubleshooting
After realizing that the network was indeed completely offline, it was time for some troubleshooting. Starting from the bottom up, the first question I asked myself: "Is it plugged in?". The answer being: no actually, it wasn't. 

When vacuuming the room, my wife must have pushed the power strip into the wall, flipping the power switch to off. I flipped the switch back to on and turned on the Proxmox machine. After a minute or two, I saw the wireless SSIDs re-populate the available networks on my phone. I figured everything had come back online and all was good. I let my wife know the internet should be working again. About 30 seconds later, she called out:

**"It's not working - it won't let me connect."**

I went to try connecting on my phone and had the same issue. I could see all the SSIDs just fine, but wasn't able to authenticate to any of them. Looking at my PC, still no signs of a wired network connection. It seemed as though the TP-Link AP was working just fine and broadcasting the SSIDs, but pfSense still was having issues. After giving it a few minutes to see if pfSense was just taking a very long time to start up but still getting the same issue, I decided to check Proxmox for issues. 

Because my PC, which is the main way I administer my homelab, wasn't able to communicate with the Proxmox host, I turned to the physical host for answers. Using the *qm list* command, I can see all of the VMs living within Proxmox. pfSense, the VM in question, was not running even though I knew I had the "Start at Boot" option enabled in Proxmox. This VM should have started as soon as Proxmox booted.  
![diagram1](2.png){: .normal }  

Regardless, it needed to be started now so I attempted to do that using *qm start*. However, instead of the host starting up, I was met with an error. Likely the same error that led to the pfSense instance not running at startup.  
![diagram1](3.png){: .normal }  

Proxmox warned me that the instance could not be started as it was missing an ISO file that it depended on. This ISO was the installer that I used to install pfSense to the instance. A couple weeks back I was looking to clear out some un-needed ISOs taking up space on Proxmox and figured this one wasn't needed as I had already installed pfSense. This slip-up wasn't discovered until today because pfSense hadn't been rebooted in the 2 weeks since that change occured. 

Seeing as this ISO was long gone from Proxmox and I had deleted it from my PC as well, I headed over to the pfSense downloads to grab it. However, as my PC couldn't reach the internet I had to connect it to my hotspot from my phone to download the ISO. Making my way to the archive, I was able to find and download the correct ISO. 
![diagram1](4.png){: .normal }  

This ISO was downloaded as a GZ zipped file, so I had to use gunzip to get the actual ISO itself.  
![diagram1](5.png){: .normal }  

Now with the ISO file on my PC, I was ready to upload it to Proxmox and give booting up pfSense another go. However, there is one glaring issue - my PC still can't communicate with any internal instances because pfSense is down. To work around this, I would have to manually set my IPv4 settings for my PC's network interface to be in the same subnet as Proxmox, then connect my PC to the Proxmox host via ethernet. 

In Debian, the configuration file for the network interfaces is stored at /etc/networks/interfaces. Just in-case I were to royally mess up the config file, I created a backup of it in the same directory by using the following command:  
```shell
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```  

Now, I had the actual interfaces config file, along with the interfaces.bak file I could revert to if anything were to go wrong.
![diagram1](6.png){: .normal }  

To manually set my IPv4 assignment, I changed the interface entry from using DHCP to static, and added the relevant information for assignment.  
![diagram1](7.png){: .normal }  

To apply these changes, I ran the following command to restart the system networking:  
```shell
sudo systemctl restart networking
```  

Using *ip address*, I could see that my PC now had the IP address of 10.0.1.100, which is within the 10.0.1.0/24 range Proxmox resides in. This paired with a direct connection to the Proxmox host, I would be able to access the Proxmox web interface and upload the pfSense ISO.
![diagram1](8.png){: .normal }  

Indeed, after heading to my Proxmox web interface, I can see it!  
![diagram1](9.png){: .normal }  

After authenticating and heading to Proxmox->Datacenter->Primary Node->local->ISO Images, I uploaded the ISO file without issue.  
![diagram1](10.png){: .normal }  

Now that I've got the ISO uploaded and am already in the Proxmox web interface, I tried starting up the pfSense instance - and it worked! After a couple of minutes, I was brought to the main pfSense terminal menu letting me know that pfSense was up and running. 
![diagram1](11.png){: .normal }  

With pfSense running, I switched my interface configuration to use DHCP by deleting the /etc/network/interfaces file and renaming the backup config file to interfaces. I also ran the systemctl command to grab the new configuration. After changing the ethernet cables to their previous ports, my PC was finally given an IP address from pfSense's DHCP server. With this assignment, my PC was now able to access the Proxmox and pfSense web interfaces without issue as expected. Even further, the mobile devices using a wireless connection were also able to authenticate and have IPs assigned as well. However, there was still one issue - access to the internet was seemingly still broken. Even though the devices were given IPs and could communicate to other internal devices, where firewall rules allowed it, they still couldn't make Google searches, play YouTube videos, or stream music. 

I switched over to my Windows machine to see if it was having the same issues as well and sure enough, it was. I decided to see if I could ping google.com and got no replies. However, when I pinged 1.1.1.1 I got results instantly. 
![diagram1](12.png){: .normal }  

That's right, DNS was also broken since the Proxmox reboot. It turns out that when setting up PiHole & Unbound as my recursive DNS server, I had forgotten to enable the "Start at Boot" option for the instance so the instance simply wasn't running. Seeing as every VLAN within my network only points to the internal DNS server, these requests could not be fulfilled and gave the appearance of the internet traffic not working, when it was really just the DNS. After discovering this, I headed over to Proxmox and started the DNS instance along with enabling "Start at Boot".  
![diagram1](13.png){: .normal }  

Shortly after the instance was running, I tried to ping google.com again and was successful. The network was finally up and running again!  
![diagram1](14.png){: .normal }  

## Closing Thoughts
This troubleshooting endeavor, which had me scratching my head at times, was a valuable exercise in recovering critical infrastructure of my network after an outage. It also shed light on the fact that I should at least get a UPS for the assets I really don't want losing power, such as the Proxmox host and other networking equipment. But above all, this experience reinforced the cliche that it is *always* DNS. 