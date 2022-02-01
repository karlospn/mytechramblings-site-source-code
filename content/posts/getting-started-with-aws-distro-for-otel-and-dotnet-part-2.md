---
title: "Getting started with AWS Distro for OpenTelemetry and distributed tracing using .NET. Part 2: Building the demo"
date: 2022-02-01T10:00:12+01:00
tags: ["AWS", "dotnet", "OpenTelemetry"]
description: "This is a 2 part-series post. In part 2 I’ll show you how to build and configure properly a few distributed .NET apps that will send traces to the AWS OTEL Collector, and also how to analyze those traces in AWS X-Ray."
draft: false
---

> This is a 2 part-series post.
> - **Part 1**: What is the AWS OpenTelemetry Collector and how to setup it up for ingesting and exporting tracing data to AWS X-Ray. If you want to read it, click [here](https://www.mytechramblings.com/posts/getting-started-with-aws-distro-for-otel-and-dotnet-part-1/)
> - **Part 2**: How to build and configure properly a few distributed .NET apps that will send traces to the AWS OTEL Collector, and also how to analyze those traces in AWS X-Ray.

**Just show me the code**   
As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/aws-otel-tracing-demo).

# Building the demo

In the previous post we talked about what is the AWS OTEL Collector and how to set it up, now it's time to focus on building some apps.

The demo we're going to build will contain 4 apps, these apps are not doing any business logic operations because that's not the goal here.    
The key part of this demo is that the apps are communicating between them and also with some AWS services like S3, DynamoDb, AmazonMQ or SQS.

Here's the diagram:

![components-diagram](/img/aws-otel-demo-components-diagram.png)

- **App1.WebApi** is a .NET6 Web API with 2 endpoints.
    - The ``/http`` endpoint  makes an HTTP request to the **App2** ``/dummy`` endpoint.
    - The ``/publish-message``  endpoint queues a message into an **AWS ActiveMQ RabbitMQ queue**.

- **App2.RabbitConsumer.Console** is a .NET6 console app. 
  - Dequeues messages from the RabbitMQ queue and makes an HTTP request to the **App3** ``/s3-to-event`` endpoint with the content of the message.

- **App3.WebApi** is a .NET6 Web API with 2 endpoints.
    - The ``/dummy`` endpoint returns a fixed "Ok" response.
    - The ``/s3-to-event`` endpoint receives a message via HTTP POST, stores it in an **S3 bucket** and afterwards publishes the message as an event into an **AWS SQS queue**.

- **App4.SqsConsumer.HostedService** is a .NET6 Worker Service.
  - It reads the messages from the **AWS SQS queue** and stores it into a **DynamoDb table**.

# Required .NET packages

To get started we’re going to need the following packages:

```xml
    <PackageReference Include="OpenTelemetry" Version="1.2.0-rc1" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.2.0-rc1" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.0.0-rc8" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.0.0-rc8" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.0.0-rc8" />
    <PackageReference Include="OpenTelemetry.Contrib.Extensions.AWSXRay" Version="1.1.0" />
    <PackageReference Include="OpenTelemetry.Contrib.Instrumentation.AWS" Version="1.0.1" />
    <PackageReference Include="AWSSDK.Extensions.NETCore.Setup" Version="3.7.1" />
    <PackageReference Include="AWSSDK.S3" Version="3.7.7.14" />
    <PackageReference Include="AWSSDK.SQS" Version="3.7.2.14" />
    <PackageReference Include="AWSSDK.DynamoDBv2" Version="3.7.2.12" />
```

- The ``OpenTelemetry package`` is the core OTEL library.
- The ``OpenTelemetry.Exporter.OpenTelemetryProtocol`` package allows us to export the traces to a gRPC or HTTP compatible endpoint. We will use this package to send the traces to the AWS OTEL Collector.
- The ``OpenTelemetry.Extensions.Hosting`` package contains some extensions for the OpenTelemetry client for .NET to integrate with .NET Core configuration and dependency injection frameworks.
- The ``OpenTelemetry.Instrumentation.*`` packages are instrumentation libraries. These packages are instrumenting common libraries/functionalities/classes so we don’t have to do all the heavy lifting by ourselves. In our demo we’re using the following ones:
  - The ``OpenTelemetry.Instrumentation.AspNetCore`` instruments ASP.NET Core and collects telemetry about incoming web requests. This instrumentation also collects incoming gRPC requests using Grpc.AspNetCore.
  - The ``OpenTelemetry.Instrumentation.Http`` instruments the ``System.Net.Http.HttpClient`` and ``System.Net.HttpWebRequest`` types and collects telemetry about outgoing HTTP requests.
- The ``OpenTelemetry.Contrib.Extensions.AWSXRay`` package is used to convert the OpenTelemetry traces into a compatible X-Ray trace format.    
- The ``OpenTelemetry.Contrib.Instrumentation.AWS`` packages is used to auto-instrument the AWS SDK clients and collects tracing data to downstream calls to AWS services.
- The ``AWSSDK.Extensions.NETCore.Setup`` package contains some extensions for the AWS SDK for .NET to integrate with .NET Core configuration and dependency injection frameworks.
- The ``AWSSDK.*`` packages are the official AWS SDK for .NET and allows us to interact with the different AWS Services. 

# Add and configure OpenTelemetry on App1

### **1. Setup the OpenTelemetry library**

```csharp
services.AddOpenTelemetryTracing(builder =>
{
  builder.AddAspNetCoreInstrumentation()
      .AddXRayTraceId()
      .AddHttpClientInstrumentation()
      .AddSource(nameof(PublishMessageController))
      .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App1"))
      .AddOtlpExporter(opts =>
      {
          opts.Endpoint = new Uri(Configuration["Otlp:Endpoint"]);
      });
});

Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());
```
As you can see it's a pretty straightforward configuration.

- The  ``AddXRayTraceId()`` method converts the OTEL trace ID into an AWS X-Ray compliant trace ID, without this method X-Ray is incapable of interpreting the application traces.
- The ``AddHttpClientInstrumentation()`` method enables HTTP auto-instrumentation. App1 makes an HTTP request to App2, if we want to trace the HTTP call between these 2 apps we can do it simply by adding this extension method.
- The ``AddSource(nameof(PublishMessageController))`` method is used to add a new ActivitySource to the provider. Everytime you create a new Activity you must add a new ActivitySource.
- The ``.SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App1"))`` method configures the Resource for the application.
- The ``AddOtlpExporter()`` method indicates that the tracing data is going to be exported to this specific endpoint. This endpoint matches with the gRPC Receiver Endpoint in the Collector configuration file.
- The ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator())`` method is needed if your app calls another application instrumented with AWS X-Ray.   
This method overrides the value of the ``Propagators.DefaultTextMapPropagator`` with the ``AWSXRayPropagator`` value. The ``Propagators.DefaultTextMapPropagator`` is used internally by some instrumentation packages to infer and propagate the trace ID.


To put it simply, if you're using AWS Distro for OpenTelemetry and you want to send all the tracing data to X-Ray you'll need to add this 2 method everytime you setup the OpenTelemetry provider in one of your apps:
- ``AddXRayTraceId()``
- ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());``

All the other method shown here are just basic OpenTelemetry setup, nothing to do with AWS Distro for OpenTelemetry.

### **2. Instrument the call to AmazonMQ RabbitMQ**

The ``OpenTelemetry.Contrib.Instrumentation.AWS`` NuGet package is used to auto-instrument the AWS SDK .NET clients, but there is no AWS SDK client to use with AmazonMQ RabbitMQ.   
That means that we need to used the ``RabbitMQ.Client`` package and that we need to instrument the downstream calls by ourselves.

In the next code snippet you'll see that we are injecting the trace information into the header of the message that is going to be sent.   
Later in App2 we'll extract the trace information from the message header and link both operations.

To be more specific, we are using the ``XRayPropagator`` to inject the Activity into the message header, this propagator will add a header named ``X-Amzn-Trace-Id`` into the message.   

In the previous section when setting up the OpenTelemetry client for this app we added the following command: ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator())``.   
This method overrides the value of the ``Propagators.DefaultTextMapPropagator`` with ``AWSXRayPropagator``.   
So now, we can get an instance of the ``AWSXRayPropagator`` propagator like this: ``var propagator = Propagators.DefaultTextMapPropagator``


```csharp
[ApiController]
[Route("publish-message")]
public class PublishMessageController : ControllerBase
{
  private static readonly ActivitySource Activity = new(nameof(PublishMessageController));

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
              var factory = new ConnectionFactory
              {
                  HostName = _configuration["RabbitMq:Host"],
                  UserName = _configuration["RabbitMq:Username"],
                  Password = _configuration["RabbitMq:Password"],
                  Ssl = new SslOption
                  {
                      Enabled = true,
                      Version = SslProtocols.None,
                      CertificateValidationCallback = (_, _, _, _) => true
                  }
              };

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

                  _logger.LogInformation("Publishing message to queue");

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
      var propagator = Propagators.DefaultTextMapPropagator;
      propagator.Inject(new PropagationContext(activity.Context, Baggage.Current), props, InjectContextIntoHeader);
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
# Add and configure OpenTelemetry on App2

### **1. Setup the OpenTelemetry library**

The App2 is a console app so we have to setup the OpenTelemetry library in a different way.   
Instead of using an ``IServiceCollection`` extension method we’re creating a ``TraceProvider``, apart from that, the setup looks almost the same as the previous one.

```csharp
var provider = Sdk.CreateTracerProviderBuilder()
    .AddXRayTraceId()
    .AddHttpClientInstrumentation()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App2"))
    .AddSource(nameof(Program))
    .AddOtlpExporter(opts =>
    {
        opts.Endpoint = new Uri(_configuration["Otlp:Endpoint"]);
    })
    .Build();

Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());

return provider;
```
### **2. Instrument the call to AmazonMQ RabbitMQ**

This app retrieves a message from an ActiveMQ RabbitMQ queue and makes an Http Request to App3.

The HTTP call is being instrumented automatically by the ``AddHttpClientInstrumentation()`` extension method, but the Rabbit part needs to be instrumented manually.

The code is pretty much the same as the one I have showed you on App1, but there are 2 differences.   
- The first one is pretty obvious, this time instead of injecting the trace into the message header we are using the ``XRayPropagator`` to extract the ``X-Amzn-Trace-Id`` header from the message and link both activies.

- The second difference is more subtle, but be careful with this one or the traces sent to X-Ray are going to be malformed.

> Only **Activities of kind Server are converted into X-Ray segments**. All other kind of spans (Internal, Client, etc.) are converted into X-Ray subsegments.

This means that if we don't set the Activity to be of  ``Kind = ActivityKind.Server`` then App2 is not going to show up correctly on X-Ray because it will be a subsegment.

In a Console app or a Worker Service is important to instantiate the main activity like this:    
``var activity = Activity.StartActivity("Process Message", ActivityKind.Server, parentContext.ActivityContext))``

So, why we need to set the ``Kind = ActivityKind.Server`` in this app and not on App1?   

That's because when we setup OpenTelemetry on a .NET WebAPI an Activity of ``Kind = ActivityKind.Server`` is created automatically everytime we invoke an API Controller, this is thanks to the ``AddAspNetCoreInstrumentation()`` extension method.

For example, if we take a quick look at how a trace on App1 looks like, you'll see this:

![x-ray-fulltrace-app1.png](/img/x-ray-fulltrace-app1.png)

In this trace:
- The parent span is created automatically when the API Controller is invoked.
- The second span ("Publish message" span) is the one we created manually. 

Remember this issue, because we're going to face it again on App4.

```csharp
 private static async Task ProcessMessage(BasicDeliverEventArgs ea,
            HttpClient httpClient,
            IModel rabbitMqChannel)
{
  try
  {
      var propagator = Propagators.DefaultTextMapPropagator;
      var parentContext = propagator.Extract(default, ea.BasicProperties, ExtractTraceContextFromBasicProperties);
      Baggage.Current = parentContext.Baggage;

      using (var activity = Activity.StartActivity("Process Message", ActivityKind.Server, parentContext.ActivityContext))
      {

          var body = ea.Body.ToArray();
          var message = Encoding.UTF8.GetString(body);
          AddActivityTags(activity);

          _logger.LogInformation("Message Received: " + message);

          _ = await httpClient.PostAsync("/s3-to-event",
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

private static void AddActivityTags(Activity activity)
{
  activity?.SetTag("messaging.system", "rabbitmq");
  activity?.SetTag("messaging.destination_kind", "queue");
  activity?.SetTag("messaging.rabbitmq.queue", "sample");
}
```

# Add and configure OpenTelemetry on App3

### **1. Setup the OpenTelemetry library**

App3 is a Web API that makes some calls to AWS S3 and AWS SQS.   
The setup of the OTEL client looks almost identical as the other ones, but with one difference:
 - The ``AddAWSInstrumentation()`` method from the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package is used to auto-instrument the downstream calls made by the AWS.SDK clients.

```csharp
services.AddOpenTelemetryTracing(builder =>
{
    builder.AddAspNetCoreInstrumentation()
        .AddXRayTraceId()
        .AddAWSInstrumentation()
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App3"))
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri(Configuration["Otlp:Endpoint"]);
        });

    Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());
});
```
### **2. Instrument the calls to AWS S3 and AWS SQS**

App3 receives a message from App2 and uploads the contents of the message into an S3 bucket.

In the next code snippet you'll see that there is nothing related to OpenTelemetry, we're simply using the ``AWSSDK.S3`` client to upload the contents of the message into the S3 bucket.

That's because the  ``AddAWSInstrumentation()`` extension method auto-instruments all the downstream calls made by the ``AWSSDK.S3`` client.    

The auto-instrumentation will create an activity of ``Kind=Activity.Client`` for every downstream call made to S3, and it will also add some tags to the activity, those tags will contain metadata about the AWS services being invoked.


```csharp
 public class S3Repository : IS3Repository
{
    private readonly IAmazonS3 _s3Client;
    private readonly ILogger<S3Repository> _logger;
    private readonly IConfiguration _configuration;

    public S3Repository(IAmazonS3 s3Client, 
        ILogger<S3Repository> logger, 
        IConfiguration configuration)
    {
        _s3Client = s3Client;
        _logger = logger;
        _configuration = configuration;
    }

    public async Task Persist(string message)
    {
        try
        {
            using var ms = new MemoryStream(Encoding.UTF8.GetBytes(message));
            var uploadRequest = new TransferUtilityUploadRequest
            {
                InputStream = ms,
                Key = $"file-{DateTime.UtcNow}",
                BucketName = _configuration["S3:BucketName"]
            };

            var fileTransferUtility = new TransferUtility(_s3Client);
            await fileTransferUtility.UploadAsync(uploadRequest);
        } 
        catch (Exception ex)
        {
            _logger.LogError("Error trying to upload file to S3 bucket", ex);
            throw;          
        }
    }
}
```
App3 also queues the message received into an Amazon SQS.

In the next code snippet there is also nothing related to OpenTelemetry because the  ``AddAWSInstrumentation()`` extension method auto-instruments all the downstream calls made by the ``AWSSDK.SQS`` client.  

There is one thing worth mentioning when working with Amazon SQS and the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package, and that is that the ``AWSTraceHeader`` attribute is going to be added into any message queued into SQS.     
The ``AWSTraceHeader`` is a message system attribute reserved by Amazon SQS to carry the X-Ray trace header.   

This header is important (more about it in the next section).


```csharp
 public class SqsRepository: ISqsRepository
{

    private readonly IAmazonSQS _client;
    private readonly IConfiguration _configuration;

    public SqsRepository(IAmazonSQS client, 
        IConfiguration configuration)
    {
        _client = client;
        _configuration = configuration;
    }

    public async Task Publish(IEvent evt)
    {
        await _client.SendMessageAsync(new SendMessageRequest
            {
                MessageBody = (evt as MessagePersistedEvent)?.Message,
                QueueUrl = _configuration["Sqs:Uri"],
            });      
    }
}
```

# Add and configure OpenTelemetry on App4

### **1. Setup the OpenTelemetry library**

App4 is a Worker Service that uses Amazon SQS and Amazon DynamoDb.

Nothing to comment here, the setup of the OTEL client is exactly the same as the one on App3.

```csharp
services.AddOpenTelemetryTracing(builder =>
{
    builder.AddAspNetCoreInstrumentation()
        .AddXRayTraceId()
        .AddAWSInstrumentation()
        .AddSource(nameof(Worker))
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App4"))
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri(hostContext.Configuration["Otlp:Endpoint"]);
        });

    Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());
});
```

### **2. Instrument the call to AWS SQS and AWS DynamoDb**

As I stated before the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package auto-instruments the downstream calls that are being made by the AWS SDK clients, but for this app we need more than that, we need to link the Activity from App3 with the Activity from App4, and the only way to do it is by hand.

In the previous section I talked a little bit about the ``AWSTraceHeader`` attribute and how it was injected into each message sent to SQS and that the attribute contains the X-Ray trace header. We're going to use this attribute to link both activities.

In the next code snippet the Worker Service runs on an infinite loop and keeps pulling messages using the ``ReceiveMessageAsync(request, stoppingToken)`` method.   
This is because Amazon SQS is not push-based like RabbitMQ, instead of that you need to keep pulling the messages from the queue by yourself.  

When a new message is pulled from SQS we call the ``ProcessMessage()`` method, this method will store the message into a DynamoDb Table and subsequently delete it from the SQS queue. 

The ``ProcessMessage()`` method creates a new Activity, extracts the ``AWSTraceHeader`` from the SQS message using the ``XRayPropagator`` propagator and links both producer and consumer activities.

The instrumentation of DynamoDb is much more simpler, there is no need to do any extra leg work, the auto-instrumentation from the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package will take care of it.

To instrument the call to delete the message from the SQS queue,  there is no need to do any extra leg work here either, the auto-instrumentation will do it for us.

```csharp
public class Worker : BackgroundService
{
  private static readonly ActivitySource Activity = new(nameof(Worker));

  private readonly ILogger<Worker> _logger;
  private readonly IConfiguration _configuration;
  private readonly IAmazonSQS _sqs;
  private readonly IAmazonDynamoDB _dynamoDb;

  public Worker(ILogger<Worker> logger,
      IConfiguration configuration, 
      IAmazonSQS sqs, 
      IAmazonDynamoDB dynamoDb)
  {
      _logger = logger;
      _configuration = configuration;
      _sqs = sqs;
      _dynamoDb = dynamoDb;
  }

  protected override async Task ExecuteAsync(CancellationToken stoppingToken)
  {
      while (!stoppingToken.IsCancellationRequested)
      {
          try
          {
              var request = new ReceiveMessageRequest
              {
                  QueueUrl = _configuration["SQS:URI"],
                  MaxNumberOfMessages = 1,
                  WaitTimeSeconds = 5,
                  AttributeNames = new List<string> { "All" }
              };

              var result = await _sqs.ReceiveMessageAsync(request, stoppingToken);

              if (result.Messages.Any())
              {
                  await ProcessMessage(result.Messages[0], stoppingToken);
              }
          }
          catch (Exception ex)
          {
              _logger.LogError(ex.ToString());
          }

          _logger.LogInformation("Worker running at: {time}", DateTime.UtcNow);
          Thread.Sleep(10000);
      }
  }

  private async Task ProcessMessage(Message msg, CancellationToken cancellationToken)
  {
      _logger.LogInformation("Processing messages from SQS");

      var propagator = Propagators.DefaultTextMapPropagator;
      var parentContext = propagator.Extract(default,
              msg,
              ActivityHelper.ExtractTraceContextFromMessage);

      Baggage.Current = parentContext.Baggage;

      using (var activity = Activity.StartActivity("Process SQS Message", 
          ActivityKind.Server, 
          parentContext.ActivityContext))
      {
          ActivityHelper.AddActivityTags(activity);
          
          _logger.LogInformation("Add item to DynamoDb");

          var items = Table.LoadTable(_dynamoDb, "Items");
          
          var doc = new Document
          {
              ["Id"] = Guid.NewGuid(),
              ["Message"] = msg.Body
          };

          await items.PutItemAsync(doc, cancellationToken);

          await _sqs.DeleteMessageAsync(_configuration["SQS:URI"], msg.ReceiptHandle, cancellationToken);
          
      }
  }
}
```
There is an extra thing worth mentioning on App4. 

Previously I stated that the ``ProcessMessage()`` method creates a new Activity, uses the ``XRayPropagator`` to extract the ``AWSTraceHeader`` from the message and links the producer and consumer activities.    
But the ``XRayPropagator`` is built to extract the ``X-Amzn-Trace-Id`` header not the ``AWSTraceHeader``, if you want to extract another header you'll need to specify it on the getter delegate, like this:

```csharp
public static class ActivityHelper
{
    public static IEnumerable<string> ExtractTraceContextFromMessage(Message msg, string key)
    {
        try
        {
            if (msg.Attributes.TryGetValue("AWSTraceHeader", out var value))
            {
                return new[] { value };
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Failed to extract trace context: {ex}");
        }

        return Enumerable.Empty<string>();
    }

    public static void AddActivityTags(Activity activity)
    {
        activity?.SetTag("messaging.system", "sqs");
    }
}
```

# AWS X-Ray output

After instrumenting these 4 apps we are going to generate some traffic and afterwards access X-Ray to start analyzing the traces that the apps are sending.

First let's invoke App1 ``/publish-message`` endpoint and take a look at an entire trace, this is how it looks on X-Ray:

![xray-fulltrace](/img/xray-fulltrace.png)

Also we can take a look at the X-Ray ServiceMap:

![xray-service](/img/xray-servicemap.png)

In the ServiceMap we can toggle between showing the map with icons or response times.

![xray-service](/img/xray-servicemap-noicons.png)

If we drill down into one of the spans that were auto-generated by the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package, we can see that some AWS metadata is added into the span.

![xray-fulltrace-dynamo-subsegment-resources](/img/xray-fulltrace-dynamo-subsegment-resources.png)

Also, do you remember that in App1 we added some extra Tags on the Activity?

```csharp
activity?.SetTag("messaging.system", "rabbitmq");
activity?.SetTag("messaging.destination_kind", "queue");
activity?.SetTag("messaging.rabbitmq.queue", "sample");
```
You'll find them on X-Ray too if you drill down on the _"Publish Message"_ span

![x-ray-fulltrace-tags-annotations](/img/x-ray-fulltrace-tags-annotations.png)

To end this, let's also invoke the ``/http`` endpoint from App1 and see an entire trace on XRay.   
Here it is:

![xray-fulltrace-http-endpoint](/img/xray-fulltrace-http-endpoint.png)

Every thing seems to be working fine, but there is one little problem...

# AWS auto-instrumentation additional tracing noise on App4

After having the app running for a short period of time, if we take a look at the traces sent to X-Ray we can see a strange phenomenom.

![xray-fulltrace-sqs-noise.png](/img/xray-fulltrace-sqs-noise.png)

A bunch of additional traces with no URL appeared on X-Ray.   
If we take a look at the full list of traces, we can see all those misterious traces with no URL.

![xray-fulltrace-sqs-noise-2.png](/img/xray-fulltrace-sqs-noise-2.png)

If we drill down on any of those traces we will see that the trace contains a call from App4 to Amazon SQS.

![xray-fulltrace-sqs-noise-3.png](/img/xray-fulltrace-sqs-noise-3.png)

Do you remember when I said that Amazon SQS uses a pull-based approach to retrieve new messages? 

The Worker Service works on an infinite loop, and in each loop iteration it is making a call to AWS SQS to try to fetch new messages.   
Also App4 uses the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package to auto-instrument any downstrem call made to an AWS Service.   

You see the problem, right? Each of these calls to SQS to try to fetch new messages in combination with the auto-instrumentation provided by the  ``OpenTelemetry.Contrib.Instrumentation.AWS`` ends up generating a bunch of useless traces.

That's a problem, because I don't want to send thousands of worthless traces into X-Ray each hour just because my Worker Service is polling AWS SQS to find if there is any new message to process.

What could we do here? The solution is simple but requires and extra bit of code.
- First of all, we need  to remove the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package from App4, with this kind of app the AWS SDK auto-instrumentation is more problematic than anything else.
- Instrument the calls to AWS DynamoDb manually.
- Instrument the calls to Amazon SQS manually.

First step, let's remove the ``AddAWSInstrumentation()`` extension method from the OTEL client.   
Also, we'll need to create a couple of extra ActivitySources:
- The "Dynamo.PutItem" ActivitySource will be used for instrumenting the call to Dynamo.
- The "SQS.DeleteMessage" ActivitySource will be used for instrumenting the call to SQS.

```csharp
 services.AddOpenTelemetryTracing(builder =>
{
    var provider = services.BuildServiceProvider();
    IConfiguration config = provider
            .GetRequiredService<IConfiguration>();

    builder.AddAspNetCoreInstrumentation()
        .AddXRayTraceId()
        .AddSource(nameof(Worker))
        .AddSource("Dynamo.PutItem")
        .AddSource("SQS.DeleteMessage")
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App4"))
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri(hostContext.Configuration["Otlp:Endpoint"]);
        });
});

Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());
```

Inside the ``Process SQS Message`` Activity we're going to create 2 new activities. 
- The ``Dynamo.PutItem`` Activity, that wraps the calls to DynamoDb.
- The ``SQS.DeleteMessage`` Activity, that wraps the call to the SQS ``DeleteMessageAsync`` method.

Those 2 activities will have the ``Process SQS Message`` Activity as their Parent Activity.

Also, we need to set a series of metadata tags if we want X-Ray to recognize each Activity as an AWS Service.

For the ``Dynamo.PutItem`` Activity will set the following tags:
```csharp
dynamoActivity?.SetTag("aws.service", "DynamoDbV2");
dynamoActivity?.SetTag("aws.operation", "PutItem");
dynamoActivity?.SetTag("aws.table_name", "Items");
dynamoActivity?.SetTag("aws.region", "eu-west-1");
```

For the ``SQS.DeleteMessage`` Activity will set the following tags:
```csharp
sqsActivity?.SetTag("aws.service", "SQS");
sqsActivity?.SetTag("aws.operation", "DeleteMessage");
sqsActivity?.SetTag("aws.queue_url", _configuration["SQS:URI"]);
sqsActivity?.SetTag("aws.region", "eu-west-1");
var response = await _sqs.DeleteMessageAsync(_configuration["SQS:URI"], msg.ReceiptHandle, cancellationToken);
sqsActivity?.SetTag("aws.requestId", response.ResponseMetadata.RequestId);
sqsActivity?.SetTag("http.status_code", (int)response.HttpStatusCode);
```

Here's how the Worker Service looks like after making those changes:

```csharp
public class Worker : BackgroundService
{
    private static readonly ActivitySource Activity = new(nameof(Worker));
    private static readonly ActivitySource ActivityDynamo = new("Dynamo.PutItem");
    private static readonly ActivitySource ActivitySQSDelete = new("SQS.DeleteMessage");

    private readonly ILogger<Worker> _logger;
    private readonly IConfiguration _configuration;
    private readonly IAmazonSQS _sqs;
    private readonly IAmazonDynamoDB _dynamoDb;

    public Worker(ILogger<Worker> logger,
        IConfiguration configuration, 
        IAmazonSQS sqs, 
        IAmazonDynamoDB dynamoDb)
    {
        _logger = logger;
        _configuration = configuration;
        _sqs = sqs;
        _dynamoDb = dynamoDb;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var request = new ReceiveMessageRequest
                {
                    QueueUrl = _configuration["SQS:URI"],
                    MaxNumberOfMessages = 1,
                    WaitTimeSeconds = 5,
                    AttributeNames = new List<string> { "All" }
                };

                var result = await _sqs.ReceiveMessageAsync(request, stoppingToken);

                if (result.Messages.Any())
                {
                    await ProcessMessage(result.Messages[0], stoppingToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex.ToString());
            }

            _logger.LogInformation("Worker running at: {time}", DateTime.UtcNow);
            Thread.Sleep(10000);
        }
    }

    private async Task ProcessMessage(Message msg, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Processing messages from SQS");

        var propagator = Propagators.DefaultTextMapPropagator;
        var parentContext = propagator.Extract(default,
                msg,
                ActivityHelper.ExtractTraceContextFromMessage);

        Baggage.Current = parentContext.Baggage;

        using (var activity = Activity.StartActivity("Process SQS Message", 
            ActivityKind.Server, 
            parentContext.ActivityContext))
        {
            using (var dynamoActivity = Activity.StartActivity("Dynamo.PutItem",
                        ActivityKind.Client,
                        activity.Context))
            {
                try
                {
                    _logger.LogInformation("Add item to DynamoDb");

                    dynamoActivity?.SetTag("aws.service", "DynamoDbV2");
                    dynamoActivity?.SetTag("aws.operation", "PutItem");
                    dynamoActivity?.SetTag("aws.table_name", "Items");
                    dynamoActivity?.SetTag("aws.region", "eu-west-1");

                    var items = Table.LoadTable(_dynamoDb, "Items");

                    var doc = new Document
                    {
                        ["Id"] = Guid.NewGuid(),
                        ["Message"] = msg.Body
                    };

                    await items.PutItemAsync(doc, cancellationToken);
                }
                catch (Exception ex)
                {
                    if (dynamoActivity != null)
                    {
                        ProcessException(dynamoActivity, ex);
                    }
                    throw;
                }
            }

            using (var sqsActivity = Activity.StartActivity("SQS.DeleteMessage",
                        ActivityKind.Client,
                        activity.Context))
            {
                try
                {
                     _logger.LogInformation("Delete message from SQS");

                    sqsActivity?.SetTag("aws.service", "SQS");
                    sqsActivity?.SetTag("aws.operation", "DeleteMessage");
                    sqsActivity?.SetTag("aws.queue_url", _configuration["SQS:URI"]);
                    sqsActivity?.SetTag("aws.region", "eu-west-1");
                    var response = await _sqs.DeleteMessageAsync(_configuration["SQS:URI"], msg.ReceiptHandle, cancellationToken);
                    sqsActivity?.SetTag("aws.requestId", response.ResponseMetadata.RequestId);
                    sqsActivity?.SetTag("http.status_code", (int)response.HttpStatusCode);
              
                }
                catch (Exception ex)
                {
                    if (sqsActivity != null)
                    {
                        ProcessException(sqsActivity, ex);
                    }
                    throw;
                }
            }
        }
    }

    private void ProcessException(Activity activity, Exception exception)
    {
        activity.RecordException(exception);
        activity.SetStatus(Status.Error.WithDescription(exception.Message));
    }
}
```

After applying those changes if we take a look at X-Ray, we will notice that the extra traces are gone.

If we invoke the ``/publish-message`` endpoint from App1 and take a peek at an entire trace, everything looks good.

![xray-fulltrace-sqs-noise-6.png](/img/xray-fulltrace-sqs-noise-6.png)

The XRay ServiceMap is still able to recognize that we're doing a series of downstream calls to DynamoDb and SQS thanks to the extra metadata we added on those activies.

![xray-fulltrace-sqs-noise-5.png](/img/xray-fulltrace-sqs-noise-5.png)

Also if we generate an exception when trying to access DynamoDb, it gets recorded properly.

![xray-fulltrace-sqs-noise-7.png](/img/xray-fulltrace-sqs-noise-7.png)

Everything look good now.

# How to run the demo

If you want to take a look at the demo, I have uploaded everything on my [GitHub repository](https://github.com/karlospn/aws-otel-tracing-demo)

The ``/cdk`` folder contains a CDK app that creates the AWS Resources needed to run the demo.   
The repository also contains a **docker-compose** file that starts up the 4 apps and the AWS OTEL collector, but there is **some work** that needs to be done in the docker-compose file before you can execute it:

- If you take a look at the compose file you'll see that there are a few values that **MUST** be replaced:
    - ``<ADD-AWS-USER-ACCESS-KEY>``
    - ``<ADD-AWS-USER-SECRET-KEY>``
    - ``<ADD-AMAZONMQ-RABBIT-HOST-ENDPOINT>``
    - ``<ADD-S3-BUCKET-NAME>``
    - ``<ADD-SQS-URI>``

After running the CDK app, you'll find the correct values in the output of the CDK app. Here's an example of how the output of the AWS CDK app looks like:

![cdk-app-output-censored](/img/aws-otel-cdk-output-censored.png)

And here's an example of how the docker-compose looks like after replacing the placeholder values:

```yaml
version: '3.4'

networks:
  tracing:
    name: tracing-network
    
services:

  otel:
    image: amazon/aws-otel-collector:latest
    command: --config /config/config.yml
    volumes:
      - ./aws-otel-collector-config:/config
    environment:
      - AWS_ACCESS_KEY_ID=AKIA5S6L5S6L5S6L5S6L
      - AWS_SECRET_ACCESS_KEY=GO7BvT9IBLb4NudL0aGO7BvT9IBLb4NudL0a
      - AWS_REGION=eu-west-1
    ports:
      - 4317:4317
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
      - otel
      - app3
    environment:
      RABBITMQ__HOST: b-27c79732-ce9d-47dc-8c51-13eec94c267e.mq.eu-west-1.amazonaws.com
      RABBITMQ__USERNAME: specialguest
      RABBITMQ__PASSWORD: P@ssw0rd111!
      APP3ENDPOINT: http://app3/dummy
      OTLP__ENDPOINT: http://otel:4317
      OTEL_RESOURCE_ATTRIBUTES: service.name=App1

  app2:
    stdin_open: true
    tty: true
    build:
      context: ./App2.RabbitConsumer.Console
    networks:
      - tracing
    depends_on: 
      - otel
      - app3
    environment:
      RABBITMQ__HOST: b-27c79732-ce9d-47dc-8c51-13eec94c267e.mq.eu-west-1.amazonaws.com
      RABBITMQ__USERNAME: specialguest
      RABBITMQ__PASSWORD: P@ssw0rd111!
      APP3URIENDPOINT: http://app3
      OTLP__ENDPOINT: http://otel:4317
      OTEL_RESOURCE_ATTRIBUTES: service.name=App2


  app3:
    build:
      context: ./App3.WebApi
    ports:
      - "5001:80"
    networks:
      - tracing
    depends_on: 
      - otel
    environment:
      OTLP__ENDPOINT: http://otel:4317
      OTEL_RESOURCE_ATTRIBUTES: service.name=App3
      S3__BUCKETNAME: aws-otel-demo-s3-bucket-17988
      SQS__URI: https://sqs.eu-west-1.amazonaws.com/7777777/aws-otel-demo-sqs-queue
      AWS_ACCESS_KEY_ID: AKIA5S6L5S6L5S6L5S6L
      AWS_SECRET_ACCESS_KEY: GO7BvT9IBLb4NudL0aGO7BvT9IBLb4NudL0a
  
  app4:
    build:
      context: ./App4.SqsConsumer.HostedService
    networks:
      - tracing
    depends_on: 
      - otel
    environment:
      OTLP__ENDPOINT: http://otel:4317
      OTEL_RESOURCE_ATTRIBUTES: service.name=App4
      SQS__URI: https://sqs.eu-west-1.amazonaws.com/7777777/aws-otel-demo-sqs-queue
      AWS_ACCESS_KEY_ID: AKIA5S6L5S6L5S6L5S6L
      AWS_SECRET_ACCESS_KEY: GO7BvT9IBLb4NudL0aGO7BvT9IBLb4NudL0a
```

To summarize, if you want to run this demo you'll need to do the following steps:
- Execute the CDK app on your AWS account.
- Replace the values on the docker-compose.
- Execute ``docker-compose up``.
