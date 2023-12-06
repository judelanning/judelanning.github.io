---
title: "Homelab Part 2 - Switching to pfSense"
date: 2023-12-4 00:00
categories: ["Homelab", "Networking"]
tags: ["Proxmox", "Firewall", "Router"]
img_path: /assets/img/2023-12-4-homelab-part-2-switching-to-pfsense
---
## Introduction
In this post we'll cover the process of switching my home network's router to pfSense from a traditional SOHO router/switch/AP combo. I wanted to switch to pfSense to improve the customization, stability, and most importantly, improve the security of my home network. pfSense will allow me to segment my network with VLANs, do network activity monitoring with the help of Suricata, and much more!  

When setting up pfSense, you have two options: you can install pfSense on bare metal or virtualize it. Both options have their pros and cons, and I decided to virtualize it on my Proxmox server. The main drawback of virtualization is that if my Proxmox server is ever offline my devices won't have an internet connection because pfSense would also be offline. I decided I was willing to accept this risk because I won't be running anything that is mission critical on my home network so I can afford downtime to resolve any Proxmox issues if needed. Worst case, if the internet is down and the wife approval rating is at a 0, I can plug my old SOHO router in to stand up a temporary network.  

In this post, I will only walk through setting pfSense up with the default, single LAN but once I have the VLANs setup I expect my home network to look something like this:  
![diagram1](0.png){: .normal }  

## Creating the VM
Since my Proxmox server has a dual-port NIC that will be soley used for pfSense, I started by plugging in a network cable from my ISP modem to the WAN port on the NIC and a network cable from the LAN port on the NIC to a port on the TP-Link switch. This setup can be seen here:  
![diagram1](0.5.png){: .normal }  

At this point, because my WAN connection was being used for the pfSense NIC, I didn't have a router in-place. This means that I couldn't access Proxmox unless my PC's IP was manually set to same range as Proxmox. Here is what I saw when I tried to access Proxmox before changing my IP:  
![cantreach](1.png){: .normal }  

Using Windows settings, I changed my PC's IP to be in the same 192.168.0.1/24 range.  
![ip_change](2.png){: .normal }  

After saving the settings, my PC was given an IP in the correct range and I could now access Proxmox.
![ip_change](3.png){: .normal }  

After authenticating, I created the VM that pfSense will run in. On the general page, I named the VM "pfSense" and gave it a VM ID of 100. I kept the "Start at boot" option disabled for now, but will enable it later as we would want pfSense to start up whenever Proxmox does. I also added a couple of tags just to put some general information about the host's purpose.  
![general](4.png){: .normal }  

On the OS page, I selected the ISO file that I had downloaded from the pfSense website and uploaded to Proxmox.  
![general](5.png){: .normal }  

On the System page, I left everything at the default values.  
![general](6.png){: .normal }  

For the VM disks, I gave it 40gb and left everything else on the default settings. I plan on using Suricata on pfSense, so I plan on provising some extra space for that by carving out some storage from the 1tb HHD in the Proxmox system.  
![general](7.png){: .normal }  

On the CPU page, I gave the host all 8 cores available. CPU over-provisioning can be an issue if your hosts attempt to use all assigned cores at the same time, but my workload should be well under that so I am comfortable giving pfSense 8 cores. Everything else was left on default values.  
![general](8.png){: .normal }  

For memory, I assigned 8gb. Since FreeBSD, the OS that pfSense is based on, automatically takes any available memory, Proxmox will show 100% memory usage from pfSense at all times even when that isn't the case. I plan on using a different solution outside of Proxmox to measure host resource usage so this won't be an issue.  
![general](9.png){: .normal }  

I didn't assign a network device as I will be passing through the PCIe NIC once the host is created.  
![general](10.png){: .normal }  

The overview of the system looked good, so I went ahead and created the VM. Note: I didn't choose to start the VM yet as it doesn't have the PCIe device with the WAN & LAN links yet.  
![general](11.png){: .normal }  

## Troubleshooting IOMMU
After creating the VM, I went to go passthrough the PCIe NIC but was met with the following error saying IOMMU wasn't present.  
![general](12.png){: .normal }  

IOMMU is the system that allow for the passthrough of devices. In my case, I wanted to passthrough the PCIe NIC. I rebooted the Proxmox server and loaded up the BIOS to check if IOMMU was enabled. It turns out it was enabled, so there must be something else wrong.  
![general](iommu.png){: .normal }  

After a bit of digging through the Proxmox forums I discovered that I needed to add a string to the /etc/default/grub file to tell Proxmox that IOMMU was enabled. I went ahead and opened the file using nano (the best text editor) and added the following string to the file: GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on"  
![general](13.png){: .normal }  
![general](14.png){: .normal }  

After saving the changes, I ran the following command the let Proxmox see the changes: update-grub  
![general](15.png){: .normal }  

After a quick restart on the Proxmox host, I am now able to add both interfaces from the NIC to the VM. I assigned both and saved the changes.  
![general](17.png){: .normal }  
![general](18.png){: .normal }  

Now we can see both interaces under the VM's hardware devices.  
![general](19.png){: .normal }  

## Configuring the Modem and Switch
Before I booted up pfSense, I wanted to make a couple more changes to ensure everything will work properly. First, I needed to put my ISP's modem into bridging mode. This allows it to pass the public IP to pfSense so it can properly configure it's WAN interface. For the C6500XK, this was done in WAN Settings-> WAN Settings->ISP Protocol.  
![general](21.png){: .normal }  

Also, I changed the static IP of the TP-Link switch to be on the correct IP range. Since the switch's default IP is 192.168.0.1, I didn't need to change my PC's IP to access the web interface. Navigating to this IP with a web browser, I was able to see the login page.  
![general](22.png){: .normal }  

After logging in with the default admin/admin I was prompted to change the password. After that, I was met with the switch's dashboard.  
![general](23.png){: .normal }  

In the System->IP Setting page I disabled the switch's DHCP server and changed it's IP to be in the default LAN pfSense will manage. pfSense will be at 10.0.1.1, the switch will be at 10.0.1.2, and the access point which we will add in later will be at 10.0.1.3.  
![general](24.png){: .normal }  

With the modem and switch properly configured, I began the process of installing pfSense.

## Installing pfSense
To start installing pfSense, I booted up the VM I created earlier in Proxmox. 

Once the installer was loaded and I accepted the Copywright Notice section, I was met with the option of Installing pfSense, launching a rescue shell, or recovering from a config file. I chose instal pfSense.  
![general](26.png){: .normal }  

Next up, I went with the automatic partition as my use case doesn't call for any manual partitioning.  
![general](27.png){: .normal }  

For the ZFS configuration, I left everything on default values.  
![general](28.png){: .normal }  

I didn't go with any redundancy for the drive, so I selected stripe on the next page. I plan on regularly backingup the config file and if needed I will be able to restore from that one file.  
![general](29.png){: .normal }  

On the drive selection page, there is only one drive. It look longer than I'd like to admit for me to realize I needed to press space to select the drive.  
![general](30.png){: .normal }  

The installer gives us one last warning that the drive will be wiped. All good there so I continued.
![general](31.png){: .normal }  

After a short bit of time, the install is complete and I was prompted to reboot the machine.
![general](32.png){: .normal }  

## Configuring pfSense

Upon the VM rebooting, I am met with the pfSense screen showing me my public IP (redacted) and it's IP. Since I was previously in the 192.168.0.1 range, I needed to switch to the 192.168.1.1 range to access the web interface.  
![general](33.png){: .normal }  
![general](34.png){: .normal }  

After changing my PC's IP, I am now able to go to the pfSense web interace. Since pfSense uses a self-signed certificate I was met with a warning. Since I trust it, I proceeded.  
![general](36.png){: .normal }  

Now I was met with the login screen for pfSense. I authenticated with the default admin/pfsense credentials.  
![general](37.png){: .normal }  

Once authenticated, I am in! 
![general](38.png){: .normal }  

For best practice, I wanted to disable the admin account. Before doing this, I needed to create another account for myself so I can still access pfSense. I did this by going to System->User Manager->Users and creating an account for myself. 

> If you are doing this, be sure to actually add the admin role to your user account. I didn't see the "not member of" section underneath admin and figured my account would be an admin. After doing that and disabling the admin account, I was locked out of pfSense and had to re-create the VM.  
{: .prompt-warning }  

![general](39.png){: .normal }  

After creating the account and logging in with it, I both changed the password of the admin account and disabled the account.  
![general](40.png){: .normal }  

Looking at the users page, we can see my account is the only active account. Perfect!
![general](41.png){: .normal }  

Next up, I headed back to the setup wizard and walked through the steps.
![general](42.png){: .normal }  

For the hostname, I named it "pfSense". The domain is farily irrelevant, you just don't want to make it a domain unless you plan on actually using the domain. I chose "homelab.lan" as it isn't a real domain. For the DNS servers, I went with Cloudflare as the primary and Google as the secondary. I will be setting up PiHole for adblocking and as a recursive DNS server later on, so these will change but the public DNS providers work for now.  
![general](43.png){: .normal }  

I left the NTP server and time zone on the default values.  
![general](44.png){: .normal }  

For the WAN interace settings, I left everything on default.  
![general](45.png){: .normal }  

On the LAN interface settings, this is where I entered the default LAN range (10.0.1.1/24).  
![general](46.png){: .normal }  

Next up, I was met with a step that had me reload pfSense to accept the changes. However, since I made changed the pfSense IP to one that my PC's IP wasn't in the same range I needed to change my IP so that I could access pfSense.  
![general](47.png){: .normal }  
![general](48.png){: .normal }  

After doing that and heading back to the wizard, I see that the changes we all succesfully applied.  
![general](49.png){: .normal }  

After finishing up the setup wizard, I am brought to the main pfSense dashbaord.
![general](51.png){: .normal }  

Since pfSense will now hand me out an IP, I switched my PC back to get an IP from DHCP.
![general](50.png){: .normal }  

After switching to DHCP and running ipconfig /all, we can see I was given an IP from pfSense and have the correct DNS servers:  
![general](54.png){: .normal }  

Trying to ping google.com, I am getting a response. The LAN is up and working with internet access and will give out IPs in the 10.0.1.1/24 range for any devices plugged into the network!  
![general](55.png){: .normal }  

Doing a speed test, I can see I am getting the full 1gb speeds provided by my ISP.  
![general](56.png){: .normal }  

## Setting Up Wireless
Now that I've got the LAN working, I will setup my TP-Link access point to get wireless working. First, I'll plug the AP into the switch and I will get an IP from pfSense. After plugging the AP in, we can check which IP it was given in pfSense by navigating to Diagnostics->ARP Table. Since I know my PC has the IP ending with .10, we know the AP was given the .11 address.  
![general](58.png){: .normal }  

Heading over to that IP address, I can see the web interface for the AP. After logging in with the default admin/admin credentials, I am prompted to enter a new username/password.  
![general](59.png){: .normal }  

After creating new credentials, I am prompted for the wireless networks I want to create. For testing I will just setup one 5GHz network named "Gnomenet 5".  
![general](60.png){: .normal }  

Now I am prompted to connect to the wireless network and proceed.  
![general](61.png){: .normal }  

After proceeding, I am now able to see the main dashboard for the AP.  
![general](62.png){: .normal }  

Now under Management -> Network I'l change the AP to a static IP address and enter the relevant information.  
![general](63.png){: .normal }  

Once the settings are applied. The wireless network should be fully functional. If I connect to the network from my phone, I am given an IP within the DHCP range and can connect to the internet. Wireless is now working!
![general](phone.png){: .normal }  

Back in pfSense, we can see my phone's IP address showing up in the ARP table.  
![general](64.png){: .normal }  

## Adjusting Proxmox Host
Now that we've got both wired and wireless working, there is only one more change I need need to make. That change is putting Proxmox on the 10.0.1.1/24 range instead of the 192.168.0.1/24 range it is still on. To do that, I first manually changed my IP to be in the correct range so I can access Proxmox.  
![general](65.png){: .normal }  

Now I'll need to change some entires in /etc/network/interfaces. To do that, I'll open it up with nano.  
![general](66.png){: .normal }  

On the vmbr0 interface, I will change both the address range and gateway IP to match the lan range and pfSense IP respectively.  
![general](67.png){: .normal }  

I'll also change the entry in /etc/hosts following the same method.  
![general](68.png){: .normal }  

That's it for getting Proxmox on the correct range. Now I will switch my PC's IP back to DHCP.  
![general](69.png){: .normal }  

Once I am back on the 10.0.1.1/24 range, I will head over to the new Proxmox IP and ensure I can access the web interface.  
![general](70.png){: .normal }  

Now that I am logged back into Proxmox, the last thing I will do is check the box for starting up the pfSense VM on Proxmox startup.  
![general](71.png){: .normal }  

## Closing Thoughts
Now that I've got pfSense fully setup in my home network, the options are endless for where to go from here. In my next post I will be setting up VLANs to segment my network to match this diagram:
![general](0.png){: .normal }  

I also plan on running the Suricata pfSense plugin to monitor network traffic for any potential threats and creating a PiHole instance that will server as a network-wide adblocker and DNS server.

Thanks for taking the time to read this write-up!