---
title: "Homelab Pt 9 - Automating Security Operations with n8n"
date: 2024-7-12 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-07-12-homelab-pt-9-automating-security-operations-with-n8n
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
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always

    ports:
      -  "5678:5678"

    environment:
      - N8N_PORT=5678

    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
    external: true
```

This compose file will pull the latest n8n image, mount the volume I just created, serve it on port 5678, and ensure that the container will restart on system restart. With the compose file created, I'll use *docker compose up -d* to start the n8n container in the background.  

Once the container pulls its dependencies, builds the container, and starts up I can now access it via the web interface. On first visit, I am prompted to configure the n8n owner account.  
![diagram1](1.png){: .normal }  

After setting up the account, I am brought into n8n and can start working on configuring workflows!  
![diagram1](2.png){: .normal }  
