---
title: "Homelab Pt 9 - Automating Security Operations with n8n"
date: 2024-7-20 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-07-20-homelab-pt-9-automating-security-operations-with-n8n
image:
    path: thumbnail.JPEG
    alt: Summit Lake, Washington
---
## Introduction
In this post I'll walk through standing up n8n, an open-source automation platform, that will serve as the SOAR for my environment. n8n will allow me to automatically enrich findings through threat intelligence integrations, manage incidents within my case management platform, update firewalls dynamically, and really any other action that can be automated. While there are specific platform integrations supported by n8n (Jira, CS Falcon, etc.), it also supports regular webhook APIs for any platform that doesn't have an official integration.

Once deployed, I will also create an end-to-end workflow for a subset of the Suricata rules I currently have running on my OPNsense firewall.

## Deploying n8n
The deployment of n8n is made very simple using containerization. n8n provides a Docker image available at docker.n8n.io/n8nio/n8n. To get n8n stood up using Docker, I'll first create a Docker volume so the n8n container can have an area for persistent storage. To do this, I'll using the following command:
```bash
docker volume create n8n_data
```

That command will create a directory at /var/lib/docker/volumes/n8n_data where n8n will store any data that needs to persist between restarts. From there, I'll create a n8n directory to store the docker-compose file for the n8n container and populate the compose file with the following:
```bash
sservices:
  n8n:                                  #Creates n8n Docker service
    image: docker.n8n.io/n8nio/n8n      #Pulls the latest image from n8n
    restart: always                     #Ensures the container will restart if the host system restarts 

    ports:
      -  "5678:5678"                    #Maps ports needed for n8n

    environment:
      - N8N_PORT=5678                   #Specifies which port to serve n8n on

    volumes:
      - n8n_data:/home/node/.n8n        #Mounts external Docker volume for persistent storage

volumes:
  n8n_data:
    external: true                      #Declares that n8n_data volume is external
```

This compose file will pull the latest n8n image, mount the volume I just created, serve it on port 5678, and ensure that the container will restart on system restart. With the compose file created, I'll use *docker compose up -d* to start the n8n container in the background.  

Once the Docker pulls its dependencies, builds the container, and starts up I can now access it via the web interface. On first visit, I am prompted to configure the n8n owner account.  
![diagram1](1.png){: .normal }  

After setting up the account, I am brought into the n8n dashboard and can start working on configuring workflows!  
![diagram1](2.png){: .normal }  

## Building a Workflow
For brevity's sake, I'll only build out a single workflow in this post. That being said, with the nature of how SOAR platforms work, there are all but endless options when it comes to what types of workflows you can create. The beauty of automation is that the sky really is the limit. Since I already have Suricata running on my OPNsense firewall with the ET Open rule sets, I'll be building out a workflow to for a specific subset of these rules. Specifically, the ET Open CNC (C2) rules will be the focus of the workflow I create today. These alerts are looking for traffic outgoing to known C2 servers, typically cause by an endpoint being compromised by malware. 

The main three points of focus for this new workflow are as follows:
* Automate enrichment of findings
* Automate opening of case within IR platform
* Automate paging of analyst

To enrich the findings, I'll be integrating with VirusTotal and GrayNoise for threat intelligence. I'm using DFIR IRIS as my IR platform, so I'll have hooks into it to manage cases. I'll also integrate Slack to serve as my paging mechanism for the on-duty analyst (Me in this scenario).

Once complete, the workflow will generally consist of the 4 phases:
1. Graylog (SIEM) detects ET Open CNC finding in the Suricata logs it's ingesting and sends data to n8n to kick off workflow.
2. n8n parses the information provided by Graylog and reaches out to VirusTotal and GrayNoise for threat intelligence on the C2 server detected.
3. n8n opens a case within DFIR IRIS with the original finding and supplemental threat intelligence data.
4. Page analyst via Slack with information on finding

From there an analyst is able to, hopefully, triage the finding much faster than they would be able to if they were going out to VirusTotal, GrayNoise, etc. and manually gathering intelligence to aid their investigation. If I were working with SLAs in my environment, the automatic paging of the on-duty analyst also helps to ensure they are met.

With the plan now set, I'll jump into building out this workflow.

### Triggering Workflow

### Enriching Findings

### Opening Incident Cases

## Closing Thoughts