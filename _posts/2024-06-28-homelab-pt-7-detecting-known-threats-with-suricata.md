---
title: "Homelab Pt 7 - Detecting Known Threats with Suricata"
date: 2024-6-28 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-06-28-homelab-pt-7-detecting-known-threats-with-suricata
image:
    path: thumbnail.JPEG
    alt: Middle Fork Snoqualmie Trail, Washington
---
## Introduction
In this post, I'll be deploying Suricata within my network. Suricata, in my use-case, will serve as an IDS/IPS running at the firewall level. I am running OPNsense as the open-source firewall on my network and it has [built-in](https://docs.opnsense.org/manual/ips.html) integration of Suricata into the platform. Using this integration, Suricata will monitor all traffic flowing through OPNsense, identify traffic that matches known malicious patterns, and alert on said traffic. 

In the scope of this post, I'll have Suricata running in IDS mode meaning it will simply alert on malicious traffic and not actually drop malicious traffic it identifies. This is because solutions like Suricata can, initially, be very noisy if not baselined correctly. To try and avoid this, I'll have Suricata running in IDS mode for a week or two, do any baselining if needed, and then turn on blocking mode to have a fully fledged IPS.

Today I'll be enabling and configuring Suricata within OPNsense, importing various alerts from open-source threat intelligence feeds, and then sending any alerts Suricata generates to my SIEM (Graylog) for further processing. 

## Signature vs Behavior Based Alerting
There is a point to be made about the effectiveness of Suricata in the constantly evolving threat landscape due to it's signature-based alerting. Suricata requires pre-loaded rules that define what it will determine as "malicious traffic". These pre-loaded rules are curated from IoCs of previous real-world attacks where the victim shared the details. Because of this inherent requirement, if you undergo a novel attack, which wouldn't have any Suricata rules created to identify it, then Suricata wouldn't alert on the traffic actually being malicious since it doesn't know to look for that pattern.

This is where NDR solutions like Corelight, Vectra, and Darktrace provide more comprehensive coverage when monitoring network traffic. Instead of using pre-defined, signature based rules to alert on malicious traffic, NDR tools look for anomalies based on behavior to identify malicious traffic. This is very similar to the distinction between anti-virus and EDR on endpoints. AV simply looks for known-bad signatures while EDR usually does that and much more by analyzing the behavior of resources on the endpoint. 

I am yet to find a good open-source NDR solution, so for now I'll be sticking with Suricata for detecting malicious network traffic. While it doesn't provide comprehensive cover whatsoever, it is much better than no solution at all. If, at some point in the future, I find a good open-source NDR solution I'll be sure to adopt it to ensure a layered defense where I can detect both known and unknown threats.  

## Configuring Suricata
To get Suricata up and running, I'll head to OPNsense->Services->Intrusion Detection->Administration. Here I'll configure the high-level settings for Suricata starting with enabling it, enabling syslog alerts (so I can send alerts to Graylog), and choosing the OPNsense interfaces to monitor. I'll choose all interfaces to ensure no traffic will go un-monitored.  
![diagram1](1.png){: .normal }  


## 