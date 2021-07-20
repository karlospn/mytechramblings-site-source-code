---
title: "Common mistakes when trying to deploy a dotnet gRPC app to AWS ECS"
date: 2021-07-19T15:32:32+02:00
tags: ["dotnet", "grpc", "containers", "aws", "ecs"]
draft: true
---

> **Just show me the code**   
As always if you don’t care about the post I have upload a few examples on my [Github](https://github.com/karlospn/deploying-net-grpc-services-on-ecs-fargate).   

Nowadays creating a new dotnet gRPC application is pretty straightforward. From the developer standpoint the experience of creating a gRPC app it's quite similar as creating an API. 
Visual Studio also offers Intellisense support for gRPC services and proto files.

Lately I've been deploying a sizable amount of gRPC services to AWS ECS so I thought it might be useful to talk a little bit about some gotchas you might encounter.   
Some of the problems are specific of the dotnet implementation of gRPC and another ones are on the AWS side.

If people are interested in the gRPC theme maybe in the near future I write a more in-depth post about dotnet gRPC apps and AWS, but for now I want to focus in a few gotchas I have encountered this past weeks.   
In this post I will talk about these 4 themes:
- **gRPC healtchecks**: How to implement a gRPC health checking service on a dotnet gRPC app and how to use it as the ALB Target Group healthcheck endpoint.
- **Adding a gRPC endpoint into an existing HTTP REST WebApi**: It is really easy to add a gRPC service into an Http REST Api, but when you deploy this service into ECS it probably won't work. Why is that? What's the workaround?
- **What's the package attribute on the .proto file for**: The ``package`` attribute on the .proto file might seem useless on dotnet mainly because the ``csharp_namespace`` option takes precendence over it, but the gRPC implementation for the ALB uses the package attribute to route the gRPC calls to the appropriate target.
- **gRPC Reflection**: What is the easiest way to test a gRPC service that has been deployed into AWS ECS?

# 1. gRPC Healthchecks

One of the first things you need to do when building a gRPC app is to add a gRPC healthchecking service. You'll need it because the ALB Target Group needs a healthcheck endpoint to probe that the service is up and running.

gRPC Health checks are used to probe whether the server is able to handle rpcs. 

A gRPC service is used as the health checking mechanism for both client-to-server scenario and other control systems such as load-balancing. Since it is a GRPC service itself, doing a health check is in the same format as a normal rpc.

If you want to read the gRPC protocol standards, click [here](https://github.com/grpc/grpc/blob/master/doc/health-checking.md).    
This is the standard .proto file that a gRPC healthcheck service needs to implement:

```csharp
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;  // Used only by the Watch method.
  }
  ServingStatus status = 1;
}

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

The healthchecking protocol standard defines that a gRPC healthcheck service should respond with the following status code and value:

- A client can query the server’s health status by calling the ``Check`` method.    
For each request received a response must be sent back with an ``OK`` status and the status field should be set to ``SERVING`` or ``NOT_SERVING`` accordingly.   
If the service name is not registered, the server returns a ``NOT_FOUND`` gRPC status.

- A client can also call the ``Watch`` method to perform a streaming health-check. The server will immediately send back a message indicating the current serving status. It will then subsequently send a new message whenever the service's serving status changes.

We could implement the healthchecking proto definition by ourselves, but it also exists the ``Grpc.HealthCheck`` NuGet package that does some heavy lifting for us. This nuget package contains a implementation of the ``Check`` and ``Watch`` methods and also contains the definitions used to send a request and reply to the healthcheck service.  

We could install and start using the package directly, but in .NET Core we have a rich offering of healthcheck implementations (https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks) that uses the ``AspNetCore.Diagnostics.HealthChecks``  NuGet as a base and I want to keep using them.   
To use the ``AspNetCore.Diagnostics.HealthChecks`` in combination with the ``Grpc.HealthCheck`` package you have to override the gRPC healthchecking service that the ``Grpc.HealthCheck`` NuGet offers.   
If you take a look at the implementation of the ``Check`` method in the ``Grpc.Healthcheck`` package, you'll see that the method can be overriden

```csharp
/// <summary>
/// Performs a health status check.
/// </summary>
/// <param name="request">The check request.</param>
/// <param name="context">The call context.</param>
/// <returns>The asynchronous response.</returns>
public override Task<HealthCheckResponse> Check(HealthCheckRequest request, ServerCallContext context)
{
    HealthCheckResponse response = GetHealthCheckResponse(request.Service, throwOnNotFound: true);

    return Task.FromResult(response);
}
```
So we can override the ``Check`` method and use the ``HealthCheckService`` from the ``AspNetCore.Diagnostics.HealthChecks`` to execute the .NET Core healthchecks.   
Here's the result:

```csharp
using System;
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Health.V1;
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace Server.gRPC
{
    public class GrpcHealthCheckService: Health.HealthBase
    {
        private readonly HealthCheckService _healthCheckService;

        public GrpcHealthCheckService(HealthCheckService healthCheckService)
        {
            _healthCheckService = healthCheckService;
        }

        public override async Task<HealthCheckResponse> Check(
            HealthCheckRequest request, 
            ServerCallContext context)
        {
            Func<HealthCheckRegistration, bool> GetHealthCheckPredicate()
            {
                var tags = request.Service?.Split(";") ??
                           Array.Empty<string>();

                static bool PassAlways(HealthCheckRegistration _) => true;

                if (tags.Length == 0)
                {
                    return PassAlways;
                }

                bool CheckContainsTags(HealthCheckRegistration healthCheck) =>
                    healthCheck.Tags.IsSupersetOf(tags);

                return CheckContainsTags;
            }

            var predicate = GetHealthCheckPredicate();

            var result = await _healthCheckService.CheckHealthAsync(predicate, 
            context.CancellationToken);

            var status = result.Status == HealthStatus.Healthy ? 
            ServingStatus.Serving : ServingStatus.NotServing;

            return new HealthCheckResponse
            {
                Status = status
            };
        }
    }
}
```
## Using the gRPC healthchecks as the ALB Target Group HealthCheck

As I have stated in the previous section this is what the gRPC healthcheck protocol says about the ``Check`` Method:

- A client can query the server’s health status by calling the ``Check`` method.    
For each request received a response must be sent back with an ``OK`` status and the status field should be set to ``SERVING`` or ``NOT_SERVING`` accordingly.   
If the service name is not registered, the server returns a ``NOT_FOUND`` gRPC status.

The protocol says that you should return an ``OK`` status code (gRPC status code = 0) if the service is healthy and also an ``OK`` status code (gRPC status code = 0) if the service is unhealthy. And you should use the status field to specify if the service is healthy or not.  

If you want to use the gRPC health check implementation from the previous section as the healthcheck endpoint for the Target Group, it won't work.  

The healthcheck protocol says that you should respond with the same status code if the service is healthy or unhealthy. You see the problem? The target group healthcheck uses the status code to ascertain that a service is healthy.   
We need to modify a little bit the implementation that I posted in the previous section. We are going to return a different status code based on the healthcheck status response.
- If the service is healthy return an ``OK`` status (gRPC status code = 0)
- If the service is unhealthy return a ``Unavailable`` status (gRPC status code = 14)

Here's a snippet of code showing the end result:

```csharp
using System;
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Health.V1;
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace Server.gRPC
{
    public class GrpcHealthCheckService: Health.HealthBase
    {
        private readonly HealthCheckService _healthCheckService;

        public GrpcHealthCheckService(HealthCheckService healthCheckService)
        {
            _healthCheckService = healthCheckService;
        }

        public override async Task<HealthCheckResponse> Check(HealthCheckRequest request, ServerCallContext context)
        {
            Func<HealthCheckRegistration, bool> GetHealthCheckPredicate()
            {
                var tags = request.Service?.Split(";") ??
                           Array.Empty<string>();

                static bool PassAlways(HealthCheckRegistration _) => true;

                if (tags.Length == 0)
                {
                    return PassAlways;
                }

                bool CheckContainsTags(HealthCheckRegistration healthCheck) =>
                    healthCheck.Tags.IsSupersetOf(tags);

                return CheckContainsTags;
            }

            var predicate = GetHealthCheckPredicate();

            var result = await _healthCheckService.CheckHealthAsync(predicate, context.CancellationToken);

            if (result.Status == HealthStatus.Unhealthy)
                throw new RpcException(new Status(StatusCode.Unavailable, "Service Unavailable"));

            return new HealthCheckResponse
            {
                Status = HealthCheckResponse.Types.ServingStatus.Serving
            };
        }
    }
}
```
After this little modification, we are ready to start using this healthcheck service as a Target group healthcheck endpoint.

# 2. Adding a gRPC endpoint into an existing HTTP REST WebApi

Image you have an existing dotnet HTTP WebApi and now you want to add a gRPC service into this API, so your clients can consume your api via JSON REST or via gRPC.       
gRPC uses HTTP/2 as its transfer protocol, but obviously you need to maintain support for HTTP/1.1 for your current clients.

## Kestrel HTTP protocols

 Using the ``Protocols`` property you can specify which HTTP Protocol Kestrel is going to use.

The possible values are the following ones:

- ``Http1``: Accept HTTP/1.1 only. Can be used with or without TLS.
- ``Http2``: Accept HTTP/2 only. May be used without TLS only if the client supports a Prior Knowledge mode.
- ``Http1AndHttp2``: Accept HTTP/1.1 and HTTP/2.*HTTP/2 requires the client to select HTTP/2 in the TLS Application-Layer Protocol Negotiation (ALPN) handshake; otherwise, the connection defaults to HTTP/1.1.
  
The default ``Protocols`` value for any endpoint is ``HttpProtocols.Http1AndHttp2``

You can set the HTTP protocol via config file or directly in code.   
The following ``Program.cs`` example establishes HTTP/1.1 for the 5001 port and HTTP/2 for the 5002 port.

```csharp
  public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureKestrel(options =>
                {
                    options.Listen(IPAddress.Any, 5001, listenOptions =>
                    {
                        listenOptions.Protocols = HttpProtocols.Http1;
                    });
                    options.Listen(IPAddress.Any, 5002, listenOptions =>
                    {
                        listenOptions.Protocols = HttpProtocols.Http2;
                    });
                });
                webBuilder.UseStartup<Startup>();
            });
```

The following ``appsettings.json`` example establishes HTTP/1.1 as the default connection protocol for all endpoints:

```javascript
{
  "Kestrel": {
    "EndpointDefaults": {
      "Protocols": "Http1"
    }
  }
}
```
Protocols specified in code override values set by configuration.

## Configuring Kestrel to work with HTTP/1.1 and HTTP/2

After reading the previous section you'll think that the solution is quite obvious: set on Kestrel the ``Protocols`` attribute to ``HttpProtocols.Http1AndHttp2``. After deploying the application to AWS ECS you'll find that it works for the clients that are using HTTP/1.1 to communicate with your API but it won't work for the clients that are using HTTP/2.    

Why is that? This is because when you set Kestrel to ``Http1AndHttp2``, it requires the client to select HTTP/2 in the TLS ALPN handshake; otherwise, the connection defaults to HTTP/1.1.   
Simply put if you set Kestrel to ``Http1AndHttp2``  and want to communicate via HTTP/2 you'll need to use TLS end-to-end.   
With AWS ECS we are doing a TLS termination in the Load Balancer, so the communication between the ALB and the ECS Service is not using TLS, so it will default to HTTP/1.1 and it won't work.

There are a few workarounds available:
- **Use TLS end-to-end**. I discarded immediately this option because I don't want to manage TLS certificates at Kestrel level.
  
- **Use HTTP/2 only**. If you set Kestrel as ``Http2`` you don't need to use TLS because ALPN is not required, but this is not a feasible option because we are breaking compatibility with the clients that are using HTTP/1.1.
  
- **Set Kestrel to listen for 2 differents ports. Each port will use a different Http protocol**. This is probably the best of the 3 options. To use this option you'll need to modify the ``Program.cs`` like this:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureKestrel(options =>
                {
                    options.Listen(IPAddress.Any, 5001, listenOptions =>
                    {
                        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;
                    });
                    options.Listen(IPAddress.Any, 5002, listenOptions =>
                    {
                        listenOptions.Protocols = HttpProtocols.Http2;
                    });
                });
                webBuilder.UseStartup<Startup>();
            });
}
```
- Kestrel will listen on port 5001 for Http/1.1 connections and on port 5002 for Http/2 connections.

## Hosting gRPC services and HTTP services using a single ALB

In the previous section we have set Kestrel to use 2 differents ports.

- Kestrel will listen on port 5001 for Http/1.1 connections and on port 5002 for Http/2 connections.

To host the WebApi you'll find that you need to create 2 Target Groups.

- One target group that routes the HTTP requests.
- One target group that routes the gRPC requests.
  
 Both Target Groups will be routing requests to the same ECS service.

 ![alb-route-grpc-http-requests](/img/alb-route-grpc-http-requests.png)


The clients will send both HTTP/1.1 and HTTP/2 requests to the same listener of the Application Load Balancer and using the path routing we will route the traffic to the Http or the gRPC target group.

Here's how the ALB Listener rule looks like:
- If the path is /api/* route the requests to the Http Target Group
- If the path is /greet.* route the requests to the gRPC Target Group.

![http-grpc-alb-rules.png](/img/http-grpc-alb-rules.png)


# 3. The Package attribute on proto files and ALB path routing.

When you create a new gRPC service you need to be aware of the package atribute. It might seem that it is useless mainly because the ``csharp_namespace`` option takes precendence over it.

- The load balancer parses gRPC requests and routes the gRPC calls to the appropriate target groups based on the package, service, and method.

Let's use the default greet proto file as an example.    
Here's the content of the greet proto file:

```csharp
syntax = "proto3";

option csharp_namespace = "Server.Grpc";

package greet;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply);
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings.
message HelloReply {
  string message = 1;
}
```
You can build 

/greet.* - routes all requests to the greet package.
/greet.Greeter/* - routes all the requests to the Greeter service in the greet package.
/greet.Greeter/SayHello - routes all the requests to the SayHello method implemented in the Greeter service in the greet package.


# 4. gRPC Reflection

Testing a gRPC application is far more troubling than testing a JSON HTTP Api and that's exactly what gRPC reflection could solve for us.

gRPC reflection is an extension for gRPC servers to assist clients in runtime construction of requests without having the proto files precompiled into the client.

gRPC reflection tries to answer the following questions:

- What methods does a service export?
- For a particular method, how do we call it? 
- What are the names of the methods, are those methods unary or streaming?
- What are the types of the argument and result?

In dotnet you can enable gRPC Reflection using the ``Grpc.AspNetCore.Server.Reflection`` package.
To configure reflection in an app:
- Add a ``Grpc.AspNetCore.Server.Reflection`` package reference.
- Register reflection in Startup.cs:
  - ``AddGrpcReflection`` to register services that enable reflection.
  - ``MapGrpcReflectionService`` to add a reflection service endpoint.

```csharp
namespace Server.Grpc
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddGrpc();
            services.AddGrpcReflection();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGrpcService<GreeterService>();
                endpoints.MapGrpcReflectionService();

                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync("Communication with gRPC endpoints must be made through a gRPC client.");
                });
            });
        }
    }
}
```
When gRPC reflection is set up:
- A gRPC reflection service is added to the server app.
- gRPC services are still called from the client. Reflection only enables service discovery and doesn't bypass server-side security. Endpoints protected by authentication and authorization require the caller to pass credentials for the endpoint to be called successfully.

Now we can invoke our gRPC server using client tools that support gRPC reflection. Those tools will call the reflection service to discover services hosted by the server.

Here's a short list of tools that you might be interested in trying:
- gRPCurl: https://github.com/fullstorydev/grpcurl
- gRPC UI: https://github.com/fullstorydev/grpcui
- BloomRPC: https://github.com/uw-labs/bloomrpc
- dotnet-grpc-cli: https://github.com/bjorkstromm/dotnet-grpc-cli


# Useful links

- https://codevalue.com/grpc-health-checks-with-net-core-kubernetes/
- https://exampleloadbalancer.com/albgrpc_demo.html
- https://grpc.io/

_I want to give a special thanks to the creator of the codevalue blog page, because the gRPC healthcheck implementation is based on his work._