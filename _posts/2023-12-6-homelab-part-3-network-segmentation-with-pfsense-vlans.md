---
title: "Homelab Part 3 - Network Segmentation with pfSense VLANs"
date: 2023-12-6 00:00
categories: ["Homelab", "Networking"]
tags: ["Segmentation", "VLANs", "Security"]
img_path: /assets/img/2023-12-6-homelab-part-3-network-segmentation-with-pfsense-vlans
---
## Introduction
In this post, I will walk through segmenting my home network using VLANs. VLANs allow for the logical separation of hosts within a network so they cannot communicate with each other unless specified. This is useful in a number of ways such as segmenting my IoT devices on the network that I may not inherently trust so they cannot communicate with my trusted devices. Another example is if I plan on having a guest network, I can isolate any devices on that network so they can only communicate with the internet and no internal devices.  

The following diagram shows what my home network will look like once all VLANs are implemented.  
![diagram1](0.png){: .normal }  

Here some examples of what each VLAN will contain:
1. Management - Contains network management infrastructure (Router, Switches, APs, DNS server, Proxmox, etc.)
2. Security Tools - Contains any self hosted security tools (Wazuh, Graylog, OpenVAS, etc.)
3. Personal Devices - Contains personal PCs, laptops, phones
4. IoT - Contains any IoT devices (smart plugs, bulbs, TV, etc.)
5. Work - Contains work devices
6. Red Team Lab - Contains victim and attcker machine for red team activities
7. Guest - Contains any guest devices

Each VLAN will be using a /24 subnet as the 253 usable IPs will be more than enough for each VLAN while also leaving plenty of room for expansion if needed.

I'll be implementing both wired and wireless VLANs, and that breakdown is as follows:
1. Wired: Management, Security Tools, Red Team Lab
2. Wireless: Personal Devices, IoT, Work, Guest

One thing to consider is that for the most part, these VLANs won't be compltely seperated. For example, once I setup Wazuh the agents on all endpoints will need to communicate to the Wazuh server in the Security Tools VLAN over TCP 1515 to register and TCP 1514 for continued use. I'll also need a rule that will allow my host to access the Wazuh web interface over TCP 443. The idea isn't necessarily to completely block off every possible connection, but keep any open commnuication down to only what is needed. This will help reduce the risk of lateral movement if any given endpoint was to be compromised.

## What's Needed?
For my VLAN needs, I needed 3 main components to properly segment my network. First is a router that supports VLANs. Previously I was using a traditional SOHO router/switch/AP combo that only supported very basic functionality and didn't offer much in terms of security. In my last post, I covered the process of switching over to pfSense as my router & firewall. My pfSense instance is virtualized in Proxmox with its own deticated PCIe NIC being passed through to it. pfSense offers a variety of network customizability but the most important for the scope of this post is, of course, VLANs! 

The VLANs that I will be implementing only work because they are tagged at the traffic level. This means that even if my pfSense router is sending out tagged VLAN traffic, if the switch that traffic goes through doesn't support VLANs, the tag will be stripped and the VLAN won't be enforced. This is where having a managed switch that supports VLANs is needed. I picked up an 8-port TP-Link switch that supports VLANs for about $40 that I will be using in my home network.

Since I also plan on having wireless VLANs, I needed an access point that supports them. I was able to find a TP-Link EAP225 AP for about $50 that will be powered via PoE from the switch. 

With the entire network stack supporting VLANs (Router, Switch, Access Point), I am now able to segment my network with VLANs. Let's get started!

## VLANs with Proxmox
Since the Management VLAN inherenlty exists as the default LAN in pfSense, I don't actually need to create it. This leaves the only wired VLANs being Security Tools and Red Team Lab, both of which will only contain virtual hosts in Proxmox. This means there will be a couple of extra steps after we create the VLAN to ensure our Proxmox hosts get put in the correct VLAN but first let's create the VLANs.

First, we will start in pfSense by actually creating the VLAN itself. To do this, I'll head over to Interfaces->Assignments->VLANs. As we can currently see, there are none created yet.  
![diagram1](1.png){: .normal }  

After clicking "Add", I'm prompted for some information about the VLAN. The parent interface for all my VLANs will be lan. The VLAN tag will be what the traffic is tagged with to identify the VLAN. I'll be following the tagging convention of using the third octect of each VLAN range as the tag. For example, the Security Tools VLAN range is 10.0.10.1/24 so the tag will be 10, the Red Team Lab VLAN range is 10.0.50.1/24 so the tag will be 50, and so on. I won't bother with the VLAN priority for now. I'll also add a quick description of the VLAN. Here is the information for both the Security Tools and Red Team Lab VLANs.  
![diagram1](2.png){: .normal }  
![diagram1](3.png){: .normal }  

Now we can see both VLANs have been created.  
![diagram1](4.png){: .normal }  

Next up, we need to actually assign the VLANs to interfaces. I'll do this by going to Interfaces->Interface Assignments. There we can see the current WAN and LAN assignments that are already in place.  
![diagram1](5.png){: .normal }  

To assign each of our VLANs to their own interfaces, I'll select the dropdown menu for available network port, choose the VLAN, and click Add.
![diagram1](6.png){: .normal }  
![diagram1](7.png){: .normal }  

After adding both, we can see each VLAN now has it's own interface. 
![diagram1](8.png){: .normal }  

I am now able to click on each interface and change their settings. Let's do that for each VLAN now. Clicking on OPT1, we will be sure to enable it by checking the top box. We'll also need to set the IP range this VLAN will contain. As mentioned before, the Security Tools VLAN should be 10.0.10.1/24 so we'll select static IPv4 under the configuration type and enter the correct range. I'll also change the description to show what the VLAN is. Everything else will be left on default.  
![diagram1](9.png){: .normal }  

After saving the changes, I'll make sure to apply the changes as mentioned in the prompt.  
![diagram1](10.png){: .normal }  

I'll also make the same changes to the Red Team Lab VLAN interface.  
![diagram1](11.png){: .normal }  

With both VLAN interfaces now created and enabled, we need to enable the DHCP server on them so they can serve out IPs to their endpoints. This can be done in Services->DHCP Server. Here we will see a tab for each interface. 
![diagram1](12.png){: .normal }  

First I will enable DHCP on the Security Tools VLAN. After clicking on its tab, I can now make the needed changes. First I will check the box to actually enable DHCP. I'll also need to enter the correct addres pool for DHCP. Here I will enter the start of the range at 10.0.10.100 and the end at 10.0.10.200. This will give about 150 reserved IPs for anything that needs a static IP and leave 100 in the pool. Lastly, I'll enter the DHCP servers for this VLAN. I will be setting up an internal DNS server in the future, but for now I will just use 1.1.1.1 as the primary DNS and 8.8.8.8 as the secondary.  
![diagram1](13.png){: .normal }  

After saving an applying those changes, I will enter the relevant information for Red Team Lab VLAN as well.  
![diagram1](14.png){: .normal }  

That's all that is needed on the pfSense side. Now we will need to ensure our switch is capturing the VLAN tags properly. After heading to the switch's web interface, I'll go to VLAN -> 802.1Q VLAN.  
![diagram1](15.png){: .normal }  

With my current setup, pfSense is going into the switch on port 6 and Proxmox is plugged in on port 7. Since the two VLANs I have created will only pass between the router and the Proxmox endpoints, I only need to create tagging on these two ports. To add a tagging profile I will add the VLAN ID (Tag that I specified in pfSense), a name for the VLAN, and choose what ports it applies to.  

Here's the profile for the Security Tools VLAN. I had to shorten the name as the character limit is very small.  
![diagram1](16.png){: .normal }  

After adding that, I'll do the same for the Red Team Lab VLAN.  
![diagram1](17.png){: .normal }  

Now with the switch aware of the tagged traffic, I am good to start creating hosts in Proxmox that will fall within the correct VLANs.  

One thing to note is the Proxmox network interface must have "VLAN aware" enabled. This can be found in Node->Network->Interface.  
![diagram1](18.png){: .normal }  

First, I'll start by creating a host within the Security Tools VLAN. This is done by adding the VLAN tag in the VLAN tag field on the Network tab.  
![diagram1](19.png){: .normal }  

After creating and starting the host with the tag of 10 in the network settings, we can see it has been automatically placed in the 10.0.10.1/24 range! It looks like the Security Tools VLAN is working properly.  
![diagram1](20.png){: .normal }  

Now I'll create a host with the tag of 50 so it should be placed in the Red Team Lab VLAN. After the host starts up, we can see it was given an IP in the correct range of 10.0.50.1/24.  
![diagram1](21.png){: .normal }  

## VLAN Firewall Rules
By default, there are no firewall rules set for VLANs. This means that any traffic that attempts to leave the VLAN will automatically be blocked. For instance, if I try to ping the host I created in the Security Tools VLAN from the host in the Red Team Lab VLAN, it will be blocked.  
![diagram1](22.png){: .normal }  

This may be exactly what we want for internal traffic, but what if I want hosts in a certain VLAN to be able to reach the internet? By default, that is also blocked as that traffic would be leaving the VLAN. We can demonstrate this by pinging google.com from either of the hosts.  
![diagram1](23.png){: .normal }  

This traffic is being blocked because the host isn't even allow to reach out to the DNS server (1.1.1.1) to find google.com's IP. Let's change that and give the Red Team Lab VLAN the ability to communicate only to the internet. To do this, I will head back to pfSense->Firewall->Rules. As mentioned before, we don't have any rules in place by default.
![diagram1](24.png){: .normal }  

To only allow access to the internet we will need two rules. One will allow all traffic from the VLAN to anywhere, no exceptions. However, this rule also allows traffic to other internal VLANs which I don't want. To counter this, I'll make a rule that blocks any traffic from the VLAN going to an internal IP address. Let's create those rules.

Before actually creating the rules, I'll want to create an alias that contains all RFC 1918 private IP addresses. This will just save me time when wanting to specify all private IP addresses in the future. Instead of having to type them out each time, I can just select the alias. I'll create this alias in Firewall->Aliases.
![diagram1](25.png){: .normal }  


Now I'll create the rule that blocks inter-VLAN traffic first so I am not opening up any gaps. To create a rule, I'll just click "Add". For this rule the action will be Block and the interface will be the REDTEAMLABVLAN. I'll select any protocol and leave the source as any. For the destination, this is where I will specify the RFC 1918 private IP ranges by selecting the alias I just created.  
![diagram1](26.png){: .normal }  

After saving and applying the changes, we can now see the new rule! 
![diagram1](27.png){: .normal }  

Now that we are explicitly blocking internal traffic, we can create the rule to allow any traffic leaving the VLAN.
> If you are doing this, make sure the rule that allows all traffic outbound is BELOW the rule that blocks internal traffic. pfSense rules lists work top-down meaning a request will be filtered starting with the first rule down to the last rule. If no rules are matched, the traffic is blocked. If I were to put the rule allowing all outbound traffic on top of the rule that blocks internal traffic, a host could still communicate with other internal hosts because it would trigger the first rule and be allowed through.
{: .prompt-danger }  

I'll do this by clicking the "Add" button with the down arrow that will add the rule to the bottom of the list. The only thing I will change on the rule is the protocol to any and add a quick description.  
![diagram1](28.png){: .normal }  

Now our host in the Red Team Lab VLAN should be able to access the internet, but no other internal hosts. Let's try it out by pinging google.com.  
![diagram1](29.png){: .normal }  

We are getting a reply, perfect! However I still want to make sure I got the other rule right by pinging the host on the Security Tools VLAN. As we can see, it's still being blocked.  
![diagram1](30.png){: .normal }  

You may have noticed that with the internal blocking rule we put in place, not only will inter-VLAN traffic be blocked but also traffic to hosts within the same VLAN. This may not be what we want for some VLANs, and I know it isn't what I want for the Red Team Lab VLAN as I will have attacker and victim machines talking to each other inside the same VLAN. To fix this, I will add another firewall rule to the top of the rule list for the VLAN that permits traffic to any IP within 10.0.50.1/24.  
![diagram1](31.png){: .normal }  

To test that this rule actually works, I will spin up another host within the Red Team Lab VLAN. This new host's IP is 10.0.50.101.  
![diagram1](32.png){: .normal }  

If I ping that address from the other host in the same VLAN, we can see I am getting replies.  
![diagram1](33.png){: .normal }  

After creating these 3 rules, the Red Team Lab VLAN is now in the spot where I want it in terms of connectivity. Hosts within the VLAN can speak with each other, but not to other VLANs, and they have access to the internet.  

I'd like to make the same changes to to the Security Tools VLAN, but without the rule that allows intra-VLAN communication for now. To do this, I can just copy whatever rule I want and change the interface to SECURITYTOOLSVLAN.  
![diagram1](34.png){: .normal }  

Back on the host in the Security Tools VLAN, I am able to access the internet but not communicate with the hosts in the Red Team Lab VLAN, perfect!  ![diagram1](35.png){: .normal }  
![diagram1](36.png){: .normal }  

That wraps up the wired VLANs for the time being. Now I will setup the wireless VLANs.

## Wireless VLANs
I'll run through the steps for creating one of the wireless VLANs to show the process, but after that I will repeat the steps for the other wireless VLANs and not show the steps as they are exactly the same.  

Let's get right into it by creating the IoT VLAN. I'll start off in pfSense->Assignments->Interface Assignments->VLANs. Here we can see the two VLANs I created for Security Tools and Red Team Lab. 
![diagram1](4.png){: .normal }  

Since I am creating the IoT VLAN which will have the range 10.0.30.1/24, its tag will be 30.  
![diagram1](37.png){: .normal }  

Now heading over to pfSense->Assignments->Interface Assignments, I will create an interface for the VLAN.  
![diagram1](38.png){: .normal }  
![diagram1](39.png){: .normal }  

Clicking on the new interface, we can customize it as desired. I'll enable the interface, rename the interface, select static for the IPv4 type, and enter the 10.0.30.1/24 range.  
![diagram1](40.png){: .normal }  

After saving and applying the changes, I'll head over to pfSense->Services->DHCP Server and select the IoT Vlan. Here I will enable DHCP, give the IP pool range, and provide the DNS servers. I'll leave some space before and after the IP pool for reserved IPs.
![diagram1](41.png){: .normal }  

After saving and applying those changes, the VLAN is good to go from the pfSense side. Now I will need to create a tagging profile for the VLAN on the switch. Since pfSense is going into the switch on port 6 and the AP is plugged in on port 1, I will include port 1 and 6 in the tagging profile.  
![diagram1](42.png){: .normal }  

Now all that's left is to create a wireless network with the VLAN tag of 30 on it which will be done from the AP's web interface. Since it will just be IoT devices on this network, I will make it 2.4GHz. I'll add a password to the network and make sure it is broadcasting.  
![diagram1](43.png){: .normal }  

The network will now be availible to join, but since it isn't tagged with a VLAN tag it would jut put devices on the default LAN. To change this, I'll head over to Wireless->VLAN. Here I can enable VLAN tagging for individual wireless networks and enter the VLAN ID.  
![diagram1](44.png){: .normal }  

I'll join this network from my Windows PC just to verify I am getting placed in the correct VLAN. Here we can see the network available to join.  
![diagram1](45.png){: .normal }  

After connecting and running ipconfig, I can see I was given an IP within the correct VLAN, perfect!  
![diagram1](46.png){: .normal }  

Since this VLAN doesn't have any firewall rules yet, it won't be able to connect to the internet so I will copy over the rules I added to the Red Team Lab VLAN to allow internet access but not inter-VLAN communication.  
![diagram1](49.png){: .normal }  


Now we are able to ping google.com, cannot communicate with the Red Team Lab VLAN!  
![diagram1](47.png){: .normal }  
![diagram1](48.png){: .normal }  

That's about it for setting up the IoT wireless VLAN. Devices are now able to connect wirelessly and access the internet, but they aren't able to do any communication with devices on my network.  

After repeating the same steps for the other wireless VLANs, they are now all in place. After connecting to each of the wireless networks and confirming the right VLAN was served, all is working as expected.  
![diagram1](50.png){: .normal }  


## Conclusion
In pfSense->Interfaces->Interface Assignments we can see all the VLANs I have created. There will surely be more in the future, but this will work for a basic segmentation layout that enhances both the security and privacy of my home network.  
![diagram1](51.png){: .normal }  

Thanks for taking the time to read through this post and feel free to reach out via LinkedIn if you have any questions!