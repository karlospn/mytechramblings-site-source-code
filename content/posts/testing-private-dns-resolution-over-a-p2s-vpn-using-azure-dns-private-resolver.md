---
title: "Testing private DNS resolution over an Azure P2S VPN connection"
date: 2022-06-02T21:54:47+02:00
draft: true
tags: ["azure", "cloud", "terraform", "dns"]
description: "The purpose of this post is to try out the new Azure DNS Private Resolver resource. To test it, we're going to try to solve one of the current issues that Azure VPN P2S has right now. The problem is when connected over an Azure P2S VPN connection the private DNS zone resolution does not work, because it tries to connect using the public endpoint instead of the private endpoint private IP. This becomes quite problematic when you're using private endpoints to secure some private resources, because there is no easy way to resolve the private endpoint DNS when connected to a P2S VPN."
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/testing-private-dns-resolution-using-azure-dns-private-resolver).

I have had this long time issue with DNS resolution of private resources on Azure when working over a Point-to-Site VPN connection.

The problem I'm talking about is that **a private DNS zone will not work over an Azure P2S VPN connection**, which means that by default you cannot resolve a private DNS zone when connected over a P2S VPN.   
This becomes quite problematic when you're using private endpoints to secure some private resources, because there is no easy way to resolve the private endpoint DNS when connected to a VPN.

Be aware that **this problem also happens if you try to resolve a private resource from an on-premise network connected via ExpressRoute or VPN S2S.**.

At first I didn't plan to write about it, mainly because I didn't find it interesting enough, but the new [Azure DNS Private Resolver](https://azure.microsoft.com/en-us/blog/announcing-azure-dns-private-resolver-now-in-preview/) resource is a potential solution to this issue and I wanted to test it.

So at the end I thought that writing a little bit about it, might be helpful to someone.

# What is the problem when trying to resolve a private resource over an Azure VPN P2S

First of all, let me explain a little more in-depth what this problem is all about.   
I'm going to start showing you a simplified example, so you can have a better understanding of what's the issue here.

![example-diagram](/img/vpn-p2s-problem-diagram.png)

As you can see this is a pretty basic setup, we have a public app where the customers can connect via public internet. This public app uses a few private resources, to be more precise, it makes a call to another app and it also needs a database to persists data.

It makes no sense that the database and the second app could be accessed from anywhere on the internet, so we're going to make them private. To make both resources private we're going to Azure Private Endpoints.

When a private endpoint is created, Azure changes the public name resolution by adding another CNAME record pointing towards the dedicated FQDN of the private endpoint.    
By default, it also creates a private DNS zone, corresponding to the ``privatelink`` subdomain, with the DNS A resource record for the private endpoint.

When you resolve the resource endpoint URL from outside the VNet with the private endpoint, it resolves to the public endpoint of the resoucee. When resolved from the VNet hosting the private endpoint, the resource endpoint URL resolves to the private endpoint's IP address.

It might sound a little bit complicated, but it's quite simple, let me show you a quick example to help you better understand how private endpoints works:

## Example about how private endpoint works 

- I have created a new App Service, and it is publicly accessible from the internet.
  
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

- Now I create a private endpoint to make the app private. As you can see Azure changes the public name resolution by adding another CNAME record pointing towards the dedicated FQDN of the private endpoint.

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

- When the private endpoint was created a private DNS zone was created with the corresponding ``privatelink`` subdomain.    
This DNS zone contains an A-record that points the private endpoint address to the private IP that is associated for the resource. Also this private DNS zone has been attached to the VNET.

![private-endpoint-dns-zone](/img/private-endpoint-dns-zone.png)

- Now when you try to resolve it from a client that's inside the VNET, it will response with the private IP address.

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

- If you try to resolve it from a client that's outside of the VNET, it will respond with the public IP address. Bear in mind that if you try to invoke the app from outside the VNET it will thrown an error.

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

Now, image the scenario where someone needs to access those private resources, for that purpose you put in place a Point-to-Site VPN, but if you try to invoke some of the private resources that are using a private endpoint while connected to the VPN you'll get an error.

```bash
$ curl -I dns-resolver-test.azurewebsites.net
HTTP/1.1 403 Ip Forbidden
Content-Length: 1895
Content-Type: text/html
x-ms-forbidden-ip: 71.11.124.148
Date: Sun, 05 Jun 2022 17:24:36 GMT
```

That's because when connected over an Azure P2S VPN connection the private DNS zone resolution does not work, because it tries to connect using the public endpoint instead of the private endpoint private IP.

The same problem happens if you're trying to access a private Azure resource from an on-premise network connected to Azure via Express Route or VPN.

To solve it, there are a few solutions available and in the next sections I'm going to talk about it.

# Solution 1: Modify the hosts file on you local machine

The Hosts file is used to override the DNS system so that a browser or other application can be redirected to a specific IP address.

This is the easiest solution if you want to invoke a private azure resource, you'll only need to modify the host file in you machine to override the DNS resolution of the private services.

Here's an example of how to do it:

```bash
10.18.2.4 app-private-api-dns-resolver-test-dev.azurewebsites.net
10.18.2.4 app-private-api-dns-resolver-test-dev.scm.azurewebsites.net
10.18.2.5 cosmos-dns-resolver-test-dev.mongo.cosmos.azure.com
10.18.2.6 cosmos-dns-resolver-test-dev-westeurope.mongo.cosmos.azure.com
```
Those services instead of resolving to the public endpoint, they will resolve to the private endpoint private IP. 

This solution works but is very ineffective.    
In the example above we only had two private resources: a Cosmos database and an App Service, but imagine that in a real project with tens of resources and multiple environments (dev, staging, prod, ...).
The hosts file ends up having thousands of private IPs and becomes quite cumbersome to manage it.

And also every time you create a new private resource on Azure you need to update the host file and also signal everyone that is using the VPN that a new resource needs to be added in their hosts file.

It might work with small projects where a small group of people need to access thoses resources via VPN, but it's not a good solution.


# Solution 2: Use a DNS Forwarder

Another option for the P2S VPN clients to be able to resolve Private Endpoint entries hosted on Azure Private DNS Zones is using a DNS Forwarder.

The main objectivo for having a DNS Formarder is to forward DNS queries to Azure DNS.

Once you have a DNS forwarder/proxy deployed on Azure, you can define the DNS server at the VNET level or set DNS Server configuration directly on client XLM profile.

Once everything is setup you will be able to resolve Private Endpoint entries from your VPN P2S clients. 

The DNS Forwarder/Proxy can be hosted on a virtual machine or on a container service like ACI or AKS.

Setup a DNS forwarder is simple, mainly because you only need to forward queries to the Azure DNS IP: 168.63.129.16.

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

If you're using a Windows Virtual Machine, it is also quite straightforward to set it up. Simply add the Azure DNS IP in the Forwarder Tab of the DNS Server.

![windows-dns-manager](img/windows-dns-manager-forwarders.png)

Using a DNS Forwarder is the de facto solution nowadays to resolve private DNS zones and it works fine.    

If you want a more in-depth documentation about it you can go here:
- https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#dns-configuration-scenarios


The inconvenients with this approach is that the DNS forwarder/proxy ends up being another piece of software that needs to be setup and mantain properly. The maintain part is even worse if you're using a VM insted of a containerized approach because the underlying OS updates became your problem. 

Also you need to set the DNS forwarder as a DNS at the VNET level, so it needs to have a high availability.

Using a DNS forwarder is the de facto solution when we want to resolve a private DNS zone when connected to a VPN, but nowadays seems that a better solution is available.

# Solution 3: Use Azure Private DNS Resolver

Until now Azure didn't provide any DNS server that is addressable from a VPN connection. 

The Azure DNS Private Resolver removes the needs to have additional DNS Forwarder to resolve private DNS zones.

Let's take a look at the previous example where I had a public app that was using a few private resources.
Here's how the diagram will look like after deploying an Azure Private DNS Resolver

![vpn-p2s-private-dns-resolver](img/vpn-p2s-private-dns-resolver.png)

As you can see now we deployed a DNS Resolver Inbound endpoint.    

An inbound endpoint enables name resolution from on-premises or other private locations via an IP address that is part of your private virtual network address space.   

This endpoint requires a subnet in the VNet where it’s provisioned. The subnet can only be delegated to ``Microsoft.Network/dnsResolvers`` and can't be used for other services.   

DNS queries received by the inbound endpoint will ingress to Azure. You can resolve names in scenarios where you have Private DNS Zones, including VMs that are using auto registration, or Private Link enabled services.

Also the **DNS Resolver Inbound endpoint needs to be set as a DNS Server in the VNET**.

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