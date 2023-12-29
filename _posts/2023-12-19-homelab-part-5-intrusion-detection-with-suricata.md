---
title: "Homelab Part 5 - Intrusion Detection with Suricata"
date: 2023-12-19 00:00
categories: ["Homelab", "Security"]
tags: ["VLANs", "Security", "IDS"]
img_path: /assets/img/2023-12-19-homelab-part-5-intrusion-detection-with-suricata
---
## Introduction
In this post, we will walk through configuring intrusion detection on my home network. In my case, the intrusion detection system (IDS) will monitor all my network traffic in real-time and provide alerts for any potentially malicious traffic it observes. It does this by comparing the traffic it sees to pre-defined rules. While there are paid rule sets that are updated faster and with better quality rules, I will just be using the free rule sets for my home network. I plan on letting the IDS run for a week or two and simply alert on traffic before switching it over to block mode. This will allow me time to suppress any false positives so my legitimate traffic is not blocked.

Since I am using pfSense as my router, any traffic within my home network goes through pfSense, so I'll want to use one of the tools available in the pfSense package manager for simplicity. While there are plenty of choices, Snort and Suricata are the most mature and well-documented of the options. They are both very similar in their capabilities in terms of detecting intrusions, so the only deciding factor for me was performance. I ended up going with Suricata because it supports multi-threading, while the version of Snort currently available through pfSense does not.

There is a point to be made about the effectiveness of IDS. For example, if you are a victim of a novel attack or even just one of the first to communicate with any new infrastructure the attacker can spin up at will, IDS won't do much for you. The infrastructure the attacker is using likely won't be on any of the rule sets the IDS is using because it literally has not been used in an attack up to that point. Since the infrastructure won't be on the rule sets, the IDS won't see the traffic as malicious and alert on it. That being said, threat actors are still human and re-use the same infrastructure quite often, so unless you are one of the unlucky ones that are a victim to an attack before the infrastructure is on the rule sets, IDS still provides some value for the low-hanging fruit.  

Let's get into setting up Suricata!

## Installing Suricata
The actual installation of Suricata on pfSense is very quick and easy. From pfSense, I'll head over to System->Package Manager.  
![diagram1](1.png){: .normal }  

Now if I head over to "Available Packages" and search for Suricata, I can see it as ready to download.  
![diagram1](2.png){: .normal }  

After clicking install I am prompted to confirm the install.  
![diagram1](3.png){: .normal }  

Once confirmed, Suricata will be downloaded and installed.  
![diagram1](4.png){: .normal }  

Now if I head to the Services menu, I can see Suricata has been added.  
![diagram1](5.png){: .normal }  

Now let's get Suricata configured and running.

## Configuring Suricata
First I'll choose what rulesets I'll want Suricata to trigger on. For now I will just be going with the following sets of rules, but there are other free rulesets and even paid rulesets available for use:  
- ETOpen Emerging Threats
- Feodo Tracker Botnet C2 IPs
- Abuse.ch SSL Blacklist

![diagram1](6.png){: .normal }  

I'll also choose 1 day as my update interval and 1 hour as my time to remove blocked hosts, even though I'll only be running Suricata in alert mode until I am confident no legitimate traffic will be blocked.
![diagram1](7.png){: .normal }  


After selecting and saving the rulesets, now I will head over to Suricata->Updates to download the rulesets. Before updating, here we can see that no hashes are available for each ruleset, so Suricata won't have anything to actually trigger on because the rules haven't been downloaded yet. Since I chose 1 day as my interval to update the rulesets, if I waited a day the rulesets would be downloaded but I will manually download them initially.  
![diagram1](7.1.png){: .normal }  

To manually download the rulesets, I'll click "Update". After a few seconds, I am now shown the MD5 hashes of the rulesets.  
![diagram1](7.2.png){: .normal }  

 Suricata now has the rulesets to alert on, and I'll just need to enable it on any interface I want monitored. To do this, I'll head over to Suricata->Interfaces->Add.  
 ![diagram1](8.png){: .normal }  

On the creation menu, I am prompted for which interface I would like to monitor. Since my network is segmented into various VLANs, I will need to create an interface monitor for each VLAN. First, I will create one for the default LAN.  
 ![diagram1](9.png){: .normal }  

After selecting the LAN, I am prompted for some general information about the interface monitor. I'll enter a quick description and leave everything else as default.  
 ![diagram1](10.png){: .normal }  

After saving the changes, the interface monitor is created. Now if I head into the interface monitor's categories I can choose which categories of the rulesets I have downloaded will be monitored on for that specific interface. I'll select all here.  
 ![diagram1](13.png){: .normal }  

With the rule categories selected, all that's left for Suricata to alert on this interface is actually enabling the monitor. Heading to Suricata->Interfaces, I can see the monitor is disabled by default.  
 ![diagram1](14.png){: .normal }  

After clicking the blue play button and waiting a few seconds, the interface monitor will be enabled and alerting.  
 ![diagram1](15.png){: .normal }  

Now I will copy this interface monitor and simply change the interface being monitored and description of each for all of my VLANs.  
 ![diagram1](16.png){: .normal }  

To look at the alerts coming in for my LAN interface, I'll head over to Suricata->Alerts->LAN. Here I can see I've already got a few informational alerts firing.  
 ![diagram1](17.png){: .normal }  

It's clear some of them I won't be worried about like the Steam user agent, Spotify P2P client, and Discord alerts. I'll condsider these false positives and will want to tune them out.

## Tuning False Positives
With any system alerting on pre-defined rules, there are bound to be false positives that you may not want to spend time triaging after you determine they are benign. In this scenario, I can tell Suricata that I no longer want to have this specific rule alerted on. For example, I know I am not concerned with users on my network using Steam because that is expected. To supress this rule so no more alerts are triggered, I've got the option to add the rule to the supress list.  
 ![diagram1](18.png){: .normal }  

After using that button to add the rule to the supress list, I can see the findings are marked as so. With this, I wont be alerted on these rules any more going forward. I'll do the same for the Discord and Spotify alerts that I am not concerned about along with any other alerts that I don't want to see. 
 ![diagram1](19.png){: .normal }  


## Monitoring WAN...for Science!
In this next section I will be enabling Suricata on my WAN interface. This is the interface where all internet traffic comes to enter my network and because of that it will be very, very noisy. Enabling Suricata on the WAN interface will show me all traffic even though the firewall is blocking it as my pfSense machine is not exposed to the internet. Since all types of scanning and monitoring occurs from the internet, there will be too much noise to actually be actionable but it should still be interesting to see what Suricata alerts on.

> Since the traffic volume on the WAN interface is so large, Suricata will still have to run on all the traffic for the interface. This means there will also be a significant cost associated with monitoring WAN traffic even with the little actionable information it provides. 
{: .prompt-danger }  

To monitor this traffic, I'll copy one of the existing interface monitors and change the interface to WAN.  
 ![diagram1](20.png){: .normal }  

After letting the monitor run for a few minutes I can already see some activity starting to be alerted on.  
 ![diagram1](21.png){: .normal }  

If we enter some of these IPs into GreyNoise to see if they are associated with any malicious activity we get some clarity. For example, the 167.248.133.163 IP appears to be related to Censys. Censys is a tool that does mass scanning on the internet so this makes sense they would be scanning my public IP.  
 ![diagram1](22.png){: .normal }  

However, if we take a look at the 43.129.39.176 IP we can see it comes up as not associated with any known vendor and appears to be mass scanning the internet as well. 
 ![diagram1](23.png){: .normal }  

GreyNoise also provides some insight into what type of attacks the host might be carrying out. This specific IP looks to be bruteforcing SSH & Telnet.  
 ![diagram1](23.1.png){: .normal }  

While this information is interesting, it isn't very actionable in terms of securing my home network because that traffic is blocked regardless. Because the information has limited value for my use case and the performance cost associated with monitoring the traffic is high, I'll be disabling monitoring on the WAN interface. 

## Triggering a True Positive
Since I haven't seen any true positives for actual malicious traffic from Suricata yet, I'll manually trigger one of the rules just so I can see what that might look like. To do this, I'll first need to find an alert that I will be able to trigger by looking at the actual rules in the rulesets I am using. The first one that shows should work fine. This alert triggers on outbound traffic to 178.128.23.9 on port 4125 over TCP.  
 ![diagram1](24.png){: .normal }  

To simulate this traffic and trigger the alert, all I need to do is nmap this host on port 4125. From the traffic Suricata is seeing, this matches up with the actual CnC traffic that would be malicious so we should see an alert.  
 ![diagram1](25.png){: .normal }  

Once the scan is complete, we do see that port 4125 is open runing Remote Web Workplace.
 ![diagram1](26.png){: .normal }  

Now if I head back to Suricata->Alerts->Interface, I can see the alert that was generated off the traffic. Suricata shows that a network trojan was detected and even tells us the possible malware family.  
 ![diagram1](27.png){: .normal }  

This is what a true positive would look like if Suricata were to alert on one.

## Closing Thoughts
Now that Suricata is setup and running on all VLAN interfaces I will monitor the alerts over the next week or two and be sure to tune out any false positive alerts so that when I do flip Suricata over to block mode, my legitimate traffic isn't blocked. I also plan on shipping the Suricata logs to a SIEM where I can search, parse, and alert on them much easier than in the pfSense web interface. For now however, Suricata is running and alerting on any potentially malicious traffic within my network.

Thanks for taking the time to read this post and feel free to reach out via LinkedIn if you have any questions!  