---
title: "Homelab Pt 4 - Deploying Graylog SIEM"
date: 2024-6-22 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-06-22-homelab-pt-4-deploying-graylog-siem
image:
    path: thumbnail.JPEG
    alt: Middle Fork Snoqualmie Trail, Washington
---
## Introduction
In this post I'll walk through the process of standing up a Graylog instance within my homelab. Graylog is an open-source SIEM that I will use for log aggregation, analysis, and management. Starting off, I'll only send OPNsense firewall logs to Graylog (which will be covered in this post), but I do plan on ingesting many log sources into Graylog as my homelab is built out including but not limited to: EDR, DNS, and IDS/IPS logs. Once configured, Graylog will be the central location where all logs are shipped. From there, SOAR workflows will be triggered by Graylog to enrich events, open cases in the case management platform, notify me of new events, etc. 

Graylog and its components will be running on a single Ubuntu 22.04 server virtualized on Proxmox with 4 CPU cores, 8GB RAM, and 100GB storage. While there is a point to be made regarding separate control and data planes of this system being a better overall design, having all the components on a single host works for my use case and limited compute resources so I'll be sticking with that. 

## Standing Up Graylog
Starting off, I'll set the time of the Graylog server to UTC to ensure consistency with all my homelab components:
```shell
sudo timedatectl set-timezone UTC
```

Next up, I'll install the components needed for Graylog. 
For my deployment of Graylog, it consists of 3 main components:
* MongoDB 5.x, 6.x - Stores configuration files for Graylog.
* OpenSearch 1.x, 2.x - Stores logs to be processed by Graylog.
* Graylog Server - Control plane for Graylog.

To get these components installed and working together, I'll be following the [docs](https://go2docs.graylog.org/5-2/downloading_and_installing_graylog/ubuntu_installation.html) provided by Graylog.

### MongoDB
To install MongoDB I will import it's PGP key, create its source list, and install the package via apt using the following commands:
```shell
#Import PGP key
curl -fsSL https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -

#Create source list
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list

#Update apt repos and install MongoDB
sudo apt update
sudo apt install mongodb-org
```
With MongoDB installed, I'll ensure it's running and will start up at system reboot with the following commands:
```shell
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
```

> When attempting to start the MongoDB service I was initially met with a core-dump failure. After some investigating, I discovered the "x86-64-v2-AES" CPU type within Proxmox was causing this error. Changing the CPU type to "host" solved this.
{: .prompt-warning }

Now with MongoDB running, I can move over to installing the OpenSearch.
![diagram1](1.png){: .normal }  

### OpenSearch
Graylog offers support for either OpenSearch or Elasticsearch. I'll be going with OpenSearch as Graylog supports far more up-to-date versions compared to Elasticsearch.

Similar to MongoDB, I'll install OpenSearch by pulling its PGP key and create an OpenSearch source list using the following:
```shell
#Import PGP key
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring

#Create source list
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```

However, when installing OpenSearch v2.12 and greater I'll need to pass an initial password during the install. I'll do that with the following:
```shell
#Update apt repositories
sudo apt update

#Install latest version of OpenSearch
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD=$(tr -dc A-Z-a-z-0-9_@#%^-_=+ < /dev/urandom  | head -c${1:-32}) apt-get install opensearch
```

Before starting up OpenSearch, I'll need to make a few configuration changes. First, I'll edit the config file at /etc/opensearch/opensearch.yml with the following entries: 
```shell
cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
discovery.type: single-node
network.host: 0.0.0.0
action.auto_create_index: false
plugins.security.disabled: true
indices.query.bool.max_clause_count: 32768
```

Next, I'll update OpenSearch's JVM configuration at /etc/opensearch/jvm.options to use half of the system's memory.
```shell
-Xms4g
-Xmx4g
```

Finally, I'll start OpenSearch and ensure it starts at system boot with the following:
```shell
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
```

With OpenSearch installed and running, now I can move over to configuring Graylog Server itself.
![diagram1](2.png){: .normal }  

### Graylog Server
To install Graylog Server, I'll use the following commands:
```shell
#Grab deb for Graylog
wget https://packages.graylog2.org/repo/packages/graylog-5.2-repository_latest.deb

#Add deb via dpkg
sudo dpkg -i graylog-5.2-repository_latest.deb

#Install Graylog Server
sudo apt-get update && sudo apt-get install graylog-server 
```

Before starting Graylog, I'll need to make a few configuration changes in the /etc/graylog/server/server.conf file. There I will add the following entries:
```shell
password_secret = passwordSecret
root_password_sha2 = passwordHash
http_bind_address = 0.0.0.0:9000
```

Of course, in my setup the password_secret and root_password_sha2 variables are set to custom values. With these set, I can now start up Graylog using the following:
```shell
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
```

After confirming that Graylog is running, I can now head to the web interface to finish the setup.
![diagram1](3.png){: .normal }  

### Initial Setup
After heading to the web interface of Graylog located, I am prompted to authenticate. These initial credentials are found within the /var/log/graylog-server/server.log file. Once authenticated, I am met with the initial setup wizard to configure Graylog.
![diagram1](4.png){: .normal }  

Here, I am prompted to configure a CA, set a renewal policy, and provision certificates to my Graylog data nodes. As I am using OpenSearch directly on the Graylog server, I don't have any data nodes and these changes will not affect my operations.  Nonetheless, I'll configure the CA and renewal policy in-case I do want data nodes in the future. I'll also skip the provisioning of certificates as they aren't currently needed.
![diagram1](5.png){: .normal }  
![diagram1](6.png){: .normal }  

After completing the setup wizard, the Graylog server is now accessible through its web interface. 
![diagram1](7.png){: .normal }  

Here, I will authenticate with root and the password of the SHA256 has I entered in the Graylog configuration file. Once authenticated I am brought to the admin interface of Graylog!
![diagram1](8.png){: .normal }  

## Ingesting Logs
Now with Graylog fully up and running, I can get started on shipping my OPNsense logs to Graylog where they will be ingested and parsed. To start, I'll need to create an input within Graylog where OPNsense can send its logs.

### Configuring Inputs
To create an input within Graylog I'll head to Graylog->System->Inputs. Here I'll choose the input type as Syslog UDP and launch the input.
![diagram1](9.png){: .normal }  

Once launched, I am prompted for the input configuration. I'll leave everything on the default settings except the name and listening port. For my setup, I plan on having an input for each log type (One for OPNsense, one for EDR, etc.) so I'll be sure to choose a different port when configuring more inputs later down the road. For now, I'll simply name the input OPNsense and have it listen on 1501. 
![diagram1](10.png){: .normal }  

Once created, I can now see the input is running and awaiting logs.
![diagram1](11.png){: .normal }  

Of course, the throughput will remain 0 until there are actually logs being sent to the input so now I'll configure OPNsense to send them.

### Configuring OPNsense
Over on OPNsense, I'll head to OPNsense->System->Settings->Logging->Remote. Here I can configure a remote server where OPNsense will send logs. I'll click the "+" button to create a logging endpoint and enter Graylog's information. At first, I'll send any and all logs OPNsense can provide to Graylog. If this ends up being too noisy or taking up too much storage on the system, I'll look to not send logs that I am not concerned about.  
![diagram1](12.png){: .normal }  

After saving and applying these changes, I can now see there is some throughput in the Graylog input - I've got logs coming into Graylog from OPNsense!  
![diagram1](13.png){: .normal }  

Heading over to the message stream, I can see the messages being sent in by OPNsense. Here is an example of a firewall log from a device on my personal devices VLAN making a DNS request to my DNS server.  
![diagram1](14.png){: .normal }  

Of course, I only know this because I am manually reviewing the log itself to see what it contains. To do anything on a large or programmatic scale with these logs, I'll need the relevant fields to be extracted from the log. In Graylog, this is done through extractors. Extractors contains patterns that they'll look for within incoming logs and extract the patterns to individual fields. From there, those fields can be used to do all sorts of analysis. To get this parsing done, I'll need to setup an extractor for the OPNsense logs.

### Parsing Logs
Graylog manages extractors on the input level, meaning I'll need to add extractors to the OPNsense input I have created. To do this I'll head to Graylog->System->Inputs->OPNsense Input->Manage Extractors. From here, I could load a sample message and work through creating an extractor that way but instead I'll import an extractor already created by the fine folks of the open-source community. I'll be importing [this](https://github.com/IRQ10/Graylog-OPNsense_Extractors/blob/master/Graylog-OPNsense_Extractors_RFC5424.json) JSON as the extractors to be applied to the OPNsense input. 

After pasting the JSON into the importer, I can see the 6 different extractors that were loaded for IPv4 & IPv6 TCP, UDP, and ICMP firewall traffic.
![diagram1](15.png){: .normal }  

While these extractors will only apply to the firewall logs being sent to Graylog, that is the majority of the logs and what I am most concerned about. 

Once imported, now if I take a look at the incoming message stream I can see that the important fields (src-ip, dst-ip, dst-port, etc.) are being extracted into their own fields!
![diagram1](16.png){: .normal }  

Now that the logs are being properly parsed, they are good to go.

## Closing Thoughts
Now with Graylog fully configured and receiving firewall logs, I can use it to conduct investigations into any traffic occurring on my network, gather statistics on network traffic, and anything else I would like to do with the data. The OPNsense logs will really just be used as an investigative/troubleshooting data source, but as I onboard more log sources to Graylog I can use them to kick off SOAR workflows which will enrich events, open cases in the case management platform, notify me of new events, etc.

Thanks for taking the time to read this post and feel free to reach out via LinkedIn if you have any questions!