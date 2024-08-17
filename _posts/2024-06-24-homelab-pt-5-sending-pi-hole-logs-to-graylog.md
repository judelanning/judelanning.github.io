---
title: "Homelab Pt 5 - Sending Pi-hole Logs to Graylog"
date: 2024-6-25 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-06-24-homelab-pt-5-sending-pi-hole-logs-to-graylog
image:
    path: thumbnail.JPEG
    alt: Granite Lake, Washington
---
## Introduction
In this this post I'll walk through the process of sending logs from my Pi-hole DNS server to my SIEM (Graylog). With my current setup I have every VLAN in my network configured with its own DHCP server, which all hand out my Pi-hole server (10.0.1.53) as the primary DNS server. For the endpoints that have static IP addresses, I also manually enter my Pi-hole server as the main DNS server. This means that, unless Pi-hole is down, it should be handling any DNS resolutions coming from the endpoints on my network.

Seeing as threat actors can abuse DNS to act as a C2 channel, exfiltrate data, and infiltrate data these logs may prove useful when conducting an investigation into a compromised endpoint. Unit 42 has a great write-up on the various ways threat actors utilize DNS [here](https://unit42.paloaltonetworks.com/dns-tunneling-how-dns-can-be-abused-by-malicious-actors/#). Outside of security relevance, being able to track DNS traffic within my network will also prove helpful during any potential troubleshooting that may involve DNS.

This process will include creating an input within Graylog where Pi-hole will send the logs, configuring Pi-hole to send the logs to that input, and then extracting relevant information from the logs using extractors within Graylog.

## Graylog Input
To create the input to receive the logs I'll head to Graylog->System->Inputs and select "Syslog UDP" as the type then launch the input.  
![diagram1](1.png){: .normal }  

Once launched, I am prompted to enter the relevant information for the input. Here I'll name it, have it listen on 1502, and leave the other settings as default.  
![diagram1](2.png){: .normal }  

After saving these settings, I can now see the input is created and active, waiting for logs to be sent to it.  
![diagram1](3.png){: .normal }  

Now I'll switch over to my Pi-hole server to configure rsyslog.

## Sending Logs
To send the Pi-hole logs to the input I've just created, I'll create a file in the /etc/rsyslog.d directory with the proper configuration so rsyslog knows where to send logs. For my use case, I'll use the configuration provided by /u/Techy-Stiggy/ [here](https://www.reddit.com/r/pihole/comments/1cmy773/comment/l5y5r7a/). Of course, I've added the information specific to my enviornment (Graylog server address, listening port, and protocol type). 

```shell
*.* action(type="omfwd" target="10.0.10.10" port="1502" protocol="udp"
action.resumeRetryCount="100"
queue.type="linkedList" queue.size="10000")
# Define extra log sources:
module(load="imfile" PollingInterval="30")
input(type="imfile" File="/var/log/pihole.log"
Tag="pihole"
StateFile="/var/spool/rsyslog/piholestate1"
Severity="notice"
Facility="local1"
reopenOnTruncate="on")
input(type="imfile" File="/var/log/pihole-FTL.log"
Tag="piFTL"
StateFile="/var/spool/rsyslog/piFTLstate1"
Severity="notice"
Facility="local1"
reopenOnTruncate="on") 
```

After creating the rsyslog configuration file, I'll simply restart rsyslog using the following command:
```shell
sudo service rsyslog restart
```

After restarting rsyslog and heading over to Graylog I can see that we have DNS logs coming in from Pi-hole! Here is an example of Pi-hole forwarding a DNS request to Unboud to be resolved.
![diagram1](4.png){: .normal }  

As we can see, the important contents of the log are all contained within the "message" field. Ideally, I'd like to extract the relevant information into their own fields such as: action (query, forward, reply, etc.), queried domain, returned value, etc. To extract this information, I'll need to implement extractors on the Graylog input where the logs are being ingested.

## Parsing Logs
Heading over to Graylog->System->Inputs->Pi-hole Input, I'll create a few extractors. The data points I'll be extracting into their owns fields are dns_action (reply, forward, etc.), dns_domain (the domain tha is attempting to be resolved), dns_returned_value_prefix (from, is, etc.), and the dns_returned_value (IP address, CNAME, etc.). When the Pi-hole host is sending over actual DNS query logs they follow this format:
```shell
#Log Format
hostname hostname Date Time dnsmasq[756]: dns_action dns_domain dns_returned_value_prefix dns_returned_value

#Actual Log Sample of Pi-hole Resolving Query:
pihole pihole Jun 26 16:59:03 dnsmasq[756]: reply a1992.dscd.akamai.net is 104.80.88.74

#Actual Log Sample of Endpoint Making DNS Query:
pihole pihole Jun 26 16:59:03 dnsmasq[756]: query[A] catalog.gamepass.com from 10.0.1.100
```

This being said, DNS query logs are not the only logs that are sent over from Pi-hole - there are also various system level logs that will not contain relevant information that shouldn't parsed. To ensure Graylog isn't processing these logs and attempting to extract the relevant fields when they aren't present, I'll add a condition to the extractors on this input to only parse the logs if the "dnsmasq" string is present within the log which limits the scope to just the DNS logs. With this in mind, I'll use the following regex patterns to extract the desired information:
* dns_action: (?:\S+\s+){6}(\S+)\s+ (Extracts data between 6th and 7th spaces)
* dns_domain: (?:\S+\s+){7}(\S+)\s+ (Extracts data between 7th and 8th spaces)
* dns_returned_value_prefix: (?:\S+\s+){8}(\S+)\s+ (Extracts data between 8th and 9th spaces)
* dns_returned_value: \s+(\S+)\s*$ (Extracts data after last space)

After creating and applying these extractors, I can now see the fields are successfully being extracted from the main log!  
![diagram1](5.png){: .normal }  

## Closing Thoughts
From here, I can now perform various analysis and aggregation with the DNS data now being sent to Graylog. In addition to this, when building out my SOAR workflows I'll be able to cross-reference the DNS traffic when investigating anything DNS related. Outside of security relevance, this data will also prove helpful if I am ever needing to do any troubleshooting that may involve DNS.

Thanks for taking the time to read this post and feel free to reach out via LinkedIn if you have any questions!
![human_badge](badge.svg){: .normal }{: width="125" }