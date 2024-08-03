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

With Wazuh, agents are deployed to endpoints which then send endpoint data to the Wazuh server for analysis.  The Wazuh server will process the data to identify vulnerabilities, malicious software, suspicious user actions, etc. If an alert triggers from the Wazuh server analysis, it is then stored and indexed within the Wazuh indexer. To wrap everything together, the Wazuh dashboard is the web user interface for data visualization and analysis. All of these components are well documented by Wazuh [here](https://documentation.wazuh.com/current/getting-started/components/index.html).

To deploy the Wazuh server, indexer, and dashboard I'll be using Docker containerization. Docker allows for the separation of Wazuh components from the infrastructure they run on such that they can run on any system where Docker is installed with the proper system specs. Docker also makes deploying and updating applications very simple. 

After Wazuh is deployed and my endpoints are enrolled, I'll gain a very important level of visibility into the activity occurring on my endpoints. This visibility will allow me to conduct threat hunts, detect malware, detect vulnerabilities, and [much more](https://documentation.wazuh.com/current/getting-started/use-cases/index.html) on my endpoints. Before this, however, I'll need to get Wazuh up and running.

## Docker Deployment
The host I'll be deploying Wazuh on is running Ubuntu 22.04 virtualized on Proxmox with 4 CPU cores, 16 GB RAM, and 100 GB of storage. Docker and Docker Compose have already been installed, so I can jump right into the installation. Before this however, Wazuh recommends increasing the maximum amount of memory mapped areas that can be created on the host. To do this I'll add **vm.max_map_count=262144** into the /etc/sysctl.conf configuration file. After saving these changes and rebooting the host, I can confirm the changes stuck using **sysctl vm.max_map_count**. I do see the expected value, so the changes did indeed stick.  
![diagram1](1.png){: .normal }  

With that covered, I'll start the installation by cloning the Wazuh docker [repository](https://github.com/wazuh/wazuh-docker) to the host. Once the repo is cloned to my host, I'll navigate to the single node directory it contains. Wazuh does support a multi-node HA deployment, but I'll be sticking with single node for my use case. Within the single node directory, there's a script that will automatically generate certificates that the various Wazuh components will use to securely communicate with each other. Since I plan on using these self-signed certificates, I'll use the following command to generate them instead of providing my own with the following:
```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

This script creates the required certificates in the config/wazuh_indexer_ssl_certs directory. After running the script and enumerating the objects within that directory, I can see that the various certificates were indeed created without issue.  
![diagram1](2.png){: .normal }  

From here, I'll make a few changes to the docker compose file used by Wazuh to initialize the containers. The contents of the default compose file can be found in [Github](https://github.com/wazuh/wazuh-docker/blob/master/single-node/docker-compose.yml), but the idea is that the 3 main Wazuh components (Indexer, Manager, and Dashboard) are provisioned within this file. Of course I'll go through and change the passwords set in this fle, but I'll also change the port that the Wazuh dashboard is served on. By default, the dashboard is served on 443 like most HTTPS applications. However, I've already got another application running on this host using that port, so Wazuh will have to be reassigned. This is a very simple change - I'll literally just change the port number for the dashboard container on line 112 of the compose file to 4433 from 443.  
![diagram1](3.png){: .normal }  

With the default passwords changed and the dashboard port adjusted, I am now ready to spin up Wazuh. This can be done using **docker compose up -d** to start the containers in the background.  
![diagram1](4.png){: .normal }  

Once the compose command finishes, I can confirm the containers are running using **docker ps -a**.  
![diagram1](5.png){: .normal }  

With everything looking good, I'll head to https://SERVERIP:4433 to access the Wazuh dashboard. Since I'm using self-signed certificates I am warned that they're untrusted by my browser.  
![diagram1](6.png){: .normal }  

As this is expected, I'll bypass the warning and proceed. Finally, I am met with the Wazuh login screen!  
![diagram1](7.png){: .normal }  

After authenticating with the credentials I previously configured, I can now see the main Wazuh dashboard.
![diagram1](8.png){: .normal }  

There are some system-level events already populated just from Wazuh running, but the data that I really want to monitor is the data from my endpoints. This is where agents come into the picture. Wazuh agents are installed on each endpoint you'd like to have Wazuh coverage on, ideally all user endpoints and as much infrastructure as possible, and send various data from that endpoint to the Wazuh manager for analysis and indexing. From there alerts can be generated from findings within the data, threat hunts can be conducted, you can get reports on your fleet's baseline compliance, and so much more.

Of course, none of this data will be present without actually enrolling agents so now I'll work to get some endpoints enrolled.

### Opening Firewall Rules


## Enrolling Agents

### Windows Endpoints

### Linux Endpoints