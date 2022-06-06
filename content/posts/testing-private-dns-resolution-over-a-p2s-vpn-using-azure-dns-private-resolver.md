---
title: "Testing private DNS resolution over an Azure P2S VPN connection using Azure DNS Private Resolver"
date: 2022-06-02T21:54:47+02:00
draft: true
tags: ["azure", "cloud", "terraform"]
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

![example-diagram](/img/vpn-p2s-problem-diagram.png)

As you can see it's a pretty standard setup, we have a public app where the customers connect via public internet and this app uses a few private resources, to be more precise, the public app makes a call to another app and it also needs a database to persists some data.

On this example it makes no sense that both the database and the second app could be accessed from anywhere on the internet, so we're going to make them private. To make both resources private we're going to Azure Private Endpoints.

When a private endpoint is created, Azure changes the public name resolution by adding another CNAME record pointing towards the dedicated FQDN of private endpoint.    
By default, it also creates a private DNS zone, corresponding to the ``privatelink`` subdomain, with the DNS A resource record for the private endpoints.

When you resolve the resource endpoint URL from outside the VNet with the private endpoint, it resolves to the public endpoint of the resoucee. When resolved from the VNet hosting the private endpoint, the resource endpoint URL resolves to the private endpoint's IP address.

It might sound a little bit complicated, but it's quite simple, here's a quick example to help you understand:

## Example about how private endpoint works 

- I have create a new App Service and it has a public IP address provided by Azure.

```bash
$ nslookup dns-resolver-test.azurewebsites.net
Servidor:  ...
Address:  ...

Respuesta no autoritativa:
Nombre:  waws-prod-am2-439-397b.westeurope.cloudapp.azure.com
Address:  20.50.2.56
Aliases:  dns-resolver-test.azurewebsites.net
          waws-prod-am2-439.sip.azurewebsites.windows.net
```

- Now I created a Private Endpoint to make this App Service private. And as you can see Azure is changing the public name resolution by adding there another CNAME record pointing towards the dedicated FQDN of private endpoint.

```bash
$ nslookup dns-resolver-test.azurewebsites.net
Servidor:  ...
Address:  ...

Respuesta no autoritativa:
Nombre:  waws-prod-am2-439-397b.westeurope.cloudapp.azure.com
Address:  20.50.2.56
Aliases:  dns-resolver-test.azurewebsites.net
          dns-resolver-test.privatelink.azurewebsites.net
          waws-prod-am2-439.sip.azurewebsites.windows.net
```

- When the private endpoint was created a private DNS zone was created, this DNS zone contains an A-record that points the private endpoint address to the private IP that is associated for the resource. Also this private DNS zone has been attached to the VNET.

![private-endpoint-dns-zone](/img/private-endpoint-dns-zone.png)

- Now when you try to resolve it from a client that's inside the VNET, it will response the private IP address.

```bash
$ nslookup dns-resolver-test.azurewebsites.net
Server:  UnKnown
Address:  168.63.129.16

Non-authoritative answer:
Name:    dns-resolver-test.privatelink.azurewebsites.net
Address:  10.0.0.4
Aliases:  dns-resolver-test.azurewebsites.net


$ curl -I dns-resolver-test.azurewebsites.net
HTTP/1.1 200 OK
Content-Length: 3161
Content-Type: text/html
Last-Modified: Thu, 27 Aug 2020 23:23:23 GMT
Accept-Ranges: bytes
ETag: "5f48406b-c59"
Server: nginx/1.19.2
Date: Sun, 05 Jun 2022 17:21:50 GMT
```

- And if you try to resolve it form a client from outside the VNET, it will response the public IP address, but bear in mind that if you try to invoke the app from outside the VNET it will thrown an error.

```bash
$ nslookup dns-resolver-test.azurewebsites.net
Servidor:  ...
Address:  ...

Respuesta no autoritativa:
Nombre:  waws-prod-am2-439-397b.westeurope.cloudapp.azure.com
Address:  20.50.2.56
Aliases:  dns-resolver-test.azurewebsites.net
          dns-resolver-test.privatelink.azurewebsites.net
          waws-prod-am2-439.sip.azurewebsites.windows.net

$ curl -I dns-resolver-test.azurewebsites.net
HTTP/1.1 403 Ip Forbidden
Content-Length: 1895
Content-Type: text/html
x-ms-forbidden-ip: 71.11.124.148
Date: Sun, 05 Jun 2022 17:18:48 GMT
```


Now, image the scenario where a developer needs to access those private resources, for that reason you put in place a Point-to-Site VPN, but when someone connects to the VPN and tries to invoke call the private app endpoint or the private cosmosdb endpoint it gets an error.

```bash
$ curl -I dns-resolver-test.azurewebsites.net
HTTP/1.1 403 Ip Forbidden
Content-Length: 1895
Content-Type: text/html
x-ms-forbidden-ip: 71.11.124.148
Date: Sun, 05 Jun 2022 17:24:36 GMT
```

And that's because when connected over an Azure P2S VPN connection the private DNS zone resolution does not work, so it tries to connect using the public endpoint instead of the private endpoint private IP.

As I stated before, be aware that the same problem happens if you're trying to access a private Azure resource from an on-premise network connected to Azure via Express Route or VPN.

To solve it, there are a few solutions available and in the next sections I'm going to talk about it.

# Solution 1: Modify the hosts file on you local machine

The Hosts file is used to override the DNS system so that a browser or other application can be redirected to a specific IP address.

This is the easiest solution you'll only need to modify the host file in you machine to override the DNS resolution of the private services.

Here's an example of how to do it:

```bash

```

Now those services instead of resolving to the public endpoint, they will resolve to the private endpoint private IP. 

This solution works but is very ineffective, in this example we only have two private resources: a cosmos database and a database, but imagine that in a real project with tens of resources and multiple environments, like dev, staging or prod.   
It becomes very difficult to manage having thousands of resolutions on your hosts file on your local machine.

Also every time you deploy a new resource you need to send the DNS resolution to everyone that is using the VPN, so that's not a good solution.


# Solution 2: Use a DNS Forwarder

Another option for the P2S VPN clients to be able to resolve Private Endpoint entries hosted on Azure Private DNS Zones is using a DNS Forwarder.

The main objectivo for having a DNS Formarder is to forward DNS queries to Azure DNS.

Once you have a DNS forwarder/proxy is deployed on Azure, you can define the DNS server at the VNET level or set DNS Server configuration directly on client XLM profile.

Once everything is setup you will be able to resolve Private Endpoint entries from your VPN P2S clients.

To setup a DNS forwarder is quite simple mainly because you only need to forward queries to this IP: 168.63.129.16. This IP represents Azure DNS.   

Here's a example of a containerized Bind DNS Server that can be deployed on ACI or AKS:

- https://github.com/whiteducksoftware/az-dns-forwarder

If you take a look at the configuration of the Bind Server, you'll see that the only action it does is to forward queries to Azure DNS:

```javascript
options {
        recursion yes;
        allow-query { any; }; # do not expose externally
        forwarders {
            168.63.129.16;
        };
        forward only;
        dnssec-validation no; # needed for private dns zones
        auth-nxdomain no; # conform to RFC1035
        listen-on { any; };
};
```

If you're using a Virtual Machine with Windows, it is also quite straightforward to set it up as a DNS Forwarder, simply add the Azure DNS in the Forwarder Tab of the DNS Server.

<add-img>


Using an IaaS DNS Forwarder is the de facto solution nowadays to resolve private DNS zones.The DNS Forwarder/Proxy can be hosted on a virtual machine or on a container within ACI or AKS.

# Solution 3: Use Azure Private DNS Resolver

Until now Azure didn't provide any DNS server that is addressable from a VPN connection. 

The Azure DNS Private Resolver removes the needs to have additional DNS Forwarder to resolve private DNS zones.

Let's take a look at the previous example where I had a public app that was using a few private resources.
Here's how the diagram will look like after deploying an Azure Private DNS Resolver

<add-img>


If you want to try it by yourself, you can find this example on my [GitHub repository](https://github.com/karlospn/testing-private-dns-resolution-using-azure-dns-private-resolver).  It uses Terraform to deploy it. There is not much worth mentioning, probably the only interesting thing is that I'm using the [AzApi Terraform Provider](https://docs.microsoft.com/en-us/azure/developer/terraform/overview-azapi-provider) to provision the Private DNS Resolver.

The AzAPI provider is a thin layer on top of the Azure ARM REST APIs. The AzAPI provider enables you to manage any Azure resource type using any API version. This provider complements the AzureRM provider by enabling the management of new Azure resources and properties.

I'm using the AzApi provider because the Private DNS Resolver is not available right now on the official AzureRM Terraform provider.

If you're interested here's a snippet of how to provision a Private DNS Resolver

```javascript
resource "azapi_resource" "dns_resolver" {
    type      = "Microsoft.Network/dnsResolvers@2020-04-01-preview"
    name      = "resolver-${var.project_name}-${var.environment}"
    parent_id = azurerm_resource_group.rg_dns_test.id
    location  = azurerm_resource_group.rg_dns_test.location
  
    depends_on = [
        azapi_resource.subnet_dns_resolver_inbound_endpoint
    ]

    body =  jsonencode({
        properties = {

            virtualNetwork = {
                id = azurerm_virtual_network.vnet_dns_test.id
            }
        }
    })
    
    tags= var.default_tags
    response_export_values = ["*"]

}

resource "azapi_resource" "dns_resolver_inbound_endpoint" {
    type      = "Microsoft.Network/dnsResolvers/inboundEndpoints@2020-04-01-preview"
    name      = "resolver-inbound-endpoint-${var.project_name}-${var.environment}"
    parent_id = azapi_resource.dns_resolver.id
    location  = azurerm_resource_group.rg_dns_test.location
  
    depends_on = [
        azapi_resource.dns_resolver
    ]

    body =  jsonencode({
        properties = {
            ipConfigurations = [
                {
                    subnet = {
                        id = azapi_resource.subnet_dns_resolver_inbound_endpoint.id
                    },
                    privateIpAllocationMethod = "Dynamic"
                }
            ]
        }
    })
    
    tags= var.default_tags
    response_export_values = ["*"]
}
```