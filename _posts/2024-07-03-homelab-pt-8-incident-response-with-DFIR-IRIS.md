---
title: "Homelab Pt 8 - Incident Response with DFIR IRIS"
date: 2024-7-03 00:00
categories: ["Homelab", "Security"]

img_path: /assets/img/2024-07-03-homelab-pt-8-incident-response-with-DFIR-IRIS
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

## Exploring DFIR IRIS