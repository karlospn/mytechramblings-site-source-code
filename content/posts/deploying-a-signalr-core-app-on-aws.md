---
title: "How to deploy a SignalR Core application on AWS"
date: 2022-07-11T16:58:40+02:00
tags: ["aws", "signalr", "dotnet"]
description: "In this post I'll try to talk a little bit about which possibilities are available when you want to deploy a SignalR Core application on AWS."
draft: true
---
Once every full moon I get asked about using SignalR app on AWS, usually the questions are along the lines of: 
- Can I use this service to deploy my app?
- Should I deploy my app on a Windows machine or can I use Linux one?
- I have a load balancer/API Gateway in front of my app and it doesn't work properly, what's wrong?
- I need to scale out the app, which service should I use? 
- I have multiple instances of my app and some messages seems to be getting lost, what's wrong?
- Can I use the Azure SignalR Service when my application is deployed in AWS?

I usually find that there is a lack of knowledge when trying to deploy a SignalR on AWS, so in this post I'll try to talk a little bit about which possibilities are available when you want to deploy a **SignalR Core** application on AWS.   

# **Compute**

## **1. AWS ECS**

## **2. AWS EC2**

## **3. AWS EKS**

# **Network**

## **1. AWS Application Load Balancer**

## **2. AWS WebSocket Api Gateway**

# **Backplane**

A SignalR application needs to keep track of ALL connected clients, which creates a problem in an environment where applications can scale out either automatically or manually like AWS.

On the diagram below we have 2 instances of a SignalR Core application behind a load balancer. When client A interacts with the application it gets routed to instance A. When client B interacts with the application it gets routed to instance B.    
In this scenario instance A is unaware of the connected clients on instance B, and viceversa, so if a client who is connected to instance A tries to broadcast a message, it will only be send to the clients connected to that instance.

<add diagram>

This is where a backplane becomes necessary. When a client makes a connection, the connection information is passed to the backplane. When a server wants to send a message to all clients, it sends to the backplane. The backplane knows all connected clients and which servers they're on. It sends the message to all clients via their respective servers.

<add diagram>

Exists a few AWS services that can be used as a SignalR backplane, and in the following sections we're going to talk about when and how to use them.

## **1. AWS ElastiCache**

Amazon ElastiCache is a fully managed, in-memory caching service. You can use ElastiCache for caching or as a primary data store for use cases that don't require durability like session stores, gaming leaderboards, streaming, and analytics. 
ElastiCache is compatible with Redis and Memcached. 

The SignalR Redis backplane uses the Redis pub/sub feature to forward messages to other servers. When a client makes a connection, the connection information is passed to the backplane. When a server wants to send a message to all clients, it sends to the backplane. The backplane knows all connected clients and which servers they're on. It sends the message to all clients via their respective servers.

Using a Redis backplane is the recommended way for scaling-out apps hosted on AWS and ElastiCache works fine with SignalR Core and it's pretty simple to use.

### **Usage**

1. Install the `Microsoft.AspNetCore.SignalR.StackExchangeRedis` NuGet package.
2. In `ConfigureServices` in `Startup.cs`, configure SignalR with `.AddStackExchangeRedis()`:


```csharp
services
    .AddSignalR()
    .AddStackExchangeRedis("connection_string");
```

If you're using one ElastiCache server for multiple SignalR apps, use a different channel prefix for each SignalR app. Like this

```csharp
services
    .AddSignalR()
    .AddStackExchangeRedis("connection_string", opts => {
        opts.Configuration.ChannelPrefix = "ChatRoom";
    });
```

Setting a channel prefix isolates one SignalR app from others that use different channel prefixes.    

If you don't assign different prefixes, a message sent from one app to all of its own clients will go to all clients of all apps that use the Redis server as a backplane.

## **2. AWS MemoryDb**

Amazon MemoryDB for Redis is a another Redis-compatible, in-memory database service that is based in part on the open source Redis platform.

### **What is the difference between MemoryDb or ElasticCache?  Which one should I use?**

The difference between Amazon ElastiCache and MemoryDB is that the former is intended as an in-memory cache that works alongside a primary database, whereas MemoryDB is a full database service in itself that is designed to operate on its own.   

We could say that MemoryDB for Redis is AWS's answer to Redis Enterprise and it exists to try to fill the gap for customers seeking a durable counterpart to ElastiCache. Consider Amazon MemoryDB, in essence, a premium tier of ElastiCache for Redis.

Both MemoryDb and ElastiCache supports the Redis pub/sub feature, so you can use both services as a SignalR backplane.   
I tend to prefer ElastiCache over MemoryDb, for two reasons:
- ElastiCache is cheaper.
- The extra features that MemoryDb has over ElastiCache, like durability and persistence, are not really useful for SignalR.
 

### **Usage**

If you want to use MemoryDb with SignalR Core go read the ElastiCache "Usage" section, because the use is exactly the same for both services.

The only thing worth mentioning is that when setting a MemoryDb Redis Cluster you'll find that authentication is required, meanwhile with on ElastiCache is optional. 

Keep that in mind when configuring SignalR on your application.

```csharp
services
    .AddSignalR()
    .AddStackExchangeRedis("connection_string", opts => {
        opts.Configuration.ChannelPrefix = "ChatRoom";
        opts.Configuration.User = "user1";
        opts.Configuration.Password = "pasword12345!";
        opts.Configuration.Ssl = true;
    });
```


### **Usage**

The use is the same as the one used for ElastiCache.

## **3. Amazon RDS for SQL Server**

Amazon RDS for SQL Server makes it easy to set up, operate, and scale SQL Servers in the cloud.

This is not the right solution for applications with a need for very high throughput, or very high degrees of scale-out. Consider to use one of the Redis services (ElasticCache or MemoryDb)  for such cases. 

There is no official Microsoft implementation for that, if you want to use it you'll need to use the `IntelliTect.AspNetCore.SignalR.SqlServer` NuGet package.   
More info about this package here: 
- https://github.com/IntelliTect/IntelliTect.AspNetCore.SignalR.SqlServer

### **Usage**

1. Install the `IntelliTect.AspNetCore.SignalR.SqlServer` NuGet package.
2. In `ConfigureServices` in `Startup.cs`, configure SignalR with `.UseSqlServer()`:


Simple configuration:
```csharp
services
    .AddSignalR()
    .AddSqlServer(Configuration.GetConnectionString("Default"));
```

Advanced configuration:

```csharp
services
    .AddSignalR()
    .AddSqlServer(o =>
    {
        o.ConnectionString = Configuration.GetConnectionString("Default");
        // See above - attempts to enable Service Broker on the database at startup
        // if not already enabled. Default false, as this can hang if the database has other sessions.
        o.AutoEnableServiceBroker = true;
        // Every hub has its own message table(s). 
        // This determines the part of the table named that is derived from the hub name.
        // IF THIS IS NOT UNIQUE AMONG ALL HUBS, YOUR HUBS WILL COLLIDE AND MESSAGES MIX.
        o.TableSlugGenerator = hubType => hubType.Name;
        // The number of tables per Hub to use. Adding a few extra could increase throughput
        // by reducing table contention, but all servers must agree on the number of tables used.
        // If you find that you need to increase this, it is probably a hint that you need to switch to Redis.
        o.TableCount = 1;
        // The SQL Server schema to use for the backing tables for this backplane.
        o.SchemaName = "SignalRCore";
    });
```


## **4. Azure SignalR Service**

Azure SignalR Service is a managed backplane, it eliminates having to manage your own Redis/SQL Server instance.
This services has a few advantages over the Redis backplane alternative:

- Sticky sessions, also known as client affinity, is not required, because clients are immediately redirected to the Azure SignalR Service when they connect.
- A SignalR app can scale out based on the number of messages sent, while the Azure SignalR Service scales to handle any number of connections. For example, there could be thousands of clients, but if only a few messages per second are sent, the SignalR app won't need to scale out to multiple servers just to handle the connections themselves.
- A SignalR app won't use significantly more connection resources than a web app without SignalR.

Probably you're asking yourself why I'm talking about an Azure service, that's because you can use the Azure SignalR Service with a SignalR app hosted in AWS as long as visibility exists between the application and the service.   

The optimal solution when choosing a backplane service is having the backplane as close as possible to the app, having the backplane on another cloud provider seems far from ideal, so if you want to use it be mindful about network latency, throughput and the amount of data transferred outside of AWS. 

### **Usage**

1. Install the `Microsoft.Azure.SignalR` NuGet package.
2. In `ConfigureServices` in `Startup.cs`, configure SignalR with `.AddAzureSignalR()`:


Simple configuration:
```csharp
services
    .AddSignalR()
    .AddAzureSignalR("connection_string");
```
The ``AddAzureSignalR()`` method can be used without passing the connection string parameter, in this case it tries to use the default configuration key for the SignalR Service resource connection string: ``Azure:SignalR:ConnectionString``.


Advanced configuration:

```csharp
services
    .AddSignalR()
    .AddAzureSignalR(o =>
    {
        options.ConnectionCount = 10;
        // This option controls the initial count of connections per hub between application 
        // server and Azure SignalR Service. Usually keep it as the default value is enough. 
        // During runtime, the SDK might start new server connections for performance tuning or 
        // load balancing. When you have big number of clients, you can give it a larger number 
        // for better throughput.
        options.AccessTokenLifetime = TimeSpan.FromDays(1);
        // This option controls the valid lifetime of the access token, which is generated by
        // Service SDK for each client. The access token is returned in the response to client's 
        // negotiate request.
        options.ClaimsProvider = context => context.User.Claims;
        // This option controls what claims you want to associate with the client connection. It 
        // will be used when Service SDK generates access token for client in client's negotiate 
        // request. By default, all claims from HttpContext.User of the negotiate request will 
        // be reserved. They can be accessed at Hub.Context.User.
        options.GracefulShutdown.Mode = GracefulShutdownMode.WaitForClientsClose;
        // This option specifies the behavior after the app server receives a SIGINT (CTRL + C).
        options.GracefulShutdown.Timeout = TimeSpan.FromSeconds(10);
        // This option specifies the longest time in waiting for clients to be closed/migrated.
    });
```

# Example

On my GitHub account you can find a repository that contains a concrete example about how to deploy a SignalR Core App on AWS.

The source code can be found here:
- https://github.com/karlospn/deploy-signalr-core-app-on-aws

The repository contains a `/cdk` folder, where you can find an AWS CDK app, that creates the following infrastructure on AWS:

- A VPC with 10.55.0.0/16 CIDR range.
- An Application Load Balancer.
- A Fargate service.
- An ElasticCache instance.
- A NAT Gateway.

<add diagram>