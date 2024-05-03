---
title: "Homelab Part 8 - Detecting Malicious Network Traffic Using Graylog and Open Threat Exchange"
date: 2024-4-26 00:00
categories: ["Homelab", "Security"]
tags: ["SIEM", "Detection Engineering"]
img_path: /assets/img/2024-4-26-homelab-part-8-detecting-malicious-network-traffic-using-graylog-and-open-threat-exchange
---
## Introduction
In this post, I will walk through the process of enriching my internal network logs using crowd-sourced threat intelligence and then use that information to identify and alert on malicious network traffic. I'll be using pfSense firewall logs as my log source, Open Threat Exchange (OTX) as my threat intelligence source, and Graylog as my SIEM. Once this process is fully implemented, I'll have visibility into traffic to or from ***known*** malicious resources occurring on my network.

It is important to make the point that using threat intelligence sources to identify IoCs within an environment will have diminishing returns if your adversary really wants to go undetected. With the nature of how IoC lists are built, by extracting indicators from past attacks, it is entirely possible for an adversary to spin up a new resource that hasn't been used in attacks previously. In this situation, because the resource won't be on any IoC lists as it hasn't been used in attacks before, any detections on IoCs will miss the attack entirely. This point stresses the importance of also detecting IoAs/TTPs (non-static) in addition to IoCs (static). With the increase of malware-free intrusions and zero-day exploits in recent times, I'd argue that detecting on IoAs/TTPs are just as important as detecting IoCs if not more important. Nonetheless, adversaries are humans and do re-use the same resources in different attacks. This is where detecting on IoCs helps to identify malicious activity, and what I'll be diving into in this post.

## Alert Lifecycle
By the end of this post, the alert lifecycle for detecting traffic to/from IPs with OTX intelligence tied to them will follow the following lifecycle: 
![diagram1](1.png){: .normal } 

Of course, working alerts out of the SIEM itself is not ideal so in my following posts I plan to use a SOAR tool to automatically enrich the alerts further and open a case within a proper open-source, self hosted case management tool. While I have not decided exactly which SOAR platform I'll be using, I have it narrowed down to either [n8n](https://github.com/n8n-io/n8n) or [Shuffle](https://github.com/Shuffle/Shuffle). For the case management platform, I plan to use [DFIR-IRIS](https://github.com/dfir-iris/iris-web).

For the scope of this post however, I'll be sticking to just creating the alerts within Graylog. Now let's jump into getting the workflow above put in-place.

## Current Setup
In my last post I walked through setting up Graylog, sending pfSense firewall logs to Graylog, and then parsing/formatting said logs with Graylog extractors. As it currently stands, every event that is actioned by a firewall rule will be sent to Graylog. Here is an example log of a device in my 10.0.20.0/24 VLAN using multicast DNS:

```shell
Action
    pass
DataLength
    107
DestIP
    224.0.0.251
DestPort
    5353
Direction
    in
FilterData
    146,,,1701937696,igb1.20,match,pass,in,4,0x0,,255,8482,0,none,17,udp,127,10.0.20.100,224.0.0.251,5353,5353,107
Flags
    none
ID
    8482
IPVersion
    4
Interface
    igb1.20
Length
    127
Offset
    0
Protocol
    udp
ProtocolID
    17
Reason
    match
RuleNumber
    146
SourceIP
    10.0.20.100
SourcePort
    5353
TOS
    0x0
TTL
    255
Tracker
    1701937696
facility
    local0
facility_num
    16
full_message
    <134>Apr 29 01:30:01 filterlog[76239]: 146,,,1701937696,igb1.20,match,pass,in,4,0x0,,255,8482,0,none,17,udp,127,10.0.20.100,224.0.0.251,5353,5353,107
level
    6
message
    filterlog[76239]: 146,,,1701937696,igb1.20,match,pass,in,4,0x0,,255,8482,0,none,17,udp,127,10.0.20.100,224.0.0.251,5353,5353,107
source
    filterlog[76239]:
timestamp
    2024-04-29 01:30:01.000
```

While the firewall log does provide the relevant information to the traffic that occurred, there really isn't any security context to it. If my goal to implement a detective control to identify network traffic relating to known malicious resources using these logs, some kind of enrichment is needed. This is where the Open Threat Exchange (OTX) comes into place.

## Open Threat Exchange
Open Threat Exchange is a crowd-sourced threat intelligence platform where members from the community can share information relating to attacks they have observed. This information include IoCs such as hashes, IPs, domain names, etc. that have been utilized by attackers when carrying out attacks. After sharing these IoCs, other organizations can reference OTX when monitoring for IoCs within their environments to  identify potential attacks. I'll be cross referencing the community-sourced intelligence on IP addresses with the traffic occurring within my network to determine if any of my endpoints are communicating with potentially malicious IPs. 

As the threat intelligence is community-sourced, meaning anyone can contribute, there are bound to be false positive entries into OTX that will likely need to be tuned out through my detection engineering process. This can be mostly avoided by using a closed-source, paid threat intelligence feed that usually get their IoCs from private sources. An example of this would be CrowdStrike's Falcon EDR providing malicious IoCs it observes to CrowdStrike's threat intelligence feed. This type of intelligence is likely much higher quality when comparing to community-sourced threat intelligence as there really isn't room for falsified/accidental contributions to the intelligence feed. Despite this, I will be sticking with OTX as my threat intelligence feed for this use case as it is free and my risk tolerance within my homelab doesn't call for state of the art threat intelligence.

Now let's get into actually setting up these detections.

## OTX Content Pack
Before I can enrich my firewall logs with OTX intelligence, I'll first need to add the mechanism to Graylog for that enrichment to occur. This is where Graylog content packs are used. While content packs can include a wide variety of additions to Graylog, the main focus of the OTX content pack is its functions provided to the user that can be used to query the OTX intelligence feed for IoCs. These functions can be used within Graylog pipelines to automate the lookup of IoCs included within any log in-scope of the pipeline, add custom fields to said log, and then set values to those custom fields depending on the result of the functions. For my use case, I will be passing the DestIP field of all my firewall logs to the otx_lookup_ip function to determine if there is intelligence on that IP. After that, a field named "otx_threat_indicated" will be added to the log as a boolean. That field will either be True or False, depending on what the IP lookup function returned. 

To install the OTX content pack I'll head to Graylog->System->Content Packs, search for the pack, and install it.
![diagram1](2.png){: .normal } 

## Enrichment Pipeline
Now that I've got the functions needed to perform enrichment installed, I can move forward with getting the enrichment pipeline created in Graylog. Graylog pipelines essentially let you transform/process messages coming in from streams. In my case, I'll be adding information to my pfSense firewall messages coming in from the default stream. In order for pipelines to work, they apply rules at different stages that facilitate the changes required. Before creating the enrichment pipeline itself, I'll need to create a rule that performs the enrichment which will be used by the enrichment pipeline. 

### Creating the Pipeline Rule
To create a new rule, I'll head to Graylog->System->Pipelines->Manage Rules->Create New Rule. On this page, I will enter a description of the rule and add the following rule logic:
```shell
rule "Check DestIP Field for OTX Intel"
when
  has_field("DestIP")
then
  let DestIP_intel = otx_lookup_ip(to_string($message.DestIP));
  set_fields(DestIP_intel);
end
```

This rule logic has 3 main step:
1. Check to see if the message contains the DestIP field. If so, do steps 1 & 2. If not, skip the message.
2. Turn the IP into a string and pass it to the OTX lookup function.
3. Create OTX fields and assign their values as returned by the OTX lookup function.

To avoid unnecessary processing, the OTX lookup function doesn't action private IP as those would not be on any IoC lists. Because of this, I won't see any OTX intelligence fields on messages where the DestIP is private.

After the rule has been created, I can now see it in the rules list. As it isn't being used by any pipelines yet, there isn't any throughput volume but I should see that number jump up once the pipeline is created.

Now with this rule created, I can create the pipeline to do the actual enrichment. 

### Creating the Pipeline
To create a pipeline I'll head to Graylog->System->Pipelines->Manage Pipelines->Add new pipeline. Here I will give the pipeline a name and description.  
![diagram1](3.png){: .normal } 

After creating the pipeline, I am given a warning that the pipeline is not connected to any streams meaning it won't have any incoming messages. I'll simply assign it to the default stream, where all my messages are arriving, to start feeding it messages.  
![diagram1](4.png){: .normal } 

After attaching it to the default stream, the throughput starts to count up as messages are flowing into the pipeline.  
![diagram1](5.png){: .normal } 

Now I simply need to attach the rule to any stage within the pipeline to start enriching messages as they are coming in. As this pipeline is rather simple, I will keep it to just one stage and attach the rule to stage 0.  
![diagram1](6.png){: .normal } 

With the rule added to the pipeline, any messages coming into the pipeline will now be actioned by the rule and enriched with OTX intelligence! We can see this by looking at a new log of an endpoint reaching out to the internet, where we see the new otx_threat_indicated field was added and is set to false for this message.  
![diagram1](7.png){: .normal } 

## Tuning Detection
When a new detection is being implemented the success criteria will vary from org to org on what exactly is needed before the detection's findings will be actioned by analysts, but in general the detection should always be tuned to an acceptable baseline. With that being said, let's take a look at what IPs, if any, are being flagged with OTX intelligence before creating alerting on this detection.

About 48 hours after implementing the threat intelligence pipeline, here are top values marked as "threat indicated" by OTX:  
![diagram1](8.png){: .normal }

Unfortunately, there's quite a bit of false positives in terms of volume. In fact, if I aggregate the findings, I can see that in the past 48 hours alone there have been just over 51,000 individual findings. Furthermore, it's not like there are just a few IPs accounting for most of the activity that could be easily tuned out - there are just over 1,000 unique IPs that have been marked as a threat.  
![diagram1](9.png){: .normal }

While I expect most, if not all, of these findings to be false positives, let's sample some of the findings to get an idea on why they're being flagged. Looking at the top IPs by volume, the top 4 all appear to be in the same CIDR range (216.239.32.0/20) so it's safe to assume they are at least somewhat related. Digging deeper into these IPs, they are all owned by Google and seem to be used for DNS. We can confirm this by looking at what/how endpoints are reaching out to these IPs. 

Using Graylog to search for pfSense firewall logs on endpoints reaching out to the 216.239.* address range, I can see that all traffic to these IPs is occurring over UDP 53, aka DNS.   
![diagram1](10.png){: .normal }


What's interesting is looking at one of these IPs in the OTX feed, they clearly spell out that this IP belongs to Google and is benign.  
![diagram1](11.png){: .normal }

Based on this, it appears that the OTX lookup function within the OTX Graylog content back will mark an IP as a threat if there is any intelligence pulses on it within the last 30 days despite those IPs permit-listed on their own service. This is of course not ideal as it enables a lot of false positives, but there isn't much of a direct workaround if I plan to use the OTX content pack within Graylog. What can be done, and what I plan to do in my next post, is to use the wide net that OTX casts, including false positives, as a starting point for determining whether traffic is truly malicious or not. From there, I'll feed the OTX findings to a SOAR platform that will use other intelligence sources to enrich the findings further and determine if it is truly a true positive or not. Only once there is confirmation that the traffic is not known-benign, then I'll open an incident in my case management tool.  

For now, manually tuning out the IPs with the largest volume of would net okay-ish results by decreasing the findings by about 50%. However, since there are still 12,000 false positives per day after those IPs, the number is still far too large to be manually triaged by an analyst. For this reason, I'll plan on alerting on all OTX threats, even the large majority of false positives and then filter out those false positives when processing the alerts with my SOAR platform.  

Of course, this workaround is not ideal but that's the nature of dealing with community-sourced threat intelligence. Using a high quality, 3rd party intelligence feed would circumvent the need for this workaround as there would ideally be far less false positives.  

Now let's take a look at setting up the alerting on the messages identified as threats by OTX that will be processed and enriched by a SOAR platform then sent to a case management platform.  

## Alerting on Identified Threats
Now that we've got all in-scope messages having the otx-threat-identified field added and set to a boolean, we can setup the alerting for when threats are identified. To create a new "event" as Graylog call it, I'll head to Graylog->Alerts->Event Definitions. Here I will create an alert definition that looks for the otx_threat_indicated field where it is set to "true". As I'll be tuning the findings later down the chain, this is all I need to capture the correct messages. I'll set this event to search every 5 minutes, looking at logs from the past 5 minutes to ensure no messages are missed.  
![diagram1](12.png){: .normal }

Now I'll create a notification that creates an alert when the event I just created occurs. To do this I'll head to Graylog->Alerts->Notifications. Here I will create a basic HTTP notification, which will be used to send alerts to the SOAR platform in the future, with example entries that won't actually do anything. This will allow for the alerts to be generated without having the processing infrastructure built out yet.  
![diagram1](13.png){: .normal }

With this notification created, I'll attach it to the event definition I just created such that when the event occurs, this notification is triggered and an alert is created. Once this is done, I can now see the alerts flowing in.  
![diagram1](14.png){: .normal }

Again, most of these are false positives that will be filtered out upon being processed by the SOAR platform.

## Closing Thoughts
Now with the pfSense logs being enriched by the OTX intelligence feed and alerting setup on those identified threats, next steps would be setting up a SOAR solution to further enrich/process the findings and then sending those final, fully enriched alerts to a case management platform. As mentioned above, I plan on using either [n8n](https://github.com/n8n-io/n8n) or [Shuffle](https://github.com/Shuffle/Shuffle) as my SOAR platform and [DFIR-IRIS](https://github.com/dfir-iris/iris-web) as my incident management platform.

Setting up those will be covered in my next post however, so that's all for this post! 

If you have any questions about the contents of this post feel free to reach out via LinkedIn. Thank you!