---
title: "Getting started with OpenTelemetry and distributed tracing in .NET Core"
date: 2021-04-08T10:11:51+02:00
tags: ["opentelemetry", "tracing", "jaeger", "dotnet", "csharp"]
description: "OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as traces, metrics, and logs. On today's post I'm going to show you how you can start using OTEL and distributed tracing with .NET Core."
Lastmod: 2021-01-06
draft: false
---

> **Just show me the code**   
> As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/opentelemetry-tracing-demo)

A few months ago the first stable version of the OpenTelemetry client for dotnet was released and since then I have wanted to write a little bit about it.   

OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as traces, metrics, and logs. Today I'm going to focus on tracing.    
From my standpoint, there are at least **four main concepts** about tracing and OpenTelemetry that you need to know. 

## **Span and activities** 
  
A span is the building block that forms a trace, it has a unique identifier and represents a piece of the workflow in the distributed system.   
Multiple spans are pieced together to create a trace. Traces are often viewed as a "tree" of spans that reflects the time that each span started and completed.

In dotnet a Span is represented by an **Activity**.  

The OpenTelemetry client for dotnet is reusing the existing Activity and associated classes to represent the OpenTelemetry Span.    
This means that users can instrument their applications/libraries to emit OpenTelemetry compatible traces by using just the .NET Runtime.

To create a Span in .NET we must first create a new activity:

```csharp
private static readonly ActivitySource Activity = new(nameof(RabbitRepository));
```
And then call _"StartActivity"_ to begin recording, everything that happens inside the using block will be recorded into that Span.

```csharp
using (var activity = Activity.StartActivity("Process Message", ActivityKind.Consumer, parentContext.ActivityContext)){}
```

## **Propagators** 

A propagator allows us to extract and inject context across process boundaries. 

This is typically required if you are not using any of the .NET communication libraries which has instrumentations already available which does the propagation (eg: HttpClient).    
In such cases, context extraction and propagation is the responsibility of the library itself.

To create a propagator in .NET we must first create a TextMapPropagator

```csharp
private static readonly TextMapPropagator Propagator = new TraceContextPropagator();
```
Then use the Inject and Extract methods for inter-process trace propagation.

The _"Inject"_ method injects the Activity into a carrier. For example, into the headers of an HTTP request.

```csharp
Propagator.Inject(new PropagationContext(activity.Context, Baggage.Current), props, InjectContextIntoHeader);
```

And the _"Extract"_ method extracts the value from an incoming request. For example, from the headers of an HTTP request.

If a value can not be parsed from the carrier, for a cross-cutting concern, the implementation should not throw an exception and should not store a new value in the Context, in order to preserve any previously existing valid value.

```csharp
 var parentContext = Propagator.Extract(default, ea.BasicProperties, ExtractTraceContextFromBasicProperties);
```

## **Exporters**

Let's be honest emiting traces is kind of pointless if you don't have a backend capable of aggregating the traces and displaying those traces in a friendly manner. 

An exporter is how data gets sent to those different back-ends. 

Generally, an exporter translates the internal format into another defined format.

These are the most well-known available trace exporters:
- Jaeger
- OTLP gRPC
- OTLP HTTP
- Zipkin

> In this post I'm going to use **Jaeger**.

## **Attributes**

Attributes are key:value pairs that provide additional information to a trace.

In .NET those are called Tags. We can add an attribute into an Activity like this:

```csharp
  activity?.SetTag("messaging.system", "rabbitmq");
  activity?.SetTag("messaging.destination_kind", "queue");
  activity?.SetTag("messaging.rabbitmq.queue", "sample");
```
***

I'm going to stop talking about OpenTelemetry theory because I could end up writing and entire post about it.   
And to be honest I would end up paraphrasing of the official docs, so instead of that I'm going to point you towards the official documentation: https://opentelemetry.io/docs/concepts/

After talking a little bit about OpenTelemetry let me refocus on the **real** purpose of this post.   

### In this post I want to show you a practical example about how you can start using OpenTelemetry when you have a series of .NET services that communicate amongst them.   


# Example

I have built 4 apps beforehand, these apps are not doing any business logic operations because that's not the goal here.    
The important part is that all the apps are talking among them and also using other external services like Redis, MSSQL or Rabbit.   

Here's the diagram:

![otel-diagram](/img/otel-components-diagram.png)


- **App1.WebApi** is a **NET6 Web API** with 2 endpoints.
    - The **/http** endpoint makes an HTTP request to the App2 _"/dummy"_ endpoint.
    - The **/publish-message** endpoint queues a message into a Rabbit queue named _"sample"_.
    
- **App2.RabbitConsumer.Console** is a **NET6 console** application. 
  - Dequeues messages from the Rabbit _"sample"_ queue and makes a HTTP request to the **App3** _"/sql-to-event"_ endpoint with the content of the message.

- **App3.WebApi** is a **NET6 Web API** with 2 endpoints
    - The **/dummy** endpoint returns a fixed _"Ok"_ response.
    - The **/sql-to-event** endpoint receives a message via HTTP POST, stores it in a MSSQL Server and afterwards publishes the message as an event into a RabbitMq queue named _"sample_2"_.

- **App4.RabbitConsumer.HostedService** is a **NET6 Worker Service**.
  - A Hosted Service reads the messages from the Rabbitmq _"sample_2"_ queue and stores it into a Redis cache database.

Those apps are not using OpenTelemetry right now, so in the next sections we're going to do a step-by-step guide about how to setup and use the OpenTelemetry client.

# OpenTelemetry .NET Client

To get started with OpenTelemetry we're going to need the following packages.

```xml
<PackageReference Include="OpenTelemetry" Version="1.2.0-rc1" />
<PackageReference Include="OpenTelemetry.Exporter.Jaeger" Version="1.2.0-rc1" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.0.0-rc8" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.0.0-rc8" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.0.0-rc8" />
<PackageReference Include="OpenTelemetry.Instrumentation.SqlClient" Version="1.0.0-rc8" />
<PackageReference Include="OpenTelemetry.Instrumentation.StackExchangeRedis" Version="1.0.0-rc8" />
```

- The _**OpenTelemetry**_ package is the core library.
- The _**OpenTelemetry.Exporter.Jaeger**_ package allows us to export the traces to Jaeger.
- The _**OpenTelemetry.Extensions.Hosting**_ package contains some extensions that allows us to add the dependencies into the DI container and configure the HostBuilder.
- The _**OpenTelemetry.Instrumentation.***_ packages are instrumentation libraries. These packages are instrumenting common libraries/functionalities/classes so we don't have to do all the heavy lifting by ourselves. In our example we're using the following ones:
  - The _**OpenTelemetry.Instrumentation.AspNetCore**_ instruments _ASP.NET Core_ and collects telemetry about incoming web requests. This instrumentation also collects incoming gRPC requests using _Grpc.AspNetCore_.
  - The _**OpenTelemetry.Instrumentation.Http**_ instruments the _System.Net.Http.HttpClient_ and _System.Net.HttpWebRequest_ types and collects telemetry about outgoing HTTP requests. 
  - The _**OpenTelemetry.Instrumentation.SqlClient**_ instruments the _Microsoft.Data.SqlClient_ and _System.Data.SqlClient_ types and collects telemetry about database operations.
  - The _**OpenTelemetry.Instrumentation.StackExchangeRedis**_ instruments the _StackExchange.Redis_ type and collects telemetry about outgoing calls to Redis.

In the near future I expect to see more and more instrumentation libraries like these ones so we can instrument the most common dependencies with no effort at all, just install a nuget, add some config lines in the _Startup_ and you're good to go.

# Adding OpenTelemetry on App1

**1 . Setup the OpenTelemetry library**

```csharp
services.AddOpenTelemetryTracing(builder =>
{
    builder.AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource(nameof(PublishMessageController))
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App1"))
        .AddJaegerExporter(opts =>
        {
            opts.AgentHost = Configuration["Jaeger:AgentHost"];
            opts.AgentPort = Convert.ToInt32(Configuration["Jaeger:AgentPort"]);
            opts.ExportProcessorType = ExportProcessorType.Simple;
        });
});
```
This is the first app we're instrumenting so I'm going to explain a little bit what we're doing line by line

```csharp
AddAspNetCoreInstrumentation()
```
Enables NET Core instrumentation.

```csharp
AddHttpClientInstrumentation()
```
Enable HTTP Instrumentation.    
The Api 1 makes an HTTP request to the App2, if we want to trace the HTTP call between these 2 apps we can do it simply by adding this extension method.

```csharp
.AddSource(nameof(PublishMessageController))
```
The `AddSource` method can be used to add a ActivitySource to the provider. Multiple AddSource can be called to add more than one span.

Why we need this? The Api 1 queues a message into a rabbit queue and we want to record this trace.   
In the former paragraph we have seen that  HTTP requests has built-in instrumentation via method extension, that is not the case for the RabbitMQ dependency. In order to continue the distributed transaction, we must create a new Activity.


```csharp
SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App1"))
```
A Resource is the immutable representation of the entity producing the telemetry.   
With the `SetResourceBuilder` method we're configuring the Resource for the application.

```csharp
AddJaegerExporter(opts =>
{
    opts.AgentHost = Configuration["Jaeger:AgentHost"];
    opts.AgentPort = Convert.ToInt32(Configuration["Jaeger:AgentPort"]);
});
```
This method sets Jaeger as the exporter where all the traces are going to be sent.

**2 . Instrument dependency calls**

Unlike the HTTP request, OpenTelemetry does not yet have support for automatic RabbitMq trace correlation.

The code below demonstrates how the _"publish message"_ trace can be created. The code snippet adds the trace information to the enqueued message header, which will later be used to link both operations.

Specifically we are using a Propagator to inject the activity into the message header that is going to be queued to Rabbit, afterwards the consumer application will use another Propagator to extract the Activity and link the producer activity with the consumer activity.

Also we are also using the Tags attribute to store relevant metadata into the Activity.

```csharp
[ApiController]
[Route("publish-message")]
public class PublishMessageController : ControllerBase
{
    private static readonly ActivitySource Activity = new(nameof(PublishMessageController));
    private static readonly TextMapPropagator Propagator = Propagators.DefaultTextMapPropagator;

    private readonly ILogger<PublishMessageController> _logger;
    private readonly IConfiguration _configuration;

    public PublishMessageController(
        ILogger<PublishMessageController> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    [HttpGet]
    public void Get()
    {
        try
        {
            using (var activity = Activity.StartActivity("RabbitMq Publish", ActivityKind.Producer))
            {
                var factory = new ConnectionFactory { HostName = _configuration["RabbitMq:Host"] };
                using (var connection = factory.CreateConnection())
                using (var channel = connection.CreateModel())
                {
                    var props = channel.CreateBasicProperties();

                    AddActivityToHeader(activity, props);

                    channel.QueueDeclare(queue: "sample",
                        durable: false,
                        exclusive: false,
                        autoDelete: false,
                        arguments: null);

                    var body = Encoding.UTF8.GetBytes("I am app1");

                    channel.BasicPublish(exchange: "",
                        routingKey: "sample",
                        basicProperties: props,
                        body: body);
                }
            }
        }
        catch (Exception e)
        {
            _logger.LogError("Error trying to publish a message", e);
            throw;
        }
    }

    private void AddActivityToHeader(Activity activity, IBasicProperties props)
    {
        Propagator.Inject(new PropagationContext(activity.Context, Baggage.Current), props, InjectContextIntoHeader);
        activity?.SetTag("messaging.system", "rabbitmq");
        activity?.SetTag("messaging.destination_kind", "queue");
        activity?.SetTag("messaging.rabbitmq.queue", "sample");
    }

    private void InjectContextIntoHeader(IBasicProperties props, string key, string value)
    {
        try
        {
            props.Headers ??= new Dictionary<string, object>();
            props.Headers[key] = value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to inject trace context.");
        }
    }
}

```

# Adding OpenTelemetry on App2

**1 . Setup the OpenTelemetry library**

The App2 is a console app so we have to setup the library in a different way,  instead of using an IServiceCollection extension we're creating directly a TraceProvider. Apart from that the setup looks pretty much the same.

```csharp
 Sdk.CreateTracerProviderBuilder()
    .AddHttpClientInstrumentation()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App2"))
    .AddSource(nameof(Program))
    .AddJaegerExporter(opts =>
    {
        opts.AgentHost = _configuration["Jaeger:AgentHost"];
        opts.AgentPort = Convert.ToInt32(_configuration["Jaeger:AgentPort"]);
    })
    .Build();
```


**2 . Instrument dependency calls**

The HTTP call is being instrumented automatically thanks to the `AddHttpClientInstrumentation` extension method, but this app also dequeues a message from Rabbit, so we want to instrument that part.   
The code is pretty much the same as the one we have created for the Rabbit producer on App1, the only difference is that instead of injecting the activity we are to extracting it from the message header.

```csharp
private static async Task ProcessMessage(BasicDeliverEventArgs ea,
            HttpClient httpClient,
            IModel rabbitMqChannel)
{
    try
    {
      //Extract the activity and set it into the current one
      var parentContext = Propagator.Extract(default, ea.BasicProperties, ExtractTraceContextFromBasicProperties);
      Baggage.Current = parentContext.Baggage;

      //Start a new Activity
      using (var activity = Activity.StartActivity("Process Message", ActivityKind.Consumer, parentContext.ActivityContext))
      {

          var body = ea.Body.ToArray();
          var message = Encoding.UTF8.GetString(body);

          //Add Tags to the Activity
          AddActivityTags(activity);

          _logger.LogInformation("Message Received: " + message);

          _ = await httpClient.PostAsync("/sql-to-event",
              new StringContent(JsonSerializer.Serialize(message),
                  Encoding.UTF8,
                  "application/json"));

          rabbitMqChannel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
      }

    }
    catch (Exception ex)
    {
        _logger.LogError($"There was an error processing the message: {ex} ");
    }
}

//Extract the Activity from the message header
private static IEnumerable<string> ExtractTraceContextFromBasicProperties(IBasicProperties props, string key)
{
    try
    {
        if (props.Headers.TryGetValue(key, out var value))
        {
            var bytes = value as byte[];
            return new[] { Encoding.UTF8.GetString(bytes) };
        }
    }
    catch (Exception ex)
    {
        _logger.LogError($"Failed to extract trace context: {ex}");
    }

    return Enumerable.Empty<string>();
}

//Add Tags to the Activity
private static void AddActivityTags(Activity activity)
{
    activity?.SetTag("messaging.system", "rabbitmq");
    activity?.SetTag("messaging.destination_kind", "queue");
    activity?.SetTag("messaging.rabbitmq.queue", "sample");
}
```


# Adding OpenTelemetry on App3

**1 . Setup the OpenTelemetry library**

```csharp
services.AddOpenTelemetryTracing(builder =>
{
    builder.AddAspNetCoreInstrumentation()
        .AddSource(nameof(RabbitRepository))
        .AddSqlClientInstrumentation()
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App3"))
        .AddJaegerExporter(opts =>
        {
            opts.AgentHost = Configuration["Jaeger:AgentHost"];
            opts.AgentPort = Convert.ToInt32(Configuration["Jaeger:AgentPort"]);
        });
});
```

The setup looks almost identical as the ones on the App1 and App2. There is only one thing worth mentioning here:

```csharp
AddSqlClientInstrumentation()
```
This app is not making any HTTP request, instead of that is querying a database, so we're using the SQL extension method to enable SQL instrumentation. We're simply swapping the `OpenTelemetry.Instrumentation.Http` library for the `OpenTelemetry.Instrumentation.SqlClient` library.

This app is also queuing a message into a RabbitMq queue, but the code it's exactly the same as the one I have showed for the app1 section.


## Adding OpenTelemetry on App4

**1 . Setup the OpenTelemetry library**

```csharp
services.AddOpenTelemetryTracing(builder =>
{
    var provider = services.BuildServiceProvider();
    IConfiguration config = provider
            .GetRequiredService<IConfiguration>();

    builder.AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .Configure((sp, builder) =>
          {
              RedisCache cache = (RedisCache)sp.GetRequiredService<IDistributedCache>();
              builder.AddRedisInstrumentation(cache.GetConnection());
          })
        .AddSource(nameof(Worker))
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App4"))
        .AddJaegerExporter(opts =>
        {                    
            
            opts.AgentHost = config["Jaeger:AgentHost"];
            opts.AgentPort = Convert.ToInt32(config["Jaeger:AgentPort"]);
            opts.ExportProcessorType = ExportProcessorType.Simple;
        });
});
```
This app dequeues a message from a RabbitMq queue and stores the message content into a Redis cache database.

The instrumentation part for dequeuing the message from RabbitMq looks exactly the same as the snippet I have put on the App2 section, so I'm going to skip that part.

The instrumentation for Redis is a little more problematic.    
The issue when trying to instrument Redis is that the `AddRedisInstrumentation()` extension method needs an instance of the Redis `ConnectionMultiplexer`.   
If you're using an `IDistributedCache` interface and the `AddStackExchangeRedisCache` extension method to configure the Redis connection, like this one: 

```csharp
services.AddStackExchangeRedisCache(options =>
{
    var connString =
        $"{hostContext.Configuration["Redis:Host"]}:{hostContext.Configuration["Redis:Port"]}";
    options.Configuration = connString;
});
```

You will realize that you can't access the `ConnectionMultiplexer` property because is not publicly accesible, so I'm writing an extension method that is using Reflection to obtain it.

```csharp
public static class RedisCacheExtensions
{
    public static ConnectionMultiplexer GetConnection(this RedisCache cache)
    {
        //ensure connection is established
        typeof(RedisCache).InvokeMember("Connect", BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.InvokeMethod, null, cache, new object[] { });

        //get connection multiplexer
        var fi = typeof(RedisCache).GetField("_connection", BindingFlags.Instance | BindingFlags.NonPublic);
        var connection = (ConnectionMultiplexer)fi.GetValue(cache);
        return connection;
    }
}
```
Also, as you can see I'm not using directly the ``AddRedisInstrumentation()`` method, instead of that I'm wrapping the ``AddRedisInstrumentation()`` methood with the ``Configure()`` method.    
The ``Configure()`` method is an overload method that allows to get a hold of the ``IServiceProvider`` instance.

# Jaeger  

After adding the OpenTelemetry library on these 4 apps we are going to generate some traffic and afterwards access Jaeger to start analyzing the traces that the apps are sending.

If we take a look at the entire trace, this is how it looks:

![otel-jaeger](/img/otel-jaeger.png)


Let me break it up a little bit for you. That's a zoomed image of the spans we have created in our trace

![otel-jaeger-span](/img/otel-jaeger-span.png)

- The first span (publish-message) corresponds to the App1 api endpoint, that span is created automatically when the api controller is executed thanks to the `AddAspNetCoreInstrumentation()` extension method.
  
- The second span (RabbitMq publish) is an Activity that we have created manually on the App1. You can see the code snippet on the "Adding OpenTelemetry on App1" section.
  
- The third span (Process Message) is an Activity that we have created manually on the App2, also you can see the code snippet on the "Adding OpenTelemetry on App2" section.
  
- The fourth span (HTTP POST) corresponds to the HTTP request that the App2 is making to the App3. That span is created automatically thanks to the `AddHttpClientInstrumentation()` extension method found on the App2 setup.
  
- The fifth span (sql-to-event) corresponds to the App3 sql-to-event api endpoint, that span is created automatically when the api controller is executed thanks to the `AddAspNetCoreInstrumentation()` extension method.
  
- The sixth span (sqlserver) corresponds to the SQL query that the App3 is doing, that span is created automatically thanks to the `AddSqlClientInstrumentation()` extension method.
  
- The seventh span (RabbitMq publish) is an Activity that we have created manually on the App3.
  
- The eigth span (Process Message) is an Activity that we have created manually on the App4.
  
- The last 2 spans correspond to the Redis instructions that we are doing on the App4, both spans are created automatically thanks to the `AddRedisInstrumentation()` extension method.

If we want more info on any span we can drill-down and inspect the metadata. For example if we inspect the "HTTP POST" span we can see more info about it.

![otel-jaeger-http-instrumentation](/img/otel-jaeger-http-instrumentation.png)


Also, do you remember that in the App1 we added some extra Tags on the Activity?

```csharp
activity?.SetTag("messaging.system", "rabbitmq");
activity?.SetTag("messaging.destination_kind", "queue");
activity?.SetTag("messaging.rabbitmq.queue", "sample");
```

Here is the result: 

![otel-jaeger-rabbit-tags](/img/otel-jaeger-rabbit-tags.png)

Jaeger is capable of building a dependency-graph after looking at the traces.

![otel-jaeger-dag](/img/otel-jaeger-dag.png)

Also is capable of giving performance statistics.

![otel-jaeger-statistics](/img/otel-jaeger-statistics.png)

And compare traces.

![otel-jaeger-compare-traces](/img/otel-jaeger-compare-traces.png)


# How to test the example apps

If you want to take a look at the 4 apps, I have uploaded everything on my [GitHub repository](https://github.com/karlospn/opentelemetry-tracing-demo)   

If you want to try for yourselves to execute this example, I have uploaded also a **docker-compose** file with everything you need, so you can run a compose up and you're good to go.   

But there is little caveat I wanted to mention in the docker-compose file.      

With docker-compose you can control the order of the services startup and shutdown with the depends_on option, but it does not wait until a container is “ready”, it only waits until is running.   

In my case that's a problem because App2 and App4 need to wait for the rabbitMq container to be ready, to avoid this problem the compose file is overwriting the "entrypoint" for both apps and executing a shell script that makes both apps sleep 30 seconds before starting up.

Here's the docker-compose file:


```yaml
version: '3.4'

networks:
  tracing:
    name: tracing-network
    
services:
  rabbitmq:
    image: rabbitmq:3.6.15-management
    ports:
      - 15672:15672
      - 5672
    networks:
      - tracing

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Pass@Word1
    ports:
      - 1433
    networks:
      - tracing

  redis:
    image: redis:6.2.1
    ports:
    - 6379:6379
    networks:
      - tracing
    
  jaeger:
    image: jaegertracing/all-in-one
    container_name: jaeger
    restart: unless-stopped
    ports:
      - 5775:5775/udp
      - 5778:5778
      - 6831:6831/udp
      - 6832:6832/udp
      - 9411:9411
      - 14268:14268
      - 16686:16686
    networks:
      - tracing

  app1:
    build:
      context: ./App1.WebApi
    ports:
      - "5000:80"
    networks:
      - tracing
    depends_on: 
      - rabbitmq
      - jaeger
      - app3
    environment:
      Jaeger__AgentHost: jaeger
      Jaeger__AgentPort: 6831
      RabbitMq__Host: rabbitmq
      App3Endpoint: http://app3/dummy

  app2:
    stdin_open: true
    tty: true
    build:
      context: ./App2.RabbitConsumer.Console
    networks:
      - tracing
    depends_on: 
      - rabbitmq
      - jaeger
      - app3
    entrypoint: ["./wait.sh", "30", "dotnet", "App2.RabbitConsumer.Console.dll"]
    environment:
      Jaeger__AgentHost: jaeger
      Jaeger__AgentPort: 6831
      RabbitMq__Host: rabbitmq
      App3UriEndpoint: http://app3

  app3:
    build:
      context: ./App3.WebApi
    ports:
      - "5001:80"
    networks:
      - tracing
    depends_on: 
      - rabbitmq
      - jaeger
      - sqlserver
    environment:
      Jaeger__AgentHost: jaeger
      Jaeger__AgentPort: 6831
      RabbitMq__Host: rabbitmq
      SqlDbConnString: server=sqlserver;user id=sa;password=Pass@Word1;

  app4:
    build:
      context: ./App4.RabbitConsumer.HostedService
    networks:
      - tracing
    depends_on: 
      - rabbitmq
      - jaeger
      - redis
    entrypoint: ["./wait.sh", "30", "dotnet", "App4.RabbitConsumer.HostedService.dll"]
    environment:
      Jaeger__AgentHost: jaeger
      Jaeger__AgentPort: 6831
      RabbitMq__Host: rabbitmq
      Redis__Host: redis
      Redis__Port: 6379
```

