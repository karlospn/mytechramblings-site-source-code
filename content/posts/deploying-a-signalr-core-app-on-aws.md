---
title: "How to deploy a SignalR Core application on AWS"
date: 2022-07-11T16:58:40+02:00
tags: ["aws", "signalr", "dotnet"]
description: "In this post I'll try to talk a little bit about which possibilities are available when you want to deploy a SignalR Core application on AWS."
draft: true
---
Once every full moon I get asked about using SignalR app on AWS, usually the questions are along the lines of: 
- Can I use this service to deploy my SignalR app?
- Should I deploy my app on a Windows VM? Can I use a Linux VM?
- I have a load balancer/API Gateway in front of my SignalR app and it doesn't work properly, what's wrong?
- I need to scale out the SignalR app, which service should I use for the backplane? 
- I have multiple instances of my app and some messages seems to be getting lost, what's wrong?
- Can I use the Azure SignalR Service when my application is deployed in AWS?

I usually find that there is a lack of knowledge when trying to deploy a SignalR on AWS, so in this post I'll try to talk a little bit about which possibilities are available when you want to deploy a **SignalR Core** application on AWS.   

# **Network**

In this first section I want to talk about AWS services that can control and balance network traffic and how to use them with SignalR.

The services we're going to review are the following ones:
- AWS Application Load Balancer
- AWS WebSocket Api Gateway

## **1. AWS Application Load Balancer**

### **WebSockets**

The ALB allows us to forward the traffic to an application by leveraging a couple of resources:
- A Target Group.
- A Listener.

The Target Group defines where to send the traffic. We can configure the port and protocol of the traffic that is forwarded, as well as a health check, so that the load balancer doesn't send traffic to a dead service.

The Listener defines how the load balancer gets its traffic from outside. That's where you define the port and the protocol to reach the load balancer, and what is the default behavior for the traffic hitting that listener. 

Amazon ALB Listeners only offer HTTP or HTTPS protocol, but the good news is that WebSocket initially contacts the server with HTTP if you use ws:// or HTTPS if you use wss:// your server will then reply with ``101 Switching Protocols``, telling the client to upgrade to a WebSocket connection.

SignalR uses the WebSocket transport where available and falls back to older transports where necessary, but you don't need to do anything on an ALB to start using them.

### **Sticky sessions**

SignalR requires that all HTTP requests for a specific connection be handled by the same instance. When a SignalR app is running behind a load balancer with multiple instances of the same service, "sticky sessions" must be used.

The only circumstances in which sticky sessions are not required are:

- When hosting on a single instance of the application.
- When using the Azure SignalR Service (later we'll talk a little bit about it).
- When all clients are configured to only use WebSockets.

The next code snippet shows how to create a Target Group with sticky sesion enabled.

```csharp
new ApplicationTargetGroup(this,
    "tg-app-ecs-signalr-core-demo",
    new ApplicationTargetGroupProps
    {
        TargetGroupName = "tg-app-ecs-signalr-core-demo",
        Vpc = vpc,
        TargetType = TargetType.IP,
        ProtocolVersion = ApplicationProtocolVersion.HTTP1,
        StickinessCookieDuration = Duration.Days(1),
        HealthCheck = new HealthCheck
        {
            Protocol = Amazon.CDK.AWS.ElasticLoadBalancingV2.Protocol.HTTP,
            HealthyThresholdCount = 3,
            Path = "/health",
            Port = "80",
            Interval = Duration.Millis(10000),
            Timeout = Duration.Millis(8000),
            UnhealthyThresholdCount = 10,
            HealthyHttpCodes = "200"
        },
        Port = 80,
        Targets = new IApplicationLoadBalancerTarget[] { target }
    });
```

## **2. AWS WebSocket Api Gateway**

In API Gateway you can create a WebSocket API as a stateful frontend for an AWS service or for an HTTP endpoint.   
The WebSocket API invokes your backend based on the content of the messages it receives from client apps.

I was **unable to make a SignalR Core app work with AWS WebSocket Api Gateway**.
   
The problem lies with how the API Gateway handles the connections and route messages. The API Gateway uses three predefined routes to communicate with the app: `$connect`, `$disconnect`, and ``$default`. In addition, you can create custom routes.

- API Gateway calls the `$connect` route when a persistent connection between the client and a WebSocket API is being initiated.
- API Gateway calls the `$disconnect` route when the client or the server disconnects from the API.
- API Gateway calls a custom route after the route selection expression is evaluated against the message if a matching route is found; the match determines which integration is invoked.
- API Gateway calls the `$default` route if the route selection expression cannot be evaluated against the message or if no matching route is found.

This is specific way to handle connections and route messages does not sit well with SignalR.   
In fact, it seems that integrating a SignalR Core app with an AWS WebSocket Api Gateway is not possible.    

If someone knows a way to do it, contact me.


# **Backplane**

A SignalR application needs to keep track of ALL connected clients, which creates a problem in an environment where applications can scale out either automatically or manually like AWS.

On the diagram below we have 2 instances of a SignalR Core application behind a load balancer. When client A interacts with the application it gets routed to instance A. When client B interacts with the application it gets routed to instance B.    
In this scenario instance A is unaware of the connected clients on instance B, and viceversa, so if a client who is connected to instance A tries to broadcast a message, it will only be send to the clients connected to that instance.

<add diagram>

This is where a backplane becomes necessary. When a client makes a connection, the connection information is passed to the backplane. When a server wants to send a message to all clients, it sends to the backplane. The backplane knows all connected clients and which servers they're on. It sends the message to all clients via their respective servers.

<add diagram>

In  this section will see a few AWS services that can be used as a SignalR backplane, those services are the following ones:
- AWS ElastiCache for Redis.
- AWS MemoryDb.
- AWS RDS for SQL Server.
- Azure SignalR Service.

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

# **Compute**

In the last section will see in which AWS services you can use to deploy your app:
- AWS ECS
- AWS EC2
- AWS EKS

## **1. AWS ECS**

Nothing worth mentioning here.   
Just build the container like any other .NET6 application and deploy it. No extra steps or configuration required here.

## **2. AWS EC2**

The part where you deploy the SignalR app into the server is business as usual, it's like deploying another .NET6 application.

### **Using a Windows EC2 with IIS as WebServer**
- Enable WebSockets. 
  - You can do it in a single command using Powershell: ``Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebSockets``
- Sticky session should be set in the load balancer.

### **Using a Linux EC2 with Apache as WebServer**
- Create a VirtualHost configuration. Here's an example:
```xml
<VirtualHost *:80>
    ProxyPass / ws://127.0.0.1:5000/
    ProxyPassReverse / ws://127.0.0.1:5000/
</VirtualHost>
```
"127.0.0.1:5000" is the local address where the SignalR app is running.

You can also set the VirtualHost configuration like this:

```xml
<VirtualHost *:80>
    ProxyPass / http://127.0.0.1:5000/
    ProxyPassReverse / http://127.0.0.1:5000/
</VirtualHost>
```
In this case it will use SSE instead of WebSockets.

- Sticky session should be set in the load balancer.

### **Using a Linux EC2 with NGINX as WebServer**
- Create a new configuration on the "http" section. Here's an example:
```javascript
map $http_connection $connection_upgrade {
    "~*Upgrade" $http_connection;
    default keep-alive;
}
```

- Create a new "Location" configuration on the "server" section. Here's an example:

```javascript
location / {
    # App server url
    proxy_pass http://localhost:5000;

    # Configuration for WebSockets
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_cache off;
    # WebSockets were implemented after http/1.0
    proxy_http_version 1.1;

    # Configuration for ServerSentEvents
    proxy_buffering off;

    # Configuration for LongPolling or if your KeepAliveInterval is longer than 60 seconds
    proxy_read_timeout 100s;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```
- Sticky session should be set in the load balancer.

## **3. AWS EKS**



# Example

On my GitHub account you can find a repository that contains a quick example about how to deploy a SignalR Core App on AWS.

The source code can be found here:
- https://github.com/karlospn/deploy-signalr-core-app-on-aws

The repository contains a `/cdk` folder, where you can find an AWS CDK app, that creates the following infrastructure on AWS:

- A VPC with 10.55.0.0/16 CIDR range.
- An Application Load Balancer.
- A Fargate service.
- An ElasticCache instance.
- A NAT Gateway.

<add diagram>