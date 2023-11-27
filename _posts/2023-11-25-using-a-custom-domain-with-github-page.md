---
title: "Using a Custom Domain with Github Pages"
date: 2023-11-26 00:00
categories: ["Tutorials", "Blog"]
tags: ["Cloudflare", "DNS"]
img_path: /assets/img/2023-11-25-using-a-custom-domain-with-github-page.md
---
## Introduction
In this post we will walk through how to use a custom domain with a Github Pages website. I recently created this blog to document my cybersecurity journey and while I am loving how easy the experience has been, I'm not too estatic about the only way to access my blog being a github.io subdomain (judelanning.github.io). Thankfully, it's a fairly straightforward and quick process to use a custom domain.  

Today we will make some DNS changes to the judelanning.com domain that will allow it to serve my blog when it is requested. I picked this domain up for just under $10 per year through Cloudflare. I went with Cloudflare as my provider as they have a great security and analytics toolset and were surpisingly cheaper than GoDaddy, Dynadot, Namecheap, and a few other vendors I looked at.

The steps in this post that deal with actually making DNS changes will be Cloudflare-specific as they are made on the domain vendor side but the process should translate to pretty much any vendor. If you are using a different vendor and are having trouble finding where to make these changes, do a Google search for "VendorName DNS Record Changes" and you should be able to find where to make DNS record changes.

## Prerequisites
To get started, you'll need the following items.
1. A domain that you can make DNS record changes to.
2. A Github Pages website, preferably already up and running so you can verify the changes work.

## Updating DNS Records
The first change we will need to make will be adding a few A records to our domain's DNS record. These records will point to the IPs where Github Pages websites are hosted. To do this from the main Cloudflare dashboard, navigate to Domain Registration-> Manage Domains. On this page we will see a list of all the domains we own. Now click the "Manage" button next to the domain that we would like to serve our Github Pages website.
![CloudflareNavigationBar](3.png){: .normal }  
![DomainManage](4.png){: .normal }  

Now we will see an overview for the domain. Click on "Update DNS configuration".
![DomainOverview](5.png){: .normal }  

This is where we will be adding 8 A records our domain's DNS record. The first set of 4 A records will be for judelanning.com that point to 4 different IPs where our Github Pages site could be served from. 

For a more in-depth explination on these changes, take a look at [Github's docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site) regarding using a custom domain.

As of 11/26/23, the following 4 IPs are what we need to make records for:
* 185.199.108.153
* 185.199.109.153
* 185.199.110.153
* 185.199.111.153

To do this, click "Add record". 
![AddRecord](6.png){: .normal }  

Now we can actually start adding DNS records. We will need to add a record for each of the 4 IPs mentioned above. Make sure each record is Type A and the Name is @. The "Content" field is where we will enter the IP address for each record. With Cloudflare we have the option to use their proxy for our site. I will leave this enabled, but feel free to disable it if you don't want that functionality. I'll also leave the TTL on auto. After adding a record for each IP, we should see something similar to these records.
![@records](8.png){: .normal }  

Now that we've got the 4 records for our root domain created, we need to create 4 more records for www so that www.judelanning.com will resolve. The process is exactly the same as above with the same 4 IPs, but instead of @ in the "Name" field, we will use "www". Below is an example of one of these records.
![@records](9.png){: .normal }  

Once we have all 4 records added for www, all the DNS records for our domain will looking something like this.
![@records](10.png){: .normal }  

Those are the only changes we need to make on the vendor side. The next section will cover what finishing changes are needed on the Github side.

## Updating Github Pages Settings
On Github, go to the repository where your Github pages site is stored. From there go to Settings->Pages. Here we will see a section for using a custom domain. Enter the domain and Github will check for the DNS records we just created. After a few seconds, we should see the following messages notifying us that Github recognizes our changes and we are good to go.
![@records](11.png){: .normal }  

If you see that the DNS check was not successful, double check your DNS records to ensure they are all correct.

From here, the changes could take up to 72 hours to fully propogate but I've found they usually go through within just a few minutes. Open up a new tab and try to access the domain. If everything is working, we should see the blog served to us! If you don't see your Github Pages site, try clearing cache or trying in a private browser tab.
![@records](14.png){: .normal }  

> If you are using Cloudflare as your domain vendor, you may run into issues when attempting to resolve your domain. I ran into this issue and wasn't able to view my blog due to default Cloudflare settings. Take a look at the section below to resolve that issue.
{: .prompt-danger }  

## Cloudflare-Specific Issue
After adding the custom domain into Github and the DNS chuck successfully going through, I tried to access judelanning.com and was met with a ERR_TOO_MANY_REDIRECTS error and could not reach the blog. It turns out, the default settings on my Cloudflare account were causing a redirect loop. This was caused by my SSL/TLS encyption mode within Cloudflare being set to "Flexible" instead of "Full". With "Flexible" selected, Cloudflare will use HTTP to connect to the origin, even when our browser connects to Cloudflare using HTTPS. From there, Github responds with an HTTPS redirect which causes the redirect loop. To resolve this, simply change the SSL/TLS encryption mode to "Full". This setting can be found in the Domain Overview-> SSL/TLS-> Overview.
![@records](15.png){: .normal }  
![@records](16.png){: .normal }  

More information on why this issue occurs can be found on [this](https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/) support doc from Cloudflare.

## Closing Thoughts
If you are able to view your Github Pages site using the custom domain, you're all set! Anyone from the internet will also be able to view your Github Pages site using your custom domain. The subdomain provided by Github (In my case, judelanning.github.io) will also continue to work as a way to access your website. 

If you have any questions regarding the content of this post, feel free to reach out via LinkedIn.

Thanks for taking the time to read this post and good luck with your page!