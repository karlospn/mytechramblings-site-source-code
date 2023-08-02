---
title: "Back to .NET basics: How you should be using HttpClient"
date: 2023-08-02T10:07:52+02:00
draft: true
tags: ["dotnet", "networking", "http", "basics"]
description: "In this post, we will explore several real-world scenarios where HttpClient is employed. For each scenario, we will discuss the reasons behind its proper or improper usage, using the help of the netstat command."
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/dotnet-httpclient-basic-usage-scenarios).

**If you are a .NET veteran, this post is probably not intended for you**.

I'm well aware that there are a ton of great articles (and probably better than this one) on the Internet, explaining exactly how you should properly use ``HttpClient`` with .NET.    
However, the truth is, even with so many resources available, I still come across many cases where its usage is incorrect.

Therefore, I have decided to write a quick post about the most common scenarios to use ``HttpClient`` nowadays.

I don't intend to provide a theoretical explanation of how ``HttpClient`` works internally. Instead, my goal is to make **a post that highlights various common scenarios where HttpClient is utilized and discuss the reasons behind its appropriate or inappropriate usage.**

# **netstat command**

The ``netstat`` command is a networking tool that allows us to investigate active network connections on a system, and in this post I will make extensive use of it to monitor ``HttpClient`` TCP connections.

``HttpClient`` is used to make HTTP requests to web servers and APIs, and it relies on underlying network connections to perform these tasks. By leveraging the ``netstat`` command, we can gain insights into the active connections created by ``HttpClient``, helping us identify potential issues.

To investigate the active TCP connections that ``HttpClient`` creates using ``netstat``, you can open a command prompt or terminal and enter ``netstat -ano`` (the '-a' flag shows all connections, the '-n' flag displays IP addresses and port numbers, and the '-o' flag displays the associated process ID (PID) ).   
The output will provide a list of all active connections, along with their status, local and remote IP addresses, and associated PID process.

![httpclient-scenarios-netstat-output](/img/httpclient-scenarios-netstat-output.png)

Monitoring HttpClient connections using ``netstat`` can help you identify if your application is properly closing connections after use or if there are lingering connections that may lead to resource leaks. It can also reveal if there are connection failures, such as connections in a ``TIME_WAIT`` state, which might indicate issues with connection pooling or DNS resolution.

The next list shows the ``netstat`` states and their meanings:

- ``ESTABLISHED``: This state indicates that a connection is active and data is being exchanged between the local and remote systems. It signifies a successful connection between the client and server.
- ``TIME_WAIT``: After the connection is closed, it enters the ``TIME_WAIT`` state. This state ensures that any delayed packets from the previous connection are handled properly. It typically lasts for a few minutes before the connection is fully closed.
- ``CLOSE_WAIT``: This state occurs when the local application has closed the connection, but the remote system has not acknowledged the closure yet. It usually implies that the local application is waiting for the remote system to release the connection.
- ``FIN_WAIT_1``, ``FIN_WAIT_2``: These states occur during the process of closing a connection. ``FIN_WAIT_1`` means the local system has initiated the closure, while ``FIN_WAIT_2`` indicates the remote system has acknowledged the closure, and the local system is waiting for a final acknowledgment.
- ``LAST_ACK``: This state appears when the local system has initiated the closure, sent a FIN packet, and is waiting for the final acknowledgment from the remote system before the connection is fully closed.
- ``SYN_SENT``: In this state, the local system has sent a synchronization (SYN) packet to initiate a connection with the remote system but has not received a response yet.
- ``SYN_RECEIVED``: The ``SYN_RECEIVED`` state occurs on the server side when it receives a SYN packet from the client and sends back its ``SYN-ACK`` packet to acknowledge the connection request.
- ``LISTENING``: When a server application is in the ``LISTENING`` state, it is waiting and ready to accept incoming connection requests from clients.
- ``CLOSING``: This state occurs when the local system has initiated the closure of the connection, but the remote system is also trying to close the connection simultaneously.

# **Scenario 1: Create a new HttpClient for every incoming request**

## Source code

- A new ``HttpClient`` is instantiated every time a new request comes in.
- The ``HttpClient`` is not disposed after being used.

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioOneController : ControllerBase
{
    
    [HttpGet()]
    public async Task<ActionResult> Get()
    {
        var client = new HttpClient
        {
            BaseAddress = new Uri("https://jsonplaceholder.typicode.com/"),
            DefaultRequestHeaders = { { "accept", "application/json" } },
            Timeout = TimeSpan.FromSeconds(15)
        };

        var response = await client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode) 
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```

## netstat output

- Every time a new request comes in, a new TCP connection is created.
- TCP connections are not being neither re-used nor closed after being used, which means that they will hang for some time waiting for incoming data.

> _The next video shows how for every request made at the ``ScenarioOneController`` a new TCP connection is created_

{{< video src="/videos/httpclient-scenario1-established.mp4" type="video/mp4" preload="auto" >}}

- After 2 minutes (default idle timeout) of hanging around doing nothing, the TCP connections will be closed by the operating system and moved to a ``TIME_WAIT`` state. 

{{< video src="/videos/httpclient-scenario1-timewait.mp4" type="video/mp4" preload="auto" >}}

- The ``TIME_WAIT`` state is a normal part of the TCP connection termination process, and it occurs after a connection is closed. During this state, the socket remains in the system for a specific period to ensure that any delayed or out-of-order packets related to the closed connection do not interfere with new connections using the same port. The duration of the ``TIME_WAIT`` state can vary depending on the operating system and TCP implementation. 
- In most modern systems, the ``TIME_WAIT`` state typically lasts for 30 seconds to 2 minutes. 

{{< video src="/videos/httpclient-scenario1-timewait2.mp4" type="video/mp4" preload="auto" >}}


## Pros & cons of this scenario

### Pros
- None

### Cons
- A new ``HttpClient`` is being created every time a new request comes in, which means that the application has a unnecessary overhead from establishing a new TCP connection for every single request.
- If the app is under heavy load this approach can lead to an accumulation of TCP connections on a ``ESTABLISHED`` state or in a ``TIME_WAIT`` state, which can cause a port exhaustion problem.


# **Scenario 2: Create a new HttpClient for every incoming request and dispose of it after use**

## Source code

- A new ``HttpClient`` is instantiated every time a new request comes in.
- The ``HttpClient`` is disposed right after being used.

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioTwoController : ControllerBase
{
    
    [HttpGet()]
    public async Task<ActionResult> Get()
    {
        using var client = new HttpClient
        {
            BaseAddress = new Uri("https://jsonplaceholder.typicode.com/"),
            DefaultRequestHeaders = { { "accept", "application/json" } },
            Timeout = TimeSpan.FromSeconds(15)
        };

        var response = await client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```

## netstat output

- Every time a new request comes in, a new TCP connection is created.
- Like scenario 1, TCP connections are not being re-used either, but this time the TCP connections are being closed right after being used.
- The fact that the ``HttpClient`` gets disposed right away (because of the ``using`` parameter) causes the TCP connections to move directly to a ``TIME_WAIT`` state.

{{< video src="/videos/httpclient-scenario2.mp4" type="video/mp4" preload="auto" >}}

- During the ``TIME_WAIT`` state, the socket remains in the system for a specific period to ensure that any delayed or out-of-order packets related to the closed connection do not interfere with new connections using the same port. The ``TIME_WAIT`` state lasts for 30 seconds to 2 minutes depending on the operating system.

## Pros & cons of this scenario
### Pros
- In this scenario, it is less likely for the application to suffer from port exhaustion issues. In scenario 1, for each request, the TCP connection would remain in a ``ESTABLISHED`` state for a few minutes until the operating system forced it to close.    
Contrastingly, in scenario 2, as we are disposing of the HTTP client after its use, the connection is promptly closed, eliminating the period of time during which the connection was hanging around in a ``ESTABLISHED`` state doing nothing.

### Cons
- A new ``HttpClient`` is being created every time a new request comes in, which means that the application has a unnecessary overhead from establishing a new TCP connection every single time.

- In this scenario, while we have managed to eliminate the fact that TCP connections remain in a  ``ESTABLISHED`` state for a few minutes being idle, we are still creating a new TCP connection for every request the controller receives, which can still lead to issues of port exhaustion if the application experiences a high volume of traffic.


# **Scenario 3: Create a static HttpClient and use it for any incoming requests**

## Source code
- A ``static`` ``HttpClient`` instance is created once and reused for incoming requests.

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioThreeController : ControllerBase
{
    private static readonly HttpClient Client = new()
    {
        BaseAddress = new Uri("https://jsonplaceholder.typicode.com/"),
        DefaultRequestHeaders = { { "accept", "application/json" } },
        Timeout = TimeSpan.FromSeconds(15),
    };

    [HttpGet()]
    public async Task<ActionResult> Get()
    {

        var response = await Client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```
## netstat output

- Now, the TCP connections are being re-used.

{{< video src="/videos/httpclient-scenario3.mp4" type="video/mp4" preload="auto" >}}

- If the application is being idle for 2 minutes, then the TCP connection will get closed by the operating systen. The next request will force a new TCP connection to be created.
- By default in .NET, an idle TCP connection is closed after 2 minutes. If a connection is not currently being used to send a request, it's considered idle.

{{< video src="/videos/httpclient-scenario3-idle.mp4" type="video/mp4" preload="auto" >}}

- ``HttpClient`` only resolves DNS entries when a TCP connection is created.   
In the current scenario (where we employ a ``static`` or ``singleton``, long-lived ``HttpClient``), if the service we are invoking experiences a DNS modification, the established TCP connections will remain oblivious to this change.

> _In the upcoming video, I modify the hosts file on my computer to redirect the DNS address for ``jsonplaceholder.typicode.com`` to ``127.0.0.1``.    
The application should throw an error because there is nothing listening on ``127.0.0.1``capable of responding accordingly, but despite this change, the subsequent requests made to ``jsonplaceholder.typicode.com`` continue responding with a 200 OK status code, that's because the client remains unaware of the DNS change I made._

{{< video src="/videos/httpclient-scenario3-change-dns.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons of this scenario
### Pros 
- TCP connections are being reused, which further reduces the likelihood of experiencing a port exhaustion issue.    
If the rate of requests is very high, the operating system limit of available ports might still be exhausted, but the best way to minimize this issue is exactly what we're doing in this scenario, reusing ``HttpClient`` instances for as many HTTP requests as possible.

### Cons

> You'll see a lot of guidelines mentioning this DNS resolution issue when talking about ``HttpClient``, the truth is that if your app is calling a service where the DNS doesn't change at all, using this approach is perfectly fine.

- ``HttpClient`` only resolves DNS entries when a TCP connection is created. If DNS entries changes regularly, then the client won't notice those updates.    

# **Scenario 4: Create a static or singleton HttpClient with PooledConnectionLifetime and use it for any incoming requests**

## Source code
- A ``static`` ``HttpClient`` instance is created once and reused for incoming requests.
- The ``HttpClient`` is created using the ``PooledConnectionLifetime`` attribute. This attribute defines how long connections remain active when pooled. Once this lifetime expires, the connection will no longer be pooled or issued for future requests.  

> In the next code snippet, the ``PooledConnectionLifetime`` is set to 15 seconds, which means that TCP connections will cease to be re-issued and be closed after a maximum of 15 seconds. This is highly inefficient and it is only done for demo purposes.

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioFourController : ControllerBase
{
    private static readonly HttpClient Client = new(new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromSeconds(10)
    })
    {
        BaseAddress = new Uri("https://jsonplaceholder.typicode.com/"),
        DefaultRequestHeaders = { { "accept", "application/json" } },
        Timeout = TimeSpan.FromSeconds(15),
    };

    [HttpGet()]
    public async Task<ActionResult> Get()
    {

        var response = await Client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```
## netstat output

- TCP connections are being reused.
- If the application is being idle for some time, then the TCP connections will get closed by the operating system. The next request will create a new TCP connection.
- The ``PooledConnectionLifetime`` attribute is set to 10 seconds, which means that after 10 seconds the TCP connection will be closed and won't be re-used anymore. The next request will create a new TCP connection.

{{< video src="/videos/httpclient-scenario4.mp4" type="video/mp4" preload="auto" >}}

Do you remember that in scenario 3, I mentioned an issue with DNS resolution?    

DNS resolution only occurs when a TCP connection is created, which means that if the DNS changes after the TCP connection has been created, then the TCP connection is unaware of it.   

The solution to avoid this issue is to create **short-lived** TCP connections that can be reused. Thus, when the time specified by the ``PooledConnectionLifetime`` property is reached, the TCP connection is closed, and a new one is created, forcing DNS resolution to occur again.

In the next video, I modify the ``hosts`` file on my computer to redirect the DNS address for ``jsonplaceholder.typicode.com`` to ``127.0.0.1``.    

Since there is nothing listening on the ``127.0.0.1`` address capable of responding to those requests, after 15 seconds, the HTTP requests start faling with a 500 error. This occurs because the TCP connection has been closed, and a new one has been created, forcing DNS resolution to occur again.   

That's a huge difference from scenario 3, where the requests keep responding always 200 OK because the DNS resolution never occurred.   

{{< video src="/videos/httpclient-scenario4-dns-change.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons of this scenario
### Pros 
- TCP connections are being reused, which further reduces the likelihood of experiencing a port exhaustion issue. 
- It solves the DNS change issue mentioned on scenario 3.

### Cons
- There are no disadvantages in this scenario.

# **Scenario 4.1: Create a static or singleton HttpClient with PooledConnectionLifetime and PooledConnectionIdleTimeout and use it for any incoming requests**

> This is scenario 4.1, not scenario 5.   
> What's the point of having a scenario 4.1? This scenario is the same as scenario 4 but with a slight modification that I think it is worth mentioning.

## Source code
- A ``static`` ``HttpClient`` instance is created once and reused for incoming requests.
- The ``HttpClient`` is created using the ``PooledConnectionLifetime`` attribute. This attribute defines how long connections remain active when pooled. Once this lifetime expires, the connection will no longer be pooled or issued for future requests.  
- The ``PooledConnectionIdleTimeout`` attribute defines how long idle connections remain within the pool while unused. Once this lifetime expires, the idle TCP connection will be closed and removed from the pool.
> In the next code snippet, the ``PooledConnectionIdleTimeout`` is set to 10 seconds, which means that idle TCP connections will be closed after a maximum of 10 seconds. This is highly inefficient and only done for demo purposes.

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioFourController : ControllerBase
{

    private static readonly HttpClient Client = new(new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(30),
        PooledConnectionIdleTimeout = TimeSpan.FromSeconds(10)
    })
    {
        BaseAddress = new Uri("https://jsonplaceholder.typicode.com/"),
        DefaultRequestHeaders = { { "accept", "application/json" } },
        Timeout = TimeSpan.FromSeconds(15),
    };

    [HttpGet()]
    public async Task<ActionResult> Get()
    {

        var response = await Client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```

Why is the ``PooledConnectionIdleTimeout`` attribute something worth mentioning? Let's take a look at the code above.

- ``PooledConnectionLifetime`` is set to 30 minutes, which means that TCP connections will be re-used during 30 minutes.
- ``PooledConnectionIdleTimeout``is set to 10 seconds, which means that idle TCP connections will be closed after a maximum of 5 seconds, **it doesn't matter if the ``PooledConnectionLifetime`` has expired or not.** 

What will happen here?
- If the app keeps receiving a constant flow of requests, then the same TCP connection will be reused during **30 minutes**.
- If for some reason the app doesn't receive any requests and the TCP connection gets considered as idle, then the TCP connection will be closed after **10 seconds**, it doesn’t matter if the ``PooledConnectionLifetime`` time has expired or not.

Let's take a look at this behaviour using the ``netstat`` command:
{{< video src="/videos/httpclient-scenario4-1.mp4" type="video/mp4" preload="auto" >}}


# **Scenario 5: Use IHttpClientFactory**

## Source code

- An ``IHttpClientFactory`` named client is setup in the ``Program.cs`` _(this Scenario uses an ``IHttpClientFactory`` named client, you could use a typed client or a basic client and the behaviour will be exactly the same)_.
- The ``SetHandlerLifetime`` extension method defines the length of time that a ``HttpMessageHandler`` instance can be reused before being discarded. It works almost identical as the ``PooledConnectionLifetime`` attribute from the previous scenario.
- We use the ``CreateClient`` method from the ``IHttpClientFactory`` to obtain a ``httpClient`` to call our API.

> The ``SetHandlerLifetime`` method is set to 15 seconds, which means that TCP connections will cease to be re-issued and be closed after a maximum of 15 seconds. This is highly inefficient and it is only done for demo purposes.

On ``Program.cs``:
```csharp
builder.Services.AddHttpClient("typicode", c =>
{
    c.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
    c.Timeout = TimeSpan.FromSeconds(15);
    c.DefaultRequestHeaders.Add(
        "accept", "application/json");
})
.SetHandlerLifetime(TimeSpan.FromSeconds(15));
````

On ``ScenarioFiveController.cs``:

```csharp
[ApiController]
[Route("[controller]")]
public class ScenarioFiveController : ControllerBase
{
    private readonly IHttpClientFactory _factory;

    public ScenarioFiveController(IHttpClientFactory factory)
    {
        _factory = factory;
    }

    [HttpGet()]
    public async Task<ActionResult> Get()
    {
        var client = _factory.CreateClient("typicode");

        var response = await client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return StatusCode(500);
    }
}
```

## netstat output

- TCP connections are being reused.
- The ``SetHandlerLifetime`` method is set to 15 seconds, which means that after 15 seconds the TCP connection will be marked for expiration. The next incoming request will spawn a new TCP connection.
- In this scenario, unlike Scenario 4 where the TCP connection was closed immediately after the time set by the ``PooledConnectionLifetime`` attribute had expired, the expiration of a handler will not promptly dispose of the TCP connection. The expired handler will be positioned in a distinct pool, which is periodically processed to dispose of handlers only when they become unreachable.

{{< video src="/videos/httpclient-scenario5.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons
### Pros 
- TCP connections are being reused, which further reduces the likelihood of experiencing a port exhaustion issue. 
- It solves the DNS change issue mentioned on scenario 3.
- It simplifies the declaration and usage of ``HttpClient`` instances.

### Cons
- The ``IHttpClientFactory`` keeps everything nice and simple as long as you only want to modify the basic ``HttpClient`` parameters, it might be a bit harder if you need to tweak some of the less common parameters.     
The next code snippet is an example of how to set the ``PooledConnectionIdleTimeout`` attribute discussed on scenario 4.1, as you can see you'll need to use the ``ConfigurePrimaryHttpMessageHandler`` extension method and create a new ``SocketsHttpHandler`` instance, just to set the value of the ``PooledConnectionIdleTimeout`` attribute.

```csharp
builder.Services.AddHttpClient("typicode", c =>
    {
        c.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
        c.Timeout = TimeSpan.FromSeconds(15);
        c.DefaultRequestHeaders.Add(
            "accept", "application/json");
    })
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler()
    {
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(5)
    })
    .SetHandlerLifetime(TimeSpan.FromMinutes(20));
```
---
---

Up to this point, we have only explored scenarios that affected .NET 5/6/7, but what if we want to use an ``HttpClient`` in an application built with .NET Framework 4.8?    
 - The recommended way is using **IHttpClientFactory**.*

_*You can use Scenario 3 if you are certain that you will not encounter DNS changes in the service you are calling._

- Can we use Scenario 4?   

No, we cannot. Scenario 4 doesn't work with .NET Framework, the ``SocketsHttpHandler`` doesn't exist in .NET Framework.   
HttpClient is built on top of the pre-existing HttpWebRequest implementation, you could use the ``ServicePoint`` API to control and manage HTTP connections, including setting a connection lifetime by configuring the ``ConnectionLeaseTimeout`` for an endpoint.   


# **Scenario 6: Using IHttpClientFactory with .NET Framework and Autofac** 

## Source code

- This scenario uses Autofac as IoC container.
- An ``IHttpClientFactory`` named client is setup in the ``AutofacWebapiConfig.cs``.
- A few steps are required to setup ``IHttpClientFactory`` with Autofac:
    - Add required packages:
        - ``Microsoft.Extensions.Http``
    - Configure Autofac:
        - Create a new ``ServiceCollection`` instance.
        - Register the ``HttpClient``.
        - Build the ``ServiceProvider`` and resolve ``IHttpClientFactory``.
    - **The ``IHttpClientFactory`` must be setup as ``SingleInstance`` on Autofac**, or it won't work properly.

On ``Global.asax``:
```csharp
protected void Application_Start()
{
    AreaRegistration.RegisterAllAreas();

    AutofacWebapiConfig.Initialize(GlobalConfiguration.Configuration);

    GlobalConfiguration.Configure(WebApiConfig.Register);
    FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
    RouteConfig.RegisterRoutes(RouteTable.Routes);
    BundleConfig.RegisterBundles(BundleTable.Bundles);
}
```

``AutofacWebApiConfig`` class implementation, looks like this:
```csharp
public class AutofacWebapiConfig
{
    public static IContainer Container;

    public static void Initialize(HttpConfiguration config)
    {
        Initialize(config, RegisterServices(new ContainerBuilder()));
    }

    public static void Initialize(HttpConfiguration config, IContainer container)
    {
        config.DependencyResolver = new AutofacWebApiDependencyResolver(container);
    }

    private static IContainer RegisterServices(ContainerBuilder builder)
    {
        builder.RegisterApiControllers(Assembly.GetExecutingAssembly());
        
        builder.Register(ctx =>
        {
            var services = new ServiceCollection();
            services.AddHttpClient("typicode", c =>
            {
                c.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
                c.Timeout = TimeSpan.FromSeconds(15);
                c.DefaultRequestHeaders.Add(
                    "accept", "application/json");
            })
            .SetHandlerLifetime(TimeSpan.FromSeconds(15));

            var provider = services.BuildServiceProvider();
            return provider.GetRequiredService<IHttpClientFactory>();

        }).SingleInstance();

        Container = builder.Build();

        return Container;
    }

}
```

``ScenarioSixController.cs`` looks like this:

```csharp
 public class ScenarioSixController : ApiController
{
    private readonly IHttpClientFactory _factory;

    public ScenarioSixController(IHttpClientFactory factory)
    {
        _factory = factory;
    }
    
    public async Task<IHttpActionResult> Get()
    {
        var client = _factory.CreateClient("typicode");

        var response = await client.GetAsync(
            "posts/1/comments");

        if (response.IsSuccessStatusCode)
            return Ok(await response.Content.ReadAsStringAsync());

        return InternalServerError();
    }

}
```

## netstat output

- TCP connections are being reused.
- The ``SetHandlerLifetime`` method is set to 15 seconds, which means that after 15 seconds the TCP connection will be marked for expiration. The next request will create a new TCP connection.
- Expiry of a handler will not immediately dispose the TCP connection. An expired handler is placed in a separate pool which is processed at intervals to dispose handlers only when they become unreachable.

## Pros & cons
### Pros 
- TCP connections are being reused, which further reduces the likelihood of experiencing a port exhaustion issue. 
- It solves the DNS change issues mentioned on scenario 3.

### Cons
- To avoid creating a new TCP connection every time a new request comes in, it is crucial to register the ``IHttpClientFactory`` as a Singleton in Autofac.
