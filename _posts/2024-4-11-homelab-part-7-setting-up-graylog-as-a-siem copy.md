---
title: "Homelab Part 7 - Setting Up Graylog as a SIEM"
date: 2024-4-11 00:00
categories: ["Homelab", "Security"]
tags: ["SIEM"]
img_path: /assets/img/2024-4-11-homelab-part-7-setting-up-graylog-as-a-siem
---
## Introduction
Today I will walk through the process of setting up Graylog on my home network. Graylog in an open-source SIEM that allows for the centralization of logs into a single platform which can then alert on various signatures, patterns, or anomalies within said logs. Moreover, you can also use Graylog to simply parse through the logs on an ad-hoc basis when conducting troubleshooting or a forensic investigation.  The primary utilization of Graylog within my homelab will be security-focused, but it will also serve as a troubleshooting tool. 

The following list are some examples of log sources that will be feeding Graylog. This list is not comprehensive.
* pfSense
* DNS
* Endpoints (syslog, sysmon, etc)
* Suricata
* Wazuh
* OpenVAS

In this post, I will walk through setting up Graylog, configuring proper logging on pfSense, sending pfSense logs to Graylog, and extracting relevant fields from those logs.

## Graylog Setup
I'll be installing Graylog on Ubuntu 22.04 VM virtualized on Proxmox. This host has 4 vCPUs, 8 GB RAM, and 100 GB of storage.

Graylog does require a few dependencies before we can get it working. Those dependencies are as follows:
* OpenJDK 17 (This is embedded in Graylog 5.0 and does not need to be separately installed.)
* OpenSearch 1.x, 2.x or Elasticsearch 7.10.2. I'll be using OpenSearch as Grafana has a plugin for it.
* MongoDB 5.x or 6.x

Let's get started with installing these dependencies.

#### Installing MongoDB
To install MongoDB, first I'll need to import it's key using the following command:
```shell
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
```  

Next I will create a list file for MongoDB using the following command:
```shell
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```  

Now, after reloading my APT repositories I can see that the new MongoDB repositories that I just imported.
![diagram1](1.png){: .normal } 

With the proper repositories in place, I can install MongoDB using APT:
```shell
sudo apt-get install -y mongodb-org
```  

Finally, I will ensure that MongoDB is running and starts up when the system does with the following:
```shell
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl --type=service --state=active | grep mongod
```
![diagram1](2.png){: .normal } 

Now with MongoDB installed and running, I will start the OpenSearch installation process.

#### Installing OpenSearch
Similar to MongoDB, I will start by importing the CPG key. This is used to verify that the APT repository is signed. I'll do that using the following command:
```shell
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo apt-key add -
```

Next, I will create an APT repository for OpenSearch with the following:
```shell
echo "deb https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/opensearch-2.x.list
```

After refreshing my APT repositories, I can now see that the correct OpenSearch repositories have been added.
![diagram1](3.png){: .normal } 

To install the latest version of OpenSearch, I will simply use the following command:
```shell
sudo apt-get install opensearch
```

#### Configuring OpenSearch
To configure OpenSearch to work with Graylog, I will update the following fields within the config file located at /etc/opensearch/opensearch.yml:
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

Another thing I need to change is the JVM options located at /etc/opensearch/jvm.options. On this config file, I will update the following values from 1g to 4g to match half of the system's memory.
```shell
-Xms4g
-Xmx4g
```

Next I will configure the kernel parameters at runtime using the following:
```shell
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

With OpenSearch configured to work with Graylog now, I will enable the service and ensure it is running with the following:
```shell
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
sudo systemctl status opensearch.service
```
![diagram1](4.png){: .normal } 

Now with OpenSearch fully installed, configured, and running I will move onto actually installing Graylog itself.

#### Installing Graylog
To install the Graylog Open repository configuration and Graylog itself, I will use the following commands:
```shell
wget https://packages.graylog2.org/repo/packages/graylog-5.0-repository_latest.deb
sudo dpkg -i graylog-5.0-repository_latest.deb
sudo apt-get update && sudo apt-get install graylog-server 
```

That's it for getting Graylog installed. Now I'll need to make a few configuration changes before I can start Graylog.

#### Configuring Graylog
The following are 3 entries we need to add to the Graylog configuration file located at /etc/graylog/server/server.conf:
* password_secret
* root_password_sha2
* http_bind_address

To get the value for password_secret, I will simply run the following command which will give me a random output:
```shell
< /dev/urandom tr -dc A-Z-a-z-0-9 | head -c${1:-96};echo;
```

To get the hash value for the root_password_sha2 entry, I will run the following command and enter my password:
```shell
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

The http_bind_address is simply the Graylog server's IP address. 

After adding these entries to the configuration file, I'll just make sure Graylog is running and enabled on startup with the following:
```shell
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
sudo systemctl --type=service --state=active | grep graylog
```
Now, if I head over to the Graylog's server on port 9000 via my web browser, I can see the web GUI!
![diagram1](5.png){: .normal } 

After authenticating with the admin username and the password I set during the Graylog configuration, I can now use Graylog!
![diagram1](6.png){: .normal } 

## Creating a Graylog Input
Inputs are, just as the name suggests, ways for the various log sources to input data into Graylog. To keep my Graylog configuration simple and prevent performance issues later down the road, I plan on having 1 input for every log source. This means pfSense, Suricata, DNS, etc logs will each have their own input.

Since I am just going to be sending pfSense logs to Graylog for the time being, I will now setup the pfSense input on Graylog. To do this, I will head to Graylog->System/Inputs->Inputs. Here I will select the input type (syslog UDP in this case) and click "Launch New Input".  
![diagram1](7.png){: .normal } 

On the next screen is where I will configure this input. The only changes I will make are as follows:
* pfSense as input title
* 1500 as listening port (This could be any number so long as it doesn't conflict with other listening ports)
* Enable store full message

After making these changes and launching the input, I can now see the input listed. 
![diagram1](8.png){: .normal } 

There is currently no throughput as pfSense is not sending logs yet. Let's change that.

## Sending pfSense Logs
To send pfSense logs over to Graylog, I'll head over to pfSense->Status->System Logs->Settings. Here I won't change any settings except for enabling "Remote Logging". On the "Remote Logging" menu, I will enter the Graylog server IP address along with the pfSense input listening port. I'll also choose what logs will be sent over, and I will choose everything for my use case.  
![diagram1](9.png){: .normal } 

After saving these changes and heading back to Graylog, I can now see that logs are flowing into Graylog "Network IO" tracker is counting up received data.  
![diagram1](10.png){: .normal } 

To view the messages coming in, I can go to "Show received messages" on the pfSense input. Here I am shown a list of the messages pfSense has sent over.  
![diagram1](11.png){: .normal } 

Because I selected everything to be sent over, I should receive logs including firewall events, system events, etc coming in. However, due to the nature of pfSense, I expect the majority of the incoming logs to be firewall logs.  

## Extracting Information from Logs
Looking deeper at the actual messages received, it's clear they aren't formatted very well. For example, take a look at this message below. All the data is pretty rough looking, and overall not what we would expect when trying to read a log.
![diagram1](12.png){: .normal }  

The reason behind this is because we are seeing the logs as they come in without any processing. In order to parse logs into a more readable and normalized format, Graylog uses extractors. Extractors can be applied to inputs where they look for the specific, defined patterns and format the data accordingly. To add an extractor to my pfSense input, I will head to Graylog->System/Inputs->Inputs->pfSense Input-> Manage Extractors.

On this page, you have the option to load a message from its ID and build an extractor that way. I do plan on doing this for future log sources, however since pfSense is so widely used I will simply import an existing extractor pre-built for pfSense logs. I will be using the Lawrence Systems pfSense extractor which can be found on their Github [here](https://github.com/lawrencesystems/graylog_extractors/blob/main/pfsense_2023.json). To do this, I will go to Actions->Import extractors and paste the JSON provided by Lawrence Systems.
![diagram1](14.png){: .normal }  

After adding the extractor, we can see that it actually contains 5 different extractors:
* Suricata alerts
* IPv4 ICMP
* OpenVPN
* IPv4 TCP
* IPv4 UDP

![diagram1](15.png){: .normal }  

Because I don't plan on ever using OpenVPN in my network for the time being I will delete this extractor so that it is not compared to each log that comes in even though there is no chance it will ever match. Worst case, if I do end up using OpenVPN in the future I can always just add that specific extractor in the future.  

With the new extractors in-place we can see that the new messages coming in look much better and formatted with all the relevant information presented. For firewall logs, we can see fields such as srcIP, destIP, destPort, etc. 
![diagram1](16.png){: .normal }  

With the logs now formatted correctly, let's do a quick search.

## Searching Logs
As a simple search, I will simply look for any destination IPs my personal PC is communicating with. I'll do this by using the following search, then viewing the aggregated count of DestIP:
```shell
SourceIP:10.0.1.105
```

In this case, my PC's IP is 10.0.1.105 and the SourceIP field has been created by one of the extractors running on the pfSense input. Here are the results:
![diagram1](17.png){: .normal }  

It looks like the top IP my PC is communicating with in the last hour is the Graylog server, not surprising. I also see some calls to what look to be Spotify, Google, and various CDN servers. These all make sense based on my PC's activity.

Outside of troubleshooting or a forensics investigation, this information is interesting at best without context. In future posts, I will walk through making not only pfSense logs actionable from a security perspective, but also other log sources as well.

## Closing Thoughts
Now that Graylog is setup and I have got pfSense logs not only flowing to it, but also being formatted correctly I can now setup alerts, visualization, custom searches, etc on these logs. In my next post, I plan to setup alerting on some common abuse cases that the pfSense logs cover. 

Feel free to reach out via LinkedIn if you have any questions about the content of this post. Thank you!