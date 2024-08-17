---
title: "Homelab Pt 8 - Incident Response with DFIR IRIS"
date: 2024-7-07 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-07-07-homelab-pt-8-incident-response-with-DFIR-IRIS
image:
    path: thumbnail.JPEG
    alt: Mt. Rainier National Park, Washington
---
## Introduction
In this post I'll be standing up and configuring [DFIR IRIS](https://dfir-iris.org/), an open-source incident response platform. This platform will allow me to open, triage, and document incidents that occur within my environment. Once deployed, I plan to have my various alerting services (EDR, IDS/IPS, etc.) trigger SOAR workflows that will ultimately open cases within DFIR IRIS with the relevant information automatically.

I'll be deploying DFIR IRIS as a container using Docker. The platform is comprised of the following Docker services:
* app - iris_webapp: The core, including web server, database management, module management, etc.
* db - A PostgreSQL database
* RabbitMQ: A RabbitMQ engine to handle job queuing and processing
* worker: A job handler relying on RabbitMQ
* nginx: A NGINX reverse proxy

Once configured, nginx will be listening on port 443 to serve the web interface to any clients that request it. DFIR IRIS will be running on an Ubuntu 22.04 server virtualized on Proxmox with 4 CPU cores and 8GB RAM.

With the context now set, I'll jump into getting DFIR IRIS stood up.

## Deploying DFIR IRIS
To deploy DFIR IRIS, I'll start by cloning the repository, checking out the latest version, and copy the .env template using the following:
```bash
#Fetch repo and pivot to directory
git clone https://github.com/dfir-iris/iris-web.git
cd iris-web

#Checkout the latest version
git checkout v2.4.7

#Copy the .env.model template contents to .env
cp .env.model .env
```

Before spinning up the platform, I'll change the following entries in the .env file to randomly generated values to ensure my instance is not using the default values:
* POSTGRES_PASSWORD
* POSTGRES_ADMIN_PASSWORD
* IRIS_SECRET_KEY
* IRIS_SECURITY_PASSWORD_SALT
* IRIS_ADM_PASSWORD
* IRIS_ADM_API_KEY

The IRIS_ADM_PASSWORD value will be the actual password I use to access the web interface and must be at lest 12 characters with a capital letter and number. Since this requirement is hardcoded and cannot be changed, if the provided password does not meet the policy the administrator account will not be able to authenticate.  

The IRIS_ADM_API_KEY value is simply just a long string that will serve as an API key for the platform. In the future, my other tools will use this API key to programmatically interface with DFIR IRIS.

To generate proper random values (or as random as you can get using a computer) I'll be using the following command:
```bash
openssl rand -base64 64
```

After generating and populating the values, I can now start up the platform using the following:
```bash
#Builds the Docker services
docker compose build


#Starts the Docker services in the background
docker compose up -d
```

Since the Docker services already have the *restart: always* flag in the docker-compose.yml file, I don't need to worry about adding anything to ensure DFIR IRIS starts on system boot.  

With the platform now up and running, if I head over to the web interface on my web browser I am met with a warning. This is expected as, by default, DFIR IRIS uses a self-signed certificate that needs to be explicitly trusted.  
![diagram1](1.png){: .normal }  

After proceeding past the warning, I can now see the platform's web interface.  
![diagram1](2.png){: .normal }  

Once I authenticate with the password I set in the .env file earlier, I am brought to the dashboard and can now start using the platform!  
![diagram1](3.png){: .normal }  

Now let's explore the platform a bit to see how I might use it when working an incident.  

## Exploring DFIR IRIS
### Cases
DFIR IRIS uses cases to track incidents. I plan on having most of my cases be opened programmatically through SOAR workflows, but they can also be manually opened if needed. When manually creating a case, you can select various field to set the scope of the case such as template, classification, description, etc. You can think of this platform like Jira but tailor-made for security incidents, and naturally much less bloated than a platform like Jira. 
![diagram1](4.png){: .normal }  

Each case can have various data points added to enrich the case such as IOCs, notes, assets involved, evidence, and much more. These all help to paint a clear picture using all the information relating to an incident. These can all be added inside of the individual cases directly.
![diagram1](5.png){: .normal }  

Aside from supplementary data, cases can also have tasks assigned to them so the analyst working the case know exactly what steps need to be taken at a minimum, a timeline for actions taken on the case, and lots of other options.


### Customers
DFIR IRIS also has support for adding various customers into the platform. This is useful if you're providing IR services to various customers/user groups and have various SLAs, points of contact, etc. for each of them. The idea is for every case, you choose what customer it applies to and the analyst can then use view that information is needed. This tagging of cases is also extremely helpful for reporting on trends and statistics for individual customers if needed. Below is a screenshot of a simple "customer" I created for my network.  
![diagram1](6.png){: .normal }  



### Much, Much More!
What I've covered here is a fraction of a scratch on the surface of DFIR IRIS's capabilities. The team behind the project has done a truly great job with the platform and I can't wait to see how it grows going forward. If you're interested in checking out the project, you can find it [here](https://dfir-iris.org/).

## Closing Thoughts
With DFIR IRIS now deployed, I can start to build out SOAR workflows that will automatically gather events, enrich them, and then open cases within the platform with all the relevant information ready for analyst triage. In my next post I plan on deploying [n8n](https://github.com/n8n-io/n8n), an open-source automation platform, to serve as my SOAR and build out an end-to-end workflow for the Suricata IDS detections I set up in my [previous post](/posts/homelab-pt-7-detecting-known-threats-with-suricata/). 

Thanks for taking the time to read this post and feel free to reach out via LinkedIn if you have any questions!
![human_badge](badge.svg){: .normal }{: width="125" }