---
title: "Testing private DNS resolution over an Azure P2S VPN connection using Azure DNS Private Resolver"
date: 2022-06-02T21:54:47+02:00
draft: true
tags: ["azure", "cloud"]
description: "The purpose of this post is to try out the new Azure DNS Private Resolver resource. To test it, we're going to try to resolve the DNS of a few private Azure resources when we're connected to an Azure Point-to-Site VPN connection."
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/testing-private-dns-resolution-using-azure-dns-private-resolver).

I have had this long time issue with DNS resolution of private resources on Azure when working over a Point-to-Site VPN connection.

At first I didn't plan to write about it, but the new [Azure DNS Private Resolver](https://azure.microsoft.com/en-us/blog/announcing-azure-dns-private-resolver-now-in-preview/) resource is a potential solution to this problem and also I have been testing it those past few days, so at the end I have thought that this post might be helpful to someone, so here I am.

The problem I'll be talking about in this post and also some of the potential solutions is that **a private DNS zone will not work over an Azure P2S VPN connection**.

Be aware that **this issue also happens if you try to resolve some Azure private resources from an on-premise network connected via ExpressRoute or VPN**.

# What is the problem when trying to resolve a private resource over an Azure VPN P2S

Let me explain a little more in-depth what the problem is about. I'm going to start showing you a simplified example so you can have a better understanding of what's the issue here.

![diagram](/img/vpn-p2s-problem-diagram.png.png)

As you can see it's a pretty standard setup, we have a public app where the customers connect via public internet and this app uses a few private resources, to be more precise, the public app makes a call to another app and it also needs a database to persists some data.

On this example it makes no sense that both the database and the second app could be accessed from anywhere on the internet, so we're going to make them private. To make both resources private we're going to Azure Private Endpoints.

When a private endpoint is created, Azure changes the public name resolution by adding another CNAME record pointing towards the dedicated FQDN of private endpoint.    
By default, it also creates a private DNS zone, corresponding to the ``privatelink`` subdomain, with the DNS A resource record for the private endpoints.

When you resolve the resource endpoint URL from outside the VNet with the private endpoint, it resolves to the public endpoint of the resoucee. When resolved from the VNet hosting the private endpoint, the resource endpoint URL resolves to the private endpoint's IP address.

It might sound a little bit complicated, but it's quite simple, here's a quick example:

- I have create a new App Service and it has a public IP address provided by Azure.

<ADD-IMG>

- Now I created a Private Endpoint to make this App Service private. And as you can see Azure is changing the public name resolution by adding there another CNAME record pointing towards the dedicated FQDN of private endpoint.

<ADD-IMG>

- When the private endpoint was created a private DNS zone was created, this DNS zone contains an A-record that points the private endpoint address to the private IP that is associated for the resource. Also this private DNS zone has been attached to the VNET.

<ADD-IMG>

- Now when you try to resolve it from a client that inside the VNET, it will response the private IP address.

<ADD-IMG>

- And if you try to resolve it form a client from outside the VNET, it will response the public IP address, but bear in mind that if you try to call the app from outside the VNET it won't work.

<ADD-IMG>

Now, image the scenario where a developer needs to access those private resources, for that reason you put in place a Point-to-Site VPN, but when someone connects to the VPN and tries to invoke call the private app endpoint or the private cosmosdb endpoint it gets an error.

<ADD-ERRORS>

And that's because when connected over an Azure P2S VPN connection the private DNS zone resolution does not work, so it tries to connect using the public endpoint instead of the private endpoint private IP.

As I stated before, be aware that the same problem happens if you're trying to access a private Azure resource from an on-premise network connected to Azure via Express Route or VPN.

To solve it, there are a few solutions available and in the next sections I'm going to talk about it.


# Solution 1: Modify the hosts.config file on you local machine

# Solution 2: Use a DNS Forwarder

In order for the P2S VPN clients to be able to resolve Private Endpoint entries hosted on Azure Private DNS Zones, you must leverage an existing DNS Server (Forwarder or Proxy) or deploy one IaaS VM using a DNS Server role

Once you have a DNS forwarder/proxy deployed on Azure, you can define the DNS server at the VNET level or set DNS Server configuration directly on client XLM profile. Post this, you will be able to resolve Private Endpoint entries from your P2S clients.

# Solution 3: Use Azure Private DNS Resolver
