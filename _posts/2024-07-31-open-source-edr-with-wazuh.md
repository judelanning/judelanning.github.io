---
title: "Homelab Pt 10 - Open-Source EDR with Wazuh"
date: 2024-7-31 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-08-01-open-source-edr-with-wazuh
image:
    path: thumbnail.JPEG
    alt: Spada Lake, Washington
---
## Introduction
In this post I'll walk through the process of deploying [Wazuh](https://wazuh.com/), an open-source endpoint detection & response (EDR) solution, within my home environment. I've currently got Suricata running on my OPNsense firewall to identify known-malicious traffic, but I don't really have any insight at the endpoint OS level. This is where a solution like Wazuh is helpful. 

With Wazuh, agents are deployed to endpoints which then send endpoint data to the Wazuh server for analysis.  If an alert triggers from the Wazuh server analysis, it is then stored and indexed within the Wazuh indexer. To wrap everything together, the Wazuh dashboard is the web user interface for data visualization and analysis. All of these components are well documented by Wazuh [here](https://documentation.wazuh.com/current/getting-started/components/index.html).

To deploy the Wazuh server, indexer, and dashboard I'll be using Docker containerization. Docker allows for the separation of Wazuh components from the infrastructure they run on such that they can run on any system where Docker is installed with the proper system specs. Docker also makes deploying and updating applications very simple. 

After Wazuh is deployed and my endpoints are enrolled, I'll gain a very important level of visibility into the activity occurring on my endpoints. This visibility will allow me to conduct threat hunts, detect malware, detect vulnerabilities, and [much more](https://documentation.wazuh.com/current/getting-started/use-cases/index.html) on my endpoints. Before this, however, I'll need to get Wazuh up and running.

## Docker Deployment

## Enrolling Agents

### Opening Firewall Rules

### Windows Endpoints

### Linux Endpoints