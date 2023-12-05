---
title: "Homelab Part 1 - Setting Up Proxmox"
date: 2023-11-30 00:00
categories: ["Homelab", "Proxmox"]
tags: ["Proxmox"]
img_path: /assets/img/2023-11-28-homelab-part-1-setting-up-proxmox
---
## Overview
In this post we will walk through the process I took to setup Proxmox, a hypervisor that allows for the creation of virtual hosts, on a used Dell Optiplex 7050 SFF. This Proxmox instance will be the center of my homelab and host my servers for networking, security tools, red-team labbing, and much more. I chose a used Optiplex over traditional rack server because it was cheaper while still working for what I intend to do within my homelab. If my needs eventually exceed what the Optiplex can offer, I can always create another Proxmox node and cluster them together so the Optiplex can still be used. 

## The Hardware
The Optiplex I ordered only came with a 128gb SSD and 8gb of RAM, definitely not enough for the needs of my homelab so I needed to order some additional parts. The motherboard the Optiplex comes with supports a maximum of 64gb of RAM, so that is what I ordered. I also ordered a 2tb SSD that will be used as the main drive for the server. I also had a 1tb HHD laying around so I will be using that for any data that can afford to be on the slower HHD. Here's a list of the complete specs of the server:
1. CPU: i7-6700 @ 3.40GHz
2. 64 gb DDR4 @ 3200 MT/s
3. 2 tb SATA M.2 SSD
4. 1 tb SATA HHD

I also plan to virtualize pfSense to act as my router & firewall for the network. You can do this without a dedicated network card, but I decided to order a dedicated 1gb dual-port Intel NIC that will be passed-through to pfSense to avoid any problems that may arise from using a single port.

With everything but the HHD in the system, this is what it looks like:
![server1](server1.jpg){: .normal }  

With the HHD in place, this is what the server looks like all put together (without the side panel of course).
![server1](server2.jpg){: .normal }  

## Installing Proxmox
To install Proxmox to the server, I first flashed the ISO file provided by Proxmox onto a USB using Balena Etcher. From there, I booted up the server with the USB plugged in and selected it as the boot drive.

Once booted, we see the Proxmox install selection screen. I chose the graphical install.
![server1](setup1.jpg){: .normal }  

On the next screen, I was presented with the EULA for Proxmox. 
![server2](setup2.jpg){: .normal }  

After agreeing to the EULA, I was give the option to chose my install drive. I chose /dev/sdb as this is the 2tb SSD.
![server3](setup3.jpg){: .normal }  

Next up, I chose the country and time zone. For these, I went with USA and UTC as the time zone.
![server4](setup4.jpg){: .normal }  

On the next screen, I entered the password that I wanted to be my root password for Proxmox. This will be used to access Proxmox via the web UI. I also entered the email that will be alerted when there are critical notifications for Proxmox.
![server5](setup5.jpg){: .normal }  

Next up, I was prompted for the management interface, hostname, and network information for Proxmox. Since we'll be using the PCIe NIC for pfSense, I'll choose the motherbaord port for the management interface. For the hostname, I went with rainier.local. Lastly, for the network information I added an IP address that was in range of my current router's range, the current router's IP, and the DNS server's IP.
![server6](setup6.jpg){: .normal }  

After that, we see a review of the changes I made so far. Everything looked good, so I proceeded with the install.
![server7](setup7.jpg){: .normal }  

After the server restarted and Proxmox was installed, I was presented with a CLI giving the IP and port to access the Proxmox web UI at. From this point on we can manage Proxmox via the web UI and no longer need a monitor.
![server8](setup8.jpg){: .normal }  

On my PC, I headed to the IP and port where Proxmox is hosted. At first, I was met with a warning because Proxmox uses a self-signed certificate. Because I trust it, I proceeded past the warning.
![warning](1.png){: .normal }  
 
 Now I am prompted for the username and password. The username is root and the password is the one I setup in the install wizard.  
 ![auth](2.png){: .normal }  

After authenticating, I was met with a warning that I do not have a subscription plan. Since I don't plan on paying for a support plan, I simply ignore this notification. This will show up everytime logging into Proxmox.  
![auth](3.png){: .normal }  

Once logged in, I can now see the Proxmox dashboard! Proxmox is fully installed. Next we will provision the 1tb HHD so it can be used by Proxmox.
![auth](4.png){: .normal }  

## Provisioning Storage
To provision my 1tb HHD, I headed over to the Disks section on the dashboard. Here we can see the 2tb SSD, 1tb HHD, and the USB containing the Proxmox installer.
![auth](5.png){: .normal }  

On this screen we can see the health of our disks under the "S.M.A.R.T" column.

If we double click on any disks "S.M.A.R.T" column, we can see the scores for each test. This is where you can keep an eye out for incoming disk failures.
![auth](6.png){: .normal }  

Next, I'll go to Disks->ZFS to provision the 1tb HHD. After clicking the "Create ZFS" button, I was prompted to enter the information relating to the disk. Here I entered "puget" as the drive name, selected the 1tb HHD, and left the other options as their defaults.
![auth](7.png){: .normal }  

After provisioning the drive, I can now see it in the Disks->ZFS section.
![auth](8.png){: .normal }  

## Closing Thoughts
With Proxmox installed, accessible, and all drives provisioned I am all set to start creating hosts now. In the next post I will be setting up pfSense as a VM in Proxmox to replace my current SOHO router. pfSense will not only allow more custimization and stability, but I will also be able to segment my network with VLANs to enhance security.