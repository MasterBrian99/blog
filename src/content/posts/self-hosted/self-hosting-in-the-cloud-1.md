---
title: Manage Your Own Applications Without Local Hardware - Self-Hosting in the Cloud (Part 1)
published: 2025-03-27
updated: 2025-03-27
description: 'using cloud services to run your own applications ("self-managing")'
image: './covers/image1.png'
tags: [self-hosting,self-managed ]
category: 'Self Hosting'
draft: true
---

> Cover image source: [lexica.art](https://lexica.art/prompt/1d46662b-7dc3-4b52-9e39-32433f94161e)


# What is Self Hosting?

 Have you ever wondered people at [Reddit - r/selfhosted/](https://www.reddit.com/r/selfhosted/)  or [Selfhosted - Lemmy.World](https://lemmy.world/c/selfhosted) talking about ? . Yes . they are talking about self hosting . So, what exactly is self-hosting?
 
 If you ask 10 people about `self hosting`, you'll likely get 10 different answers.

 from [wikipedia](https://en.wikipedia.org/wiki/Self-hosting_(web_services)):
 :::note
Self-hosting is the practice of running and maintaining a website or service using a private web server, instead of using a service outside of the administrator's own control. Self-hosting allows users to have more control over their data, privacy, and computing infrastructure, as well as potentially saving costs and improving skills.
:::


random user from [reddit](https://www.reddit.com/r/selfhosted/comments/75fbmj/comment/do6emp2/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button):

 :::note
Selfhosting means that you are running some form of application on your own premises or servers (including virtual servers). Usually it also implies Open-Source and Privacy Selfcontrol.

There aren't any requirements for selfhosting itself. You need to check the apps you want to selfhost to check that.

You can host selfhosted stuff on your own hardware or on a VPS, as far as I am concerned it's the same for most concerns in selfhosting. (If you are privacy concerned you might prefer a physical server you own at some point)
:::

---


Traditionally, self-hosting implies running applications on your own physical hardware. However, in today's cloud-centric world, the definition has evolved. While strictly speaking, running applications on someone else's hardware (like AWS or Google Cloud) isn't "self-hosting" in the purest sense, it's more accurately termed "self-managed."

# But why self-managed ?

Why would you choose to self-manage applications in a cloud environment instead of relying on **local hardware?**

## Living Conditions
For many, especially those in regions with unreliable infrastructure, the cloud offers a lifeline. electricity cost, expensive hardware costs, and limited access to reliable hardware make local self-hosting a painful task. Cloud servers provide a stable and cost-effective alternative, allowing you to manage my applications without the constant worry of hardware failures or power disruptions.

## Accessibility
The ability to access your applications from anywhere with an internet connection. Forget the complexities of port forwarding, static IPs, and other network configurations. Cloud platforms handle the networking, ensuring your services are always within reach.

## Reliability
Cloud servers, by design, prioritize reliability. Providers invest heavily in redundant infrastructure, minimizing downtime and ensuring your applications remain available. This level of reliability is often difficult, if not impossible, to achieve with local hardware.

## Reduced Maintenance
Eliminating the hardware maintenance is a significant advantage. No more worrying about power outages, physical security, or hardware failures. 

## Cost-Effectiveness
Especially for individuals facing high electricity bills and limited budgets, cloud self-management can be surprisingly cost-effective. The pay-as-you-go model allows you to scale resources as needed, avoiding the upfront costs and ongoing expenses associated with local hardware. This is especially true for people who live in areas with high electricity prices, where running local servers can be very expensive.


#  How to Get Started ?

## Choose a Cloud Provider
The foundation of your self-managed cloud environment rests on your choice of cloud provider. This decision impacts everything from cost and performance to the available services and your overall experience.

- Major Cloud Providers (AWS, GCP, Azure): The **BIG** Providers
- Smaller, Specialized Cloud Providers
- Renting Servers from a Cloud Provider: Infrastructure as a Service (IaaS)

The decision between major and smaller providers depends on your requirements. If you already have an account with a major provider and need its extensive ecosystem, it's a logical choice. However, if you're prioritizing cost-effectiveness, simplicity, and a more focused service offering, smaller providers are worth serious consideration.

**Where to Find Smaller Providers:**

To explore smaller cloud providers, here are a couple of starting points:
- [LowEndBox - Cheap VPS, Dedicated Servers and Hosting Deals](https://lowendbox.com/)
- [LowEndTalk](https://lowendtalk.com/)
- [LowEndSpirit](https://lowendspirit.com/)


##  Select a Virtual Machine (VM)

The required specifications for your VM will  depend  on the applications you intend to run. For instance, a simple web server or file storage solution might function  with minimal resources, while a complex database or media server will demand more power.

If you're just beginning your self-hosting journey, it's wise to start with a modest configuration. A VM with 1GB or 2GB of RAM and 1 or 2 CPU cores will likely suffice for many basic applications. This allows you to experiment and learn without incurring excessive costs.

## Install Your Operating System

This step involves choosing and installing the operating system (OS) that will run on your virtual machine. Linux distributions, like Ubuntu or Debian, are popular choices due to their stability, security, and flexibility. Select an OS that you are familiar with, and that is compatible with your applications. The OS will be the base on which all your applications will reside.

## Install and Configure Your Applications

Once your OS is installed, you'll install and configure the applications you want to run. This could include web servers, media servers, databases, or any other software. Follow the application's installation instructions carefully, and configure them to meet your specific needs. This is where you tailor your server to perform the tasks you want it to perform.

## Set Up Security

Security is very important. Configure firewalls to restrict unauthorized access, use strong passwords, and enable SSH key-based authentication for secure remote access. Regularly update your OS and applications to patch security vulnerabilities. This step ensures that your server is safe from unwanted access, and data breaches.


## Manage Backups

Implement a backup strategy to protect your data from loss due to hardware failures or accidental deletion. Schedule regular backups of your important files and databases, and store them in a secure location. This allows you to restore your server to a previous state, and recover data, if needed.

## Monitor Your Applications

Use monitoring tools to track your server's performance, including CPU usage, RAM usage, and network traffic. This allows you to identify and address potential issues before they impact your applications. Monitoring helps you keep your server running smoothly, and allows you to catch problems before they become critical.


# Example Applications that you can self host 


- Web servers
	- [nginx](https://nginx.org/),[caddy](https://caddyserver.com/)
- Media Servers
	- [jellyfin](https://jellyfin.org/),[plex](https://www.plex.tv/)
- Databases
	- [mysql](https://www.mysql.com/),[postgres](https://www.postgres.org/)
- Game Servers
	- minecraft server
- Automation tools
	- [nodered](https://nodered.org/),[n8n](https://n8n.io/)


# Conclusion
**In short, using cloud services to run your own applications ("self-managing") lets you skip the hassle of owning and maintaining physical hardware. It's especially useful if you have limited resources or unreliable local infrastructure.**

**To get started, pick a cloud provider (big or small), choose a virtual server, install your operating system, set up your applications, and make sure everything is secure and backed up. You can run all sorts of things, like websites, media servers, and databases.**

**Cloud self-managing gives you more control and flexibility, letting you run your digital life your way, without the headaches of traditional hardware.**

