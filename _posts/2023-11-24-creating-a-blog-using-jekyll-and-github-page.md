---
title: "Creating a Blog Using Jekyll and Github Pages"
date: 2023-11-24 00:00
categories: ["Tutorials", "Blog"]
tags: []
img_path: /assets/img/2023-11-24-creating-a-blog-using-jekyll-and-github-pages
---
## Introduction
When looking to create a blog, you may be overwhelmed to find there are a plethora of options out there such as Blogger, Wordpress, Medium, and the list goes on. I was in the same boat when I was looking at hosting options for this blog. I ended up using a Jekyll template hosted on Github Pages for no reason other than it's free and I really love the look and feel of the Chirpy Jekyll template this blog uses. This post will walk through setting up a blog just like this one, for free!

## Prerequisites
Before we can get started blog, we will will need the following:
1. A Github account.
2. A computer that has Ruby installed. Instructions for installing Ruby on various systems can be found [here](https://jekyllrb.com/docs/installation/).
3. A text editor of our choice to make modifications to the template. I use VS Code as it is free, open-source, and lightweight.
4. A computer that has Git installed. Instructions for installing Git on various sytems can be found [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git).

## Getting Started
To get started building our blog, we will first need use the Chirpy Starter template to create a new repository that will contain our blog. Go to [this](https://github.com/cotes2020/chirpy-starter) link and click "use this template" under the installation section.

![chirpyStarterTemplate](1.png){: .normal }

Now we'll be prompted to enter some information about our new repository. Enter the repository name in the following format: githubUsername.github.io

For this example, the Github username for my blog is jekyllblogdemo, so I will enter: jekyllblogdemo.github.io

Enter if a descripion if you would like and leave the visibility to public. Click "Create repository".

> Make sure the repository is named correctly, the site will not work properly if it isn't.
{: .prompt-warning }  

![repositoryInformation](2.png){: .normal }  

After a few seconds, we will now see the new repository. Now we need to change the source where Github Pages pulls from. To do this go to Settings -> Pages -> Source and change it to Github Actions.
![repositoryInformation](3.png){: .normal }  
![repositoryInformation](4.png){: .normal }  
![repositoryInformation](5.png){: .normal }  

Now that we've got the correct source set, we're just a few steps from having the blog up and running! 

## Hosting the Blog Locally
Next, we will clone the repository we have in Github to our local machine.

To get the URL we need to clone, copy the URL provided in the "HTTPS" section under the green "Code" button.
![repositoryInformation](6.png){: .normal }  

To clone the repository, open up the terminal and type the following command with the URL you just copied.

```shell
git clone https://github.com/jekyllblogdemo/jekyllblogdemo.github.io.git
```  

> If this is your first time using Git, you may get an authentication error. If so, take a look at [this](https://docs.github.com/en/get-started/getting-started-with-git/about-remote-repositories) page for setting up authentication options.
{: .prompt-warning }  

Now we can cd into the repository and see what files are present using the following commands in that order.

```shell
cd jekyllblogdemo.github.io
```  

```shell
ls
```  

Here is what we can see for the files that are now on our local host. 
![repositoryInformation](7.png){: .normal }  

To open all items in the current folder in VS Code, type the following command.
```shell
code .
```  

If you get a prompt asking if you trust the author, click the box that says you do because the author is you! Now that we've got the repository opened in VS Code, click Ctrl + ` to open the terminal in VS Code. Once the terminal is open, type "bundle" and hit enter. This will now download any dependencies needed for the blog. This usually takes a few minutes.

> If you get an error saying bundle doesn't exist, make sure you have Ruby installed and added to your hosts path.
{: .prompt-warning }  

Once we see that the bundle is complete, we can run the following command to finally get the blog running on our local machine.

```shell
bundle exec jekyll s
``` 

After a few seconds, we should see an local IP address we can click on where our blog is running to view it. Ctrl + Click the IP address to view the page.
![repositoryInformation](8.png){: .normal }  
![repositoryInformation](9.png){: .normal }  

There it is! Now we can customize it locally, then push to changes to Github and we will have a fully functioning blog.

We'll need to shut down the local blog to make any changes to it's content. In the VS Code terminal, click Ctrl + C to shut it down.

## Customizing the Blog
While there are plently of files we can change to customize the blog, the most important will be _config.yml. Click on this file in VS Code to open the file. There we can see various fields we can alter to change the blog. Feel free to dive into the very specifics, but we will just change a few of the obvious items.
1. Title. This is the large text that shows up under the picture and in the top of the tab. We'll change this to "Jude's Blog".
2. Tagline. This is the smaller text that goes below the title, usually to provide a brief overview of the blog. We'll change this to "A tutorial on creating a blog!"
3. Avatar. This is the picture that shows on the left side of the blog. I use the link from my LinkedIn profile picture, but any link works. You could also upload an image file and link it's path to use a local image.

![repositoryInformation](10.png){: .normal }  
![repositoryInformation](11.png){: .normal }  

Now if we save the file and run the command to start the blog locally, we should see these changes. Run this command again and click the IP address it provides.
```shell
bundle exec jekyll s
``` 

Here's what it looks like now - starting to look pretty good!
![repositoryInformation](12.png){: .normal }  

There are a bunch of other things that you can and should customize like the blogs favicon, social media links, contact information, and the list goes on. You can find most of that within the _config.yml file, but if you have any questions there is plenty of information on [this](https://chirpy.cotes.page/) site.

Before we push these changes to Github, there is one more thing we must change in order for the blog to function properly. Back in the _config.yml file, find both the url and baseurl fields and enter the repository name (githubUsername.github.io) to the right of the colon on both of them. It should look something like this, but with your repository name.

![repositoryInformation](13.png){: .normal }  
![repositoryInformation](14.png){: .normal }  

With those fields changes, we can now finally push the changes to Github and have the blog availible from the internet!

## Pushing Changes to Github
First, we'll need to stage our changes. This is our way to telling Github what we would like to send. This can be done by typing the following command into the VS Code terminal.
```shell
git add .
``` 

This will add all changes we've made to what were sending to Github. Once there are more components invovled with the blog, this may not be the ideal way of staging our changes because it doesn't allow for picking and choosing what we are sending so if there is something like an unfinished blog post that we don't want to be uploaded, that would get uploaded. [This](https://levelup.gitconnected.com/staging-commits-with-git-add-patch-1eb18849aedb) page gives a good overview of doing granular staging if that is something you need.

Now we need to commit our changes. This is the final step before actually sending the changes to Github. We will need to add a message at the end of this command to tell Github what we changed. This is just used for human reading, so it doesn't need to follow any specific format but it should give a general overview of the changes you have staged. For this example, we will just say it is an inital commit.
```shell
git commit -m "Initial commit."
``` 

We will now see an overview of the changes that are going to be sent to Github. Verify these changes are correct.

Now we can send these changes to Github where they will process them and update our blog. To do this, run the following command.
```shell
git push origin main
``` 

> If this is your first time using Git, you will be asked to verify you are actually approved to make changes to the repository. Follow the instructions to sign into Github.
{: .prompt-warning }  

> Well, after a bit of digging into why my Github actions weren't working, it turns out Github will disable them on any accounts that were created with an email from a temporary email site. If you can't see the actions in this next section that may be why. I created a new Github account with a gmail email and am back up to this point. Going forward, the Github username will be jekyllblogdemo2 instead of jekyllblogdemo. 
{: .prompt-danger }  

Once the git push goes through, we can view the "Actions" tab back on the Github repository page and see blog being built.
![repositoryInformation](15.png){: .normal }  

Once these actions are all green and marked as done, we should be able to view the blog from any device by going to the name of the Github repository (jekyllblogdemo2.github.io in this case). If we go to there, you can see the site!
![repositoryInformation](16.png){: .normal }  

Now anyone can go and visit our blog from the internet. However, there isn't much to see - let's fix that.

## Making Blog Posts
Now that our site is up and running, we need to add some content to it. Back in VS Code, we add posts by creating new files under the _posts folder. Right click the folder and click "New File". The name of the file will need to match the naming convention as follows YEAR-MONTH-DAY-TITLE.md. So for this post, we will call the file 2023-11-24-first-post.md.

If we save the file, even with nothing in it, we can still view the post on our locally hosted blog if we start it up. Let's do that with the following command.
```shell
bundle exec jekyll s
``` 

If we click the IP address it gives us, we can see the first post right there on the home page. If we click on the post we can see the content within it, which is nothing currently.
![repositoryInformation](17.png){: .normal }  
![repositoryInformation](18.png){: .normal }  

Let's add some content to it! Each post page should start with some basic information that allows Chirpy to properly handle it. This information requirement along with all the other formatting options can be found [here](https://chirpy.cotes.page/posts/write-a-new-post/#prompts). Be sure to check it out if you want to use some of the advanced features such as the alerts or code blocks you have seen in this post. For now, feel free to copy the information below to get the basic information down.
```
---
title: "My First Blog Post!"
date: 2023-11-24 00:00:00 +0800
categories: ["Tutorials", "Blog"]
tags: ["First Post"]
---
# This is my first blog post!
Blah blah blah.
```

If we add that to the 2023-11-24-first-post.md file and save, then refresh our server we can see the changes.
![repositoryInformation](19.png){: .normal }  

Of course, you'll want to add lots of more information on your first real post. You can add things like pictures, videos, and all sorts of cool formatting which can be found [here](https://chirpy.cotes.page/posts/write-a-new-post/#prompts).

For now though, let's push this new post to our Github repository so the changes will be reflected on the public version of the blog. First, stage the changes.
```bash
git add .
```

Next, commit the changes with a comment on what the changes are doing.
```bash
git commit -m "Added first post"
```

Finally, push the changes to Github with the following.
```bash
git push origin main
```

Back on the actions page on our repository we can see the changes being added.
![repositoryInformation](20.png){: .normal }  

Once the changes are done, if we go to the blog from the internet we can now see the new post ready for viewing!
![repositoryInformation](21.png){: .normal }  

## Closing Thoughts
Armed with the ability to create new blog posts, we can create new posts in different categories with various tags and it will all fit nicely together thanks to the Chirpy Jekyll template. Almost every aspect of the blog is customizable, so feel free to customize elements as per your preferences. For example, on my blog I wanted to remove the Twitter, RSS Feed, and Email contact buttons from the bottom left of the site. I was able to do that and add a LinkedIn button instead.

If you have any questions or troubles setting up your blog, feel free to reach out to me directly or there is a bunch of online information regarding Jekyll and this template in specific as well. 

Thanks for taking the time to read this post and good luck!

## Resources
[Chirpy Starter Template Github](https://github.com/cotes2020/chirpy-starter)  
[Chirpy Template Docs](https://chirpy.cotes.page/)  
[Git Install Docs](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)  
[Github Remote Repository Authentication Docs](https://docs.github.com/en/get-started/getting-started-with-git/about-remote-repositories)  
[Jekyll/Ruby Install Docs](https://jekyllrb.com/docs/installation/)