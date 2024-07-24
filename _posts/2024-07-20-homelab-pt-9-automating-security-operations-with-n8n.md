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
Since I've already got my Suricata logs being shipped  to and parsed by Graylog (covered in [this](/posts/homelab-pt-7-detecting-known-threats-with-suricata) post), getting these findings to kick off an n8n workflow is relatively simple. To start, I'll create a new workflow within n8n and select the event that will trigger the workflow. While there are app integrations under the "On app event" section, Graylog doesn't have an integration. Instead, I'll stick with the basic webhook call which will require a bit more work than an integration, but that's the fun of it!  
![diagram1](3.png){: .normal }  

After selecting webhook call as the trigger, I am now prompted to customize the node that will be listening for the trigger. For this workflow, Graylog will be the resource sending POST requests to the node with the relevant information. Here I'll name the node, select POST as the type, and ensure that the workflow will respond to the POST request immediately. To ensure that only Graylog can make POST requests to this node I've also added IP filtering to only allow requests from the Graylog server IP along with creating a credential that Graylog will use for authentication. These credentials will need to be configured on Graylog as well so it knows what credentials to send to n8n.  
![diagram1](4.png){: .normal }  

I am also provided with the path where Graylog will need to POST to such that n8n can receive the data. For now I'll stick the with auto-generated paths, but in the future I may want to switch over to human readable paths to make things simpler. 

Before building out the rest of the workflow, I'll get Graylog configured to send the relevant information to this node when it ingests an ET Open CNC finding. After saving and enabling the workflow, I'll head over to Graylog->Alerts->Notifications. Notifications are used by Alerts within Graylog to notify something or someone about a pattern it observed. In this case, Graylog will use this notification to kick off the n8n workflow.

On this notification I'll give it a name and quick description, enter the path it should POST to, and supply the credentials it will send to n8n to verify itself.  
![diagram1](5.png){: .normal }  

After creating the notification, I can perform a test notification with the test button. After testing the notification, I am given a green message saying it was successful, meaning Graylog must have received a 20X response from n8n. Heading over to n8n, I can indeed see that test data was POSTed to the trigger node!
![diagram1](6.png){: .normal }  

With the notification now configured and working, I now just need to define what will trigger the notification within Graylog. Heading to Graylog->Alerts->Event Definitions I'll create a new event definition that monitors for incoming ET Open CNC findings. This definition will use a Graylog search ran every 1 minute looking at the past 1 minute to identify the alerts.  
![diagram1](7.png){: .normal }  

On the fields section of this event definition, I'll also add the various message fields I want to POST to n8n. With the nature of these findings, I'll send the Suricata information and the networking information to n8n. Since the Graylog message will serve as the source, I'll use the following format the grab the fields: $source.field_name.
![diagram1](8.png){: .normal }  

On the notifications section, I'll select the notification I just created and tested as well.  
![diagram1](9.png){: .normal }  

With this, Graylog should be fully configured to send any findings from the ET Open CNC detections. To manually test this without actually infecting one of my endpoints, I'll simply ping an IP that's on the ET Open CNC rule set to mimic traffic of a compromised device to trigger the detection. After doing this, I can see the traffic being detected in Suricata.  
![diagram1](10.png){: .normal }  

Over in Graylog, I can also see the finding being sent over following the expected pattern.
![diagram1](11.png){: .normal }  

Checking the alerts section in Graylog, I can also see that the alert has fired and n8n should've been sent the information from this finding.  
![diagram1](12.png){: .normal }  

Finally, heading over to n8n I can see the information was POSTed to the trigger node.
![diagram1](13.png){: .normal }  

The process from detection firing to sending the data to n8n is now in place! From here I'll work on building out the rest of the workflow to enrich the findings, open cases within DFIR IRIS, and page an analyst to respond to the findings.

### Enriching Findings
To enrich these Suricata findings, I'll be using VirusTotal and GrayNoise as my threat intelligence sources. This type of enrichment is usually manually done by analysts when they want to investigate a resource. This information can help provide context to an investigation by shining light on the resource's owner, configurations, history, etc. Think of it like auditing someone for fraud. The investigating body will collect tax records, bank account information, etc in order to gain as much information as possible. Not every piece of data will be useful but it is better to have it than not.

I'll start this off with adding a node to the workflow. Since n8n has an integration with VirusTotal, I'll choose VirusTotal HTTP Request.  
![diagram1](14.png){: .normal }  



### Opening Incident Cases

## Closing Thoughts