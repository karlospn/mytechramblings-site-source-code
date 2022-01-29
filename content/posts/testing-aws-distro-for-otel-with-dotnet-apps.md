---
title: "Testing distributed tracing with AWS Distro for OpenTelemetry in .NET apps"
date: 2022-01-24T11:19:12+01:00
tags: ["AWS", "dotnet", "OpenTelemetry"]
description: "TBD"
draft: true
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/aws-otel-tracing-demo).

# Introduction

A few months ago I wrote a post about how to start using OpenTelemetry with .NET, if you're interested you can read more about it [here](https://www.mytechramblings.com/posts/getting-started-with-opentelemetry-and-dotnet-core/).   
In that post I talked a little bit about the main concepts around OpenTelemetry and how to instrument a .NET app with the .NET OpenTelemetry client.

In this post I'm going to get back to OpenTelemetry and .NET, but this time I'm planning to focus on testing the **AWS Distro for OpenTelemetry**.   
I'm sure there's going to be some overlap with my previous post, but I'll try it to keep it to a minimum. 

I'm not going to repeat myself talking about what's an activity, what's an span or a propagator, you can go read my other post.

In this post I'm planning to be talk about the following topics:
- What is the AWS OpenTelemetry Collector and how to setup it up.
- Build a demo to show you how to configure properly a .NET app to send the tracing data to the collector
- How to use XRay to view the traces sent.

But first of all: _What's the AWS Distro for OpenTelemtry?_

The AWS Distro for OpenTelemetry is an AWS-supported distribution of the OpenTelemetry project.

With the AWS Distro for OpenTelemetry you can instrument your application to send metrics and traces to multiple AWS solutions such as AWS CloudWatch, Amazon Managed Service for Prometheus, AWS X-Ray or Amazon OpenSearch Service.   

It also can collect metadata from your AWS resources and managed services, so you can correlate application data with infrastructure data, reducing the mean time to problem resolution.

# AWS OpenTelemetry Collector

What's the OpenTelemetry Collector? The OpenTelemetry Collector is a standalone process designed to receive, process and export telemetry data.   
It removes the need to run, operate and maintain multiple agents/collectors in order to support open-source telemetry data formats (e.g. Jaeger, Prometheus, Zipkin, etc.) sending to multiple open-source or commercial back-ends.

The Collector has two different deployment strategies: either running as an agent alongside a service, or as a remote application.    
In general, using both is recommended: the agent would be deployed with your service and run as a separate process or in a sidecar; meanwhile, the collector would be deployed separately, as its own application running in a container or virtual machine.

Here's a diagram showing how the collector works:

![otel-collector-diagram](/img/otel-collector-diagram.png)

In our case we want to send all the tracing data to the OpenTelemetry Collector and the Collector will send the data to XRay.

But wait a minute, we are not going to use the OpenTelemetry Collector, we are going to use the AWS OpenTelemetry Collector.

AWS instead of using directly the OpenTelemetry Collector has decided to implement their own Collector.    
The AWS OTEL Collector is built using the OpenTelemetry collector but they have added AWS-specific exporters to send data to AWS services including AWS X-Ray, Amazon CloudWatch or Amazon Managed Service for Prometheus.   

I could use the OpenTelemtry Collector with the AWS exporters and it would work, but using the AWS OTEL Collector is simpler and faster to setup, so I decided to use it for this demo.

# Configuring the AWS OpenTelemetry Collector

If you want to configure it you need to know that the configuration contains three main components:
- **Receivers**: They can be push or pull based and is how data gets into the Collector.  
- **Exporters**: They can be push or pull based and is how you send data to one or more backends/destinations. 
- **Processors**: They run on data between being received and being exported. Processors are optional.

For our demo purposes we need the following ones:
- **Receivers**:
  - **OTEL gRPC Receiver**: The .NET apps will have to send the tracing data to the collector and to achieve it we're going to use the OTLP gRPC receiver. This receiver receives data via gRPC using OTLP format.    
  Instead of the gRPC receiver we could use the HTTP receiver, this one receives data via HTTP/JSON.
- **Exporters**:
  - **XRay Exporter**: This exporter converts OTLP traces into the AWS X-Ray format and then exports the data to the AWS X-Ray service.
  - **Logging Exporter**: This exporter writes the OTLP traces to the console. This one is not needed, but I find it quite useful to review what data are the apps sending to the collector.
- **Processors**:
  - **Memory limiter Processor**: This processor ensure that the Collector does not goes crazy using memory.   
  This processor is really useful if you deploy the Collector as a sidecar along your application because it will ensure that the Collector does not use up memory needed by your application.
  - **Batch Processor**: This processor accepts spans, metrics, or logs and places them into batches. Batching helps better compress the data and reduce the number of outgoing connections required to transmit the data.

If you're curious about the logging exporter, here's an example of how output of the exporter looks like:

![otel-collector-logging-output](/img/aws-otel-collector-logging-output.png)

And here's the configuration file:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  memory_limiter:
    limit_mib: 50
    check_interval: 1s
  batch/traces:
    timeout: 1s
    send_batch_size: 50

exporters:
  logging:
    loglevel: debug
  awsxray:
    region: eu-west-1

service:
  pipelines:
    traces:
      receivers:
        - otlp
      processors:
        - memory_limiter
        - batch/traces
      exporters:
        - logging
        - awsxray
```

# Configuring permissions for the AWS OpenTelemetry Collector

Running the collector from my local machine in a docker-compose requires that you specify an ``AWS_ACCESS_KEY`` and ``AWS_SECRET_ACCESS_KEY``. Like this: 

```yaml
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
```

This user requires at least permissions for AWS CloudWatch Logs for metric publishing, and for AWS X-Ray for sending traces.

So, you'll need to attach this IAM policy:
```yaml
Copy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:DescribeLogGroups",
                "xray:*",
                "ssm:GetParameters"
            ],
            "Resource": "*"
        }
    ]
}
```

# Demo

In the previous section we have talked about what is the AWS OTEL Collector and how to set it up, now it's time to focus on building some apps.

The demo we're going to build will contain 4 apps, these apps are not doing any business logic operations because that's not the goal here.    
The key part of this demo is that the apps are communicating between them and also with some AWS services like S3, DynamoDb, AmazonMQ RabbitMQ or SQS.

Here's the diagram:

![components-diagram](/img/aws-otel-demo-components-diagram.png)

- **App1.WebApi** is a .NET6 Web API with 2 endpoints.
    - The ``/http`` endpoint  makes an HTTP request to the **App2** ``/dummy`` endpoint.
    - The ``/publish-message``  endpoint queues a message into an **AWS ActiveMQ Rabbit queue**.

- **App2.RabbitConsumer.Console** is a .NET6 console app. 
  - Dequeues messages from the Rabbit queue and makes a HTTP request to the **App3** ``/s3-to-event`` endpoint with the content of the message.

- **App3.WebApi** is a .NET6 Web API with 2 endpoints.
    - The ``/dummy`` endpoint returns a fixed "Ok" response.
    - The ``/s3-to-event`` endpoint receives a message via HTTP POST, stores it in an **S3 bucket** and afterwards publishes the message as an event into an **AWS SQS queue**.

- **App4.SqsConsumer.HostedService** is a .NET6 Worker Service.
  - A Hosted Service reads the messages from the **AWS SQS queue** and stores it into a **DynamoDb table**.

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
- The ``OpenTelemetry.Extensions.Hosting`` package contains some extensions that allows us to add the dependencies into the DI container and configure the HostBuilder.
- The ``OpenTelemetry.Instrumentation.*`` packages are instrumentation libraries. These packages are instrumenting common libraries/functionalities/classes so we don’t have to do all the heavy lifting by ourselves. In our example we’re using the following ones:
  - The ``OpenTelemetry.Instrumentation.AspNetCore`` instruments ASP.NET Core and collects telemetry about incoming web requests. This instrumentation also collects incoming gRPC requests using Grpc.AspNetCore.
  - The ``OpenTelemetry.Instrumentation.Http`` instruments the System.Net.Http.HttpClient and System.Net.HttpWebRequest types and collects telemetry about outgoing HTTP requests.
- The ``OpenTelemetry.Contrib.Extensions.AWSXRay`` package is used to convert the format of the OpenTelemetry traces into the X-Ray trace format.    
- The ``OpenTelemetry.Contrib.Instrumentation.AWS`` auto-instruments the AWS SDK clients and collects tracing data to downstream calls to AWS services.
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

- The  ``AddXRayTraceId()`` method is needed because we need to convert the OTEL trace ID into aAWS X-Ray compliant trace IDs, without this method the X-Ray is incapable of interpreting the application data traces.
- The ``AddHttpClientInstrumentation()`` method enables HTTP auto-instrumentation.The Api 1 makes an HTTP request to the App2, if we want to trace the HTTP call between these 2 apps we can do it simply by adding this extension method.
- The ``AddSource(nameof(PublishMessageController))`` method is used to add a new ActivitySource to the provider. Everytime you create a new Activity you should add a new ActivitySource.
- The ``.SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App1"))`` method configures the Resource for the application.
- The ``AddOtlpExporter()`` method is used to indicate that the tracing data is going to be sent to this specific endpoint. This endpoint matches the OTEL gRPC Receiver Endpoint in the AWS OTEL Collector.
- The ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator())`` method is needed if your app calls another application instrumented with AWS X-Ray SDK. This method overrides the value of the ``Propagators.DefaultTextMapPropagator`` with the ``AWSXRayPropagator``.    
The ``Propagators.DefaultTextMapPropagator`` is used internally by some instrumentation packages to infer and propagate the trace ID.


To put it simply, if you're using AWS Distro for OpenTelemetry and you want to send all the tracing data to X-Ray you'll need to add this 2 method everytime you setup the OpenTelemetry provider in one of your apps:
- ``AddXRayTraceId()``
- ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());``

All the other method are just basic OpenTelemetry setup, nothing to do with AWS Distro for OpenTelemetry.

### **2. Instrument the call to AmazonMQ RabbitMQ**

The ``OpenTelemetry.Contrib.Instrumentation.AWS`` NuGet package is used to auto-instrument the AWS SDK .NET clients, but there is no AWS SDK client to use with AmazonMQ Rabbit.   
That means that we need to used the ``RabbitMQ.Client`` package and that we need to instrument the downstream call by ourselves.

In the following code snippet you'll see that we are injecting the trace information into the header of the message that is going to be send.   
Later in App2 we'll extract the trace information from the message header and link both operations.

To be more specific we are using an ``XRayPropagator`` to inject the Activity into the message header, this concrete propagator will add a header named ``X-Amzn-Trace-Id`` into the message.   

When setting up the OpenTelemetry provider, we add the following command: ``Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator())``. This method overrides the value of the ``Propagators.DefaultTextMapPropagator`` with ``AWSXRayPropagator``.   
So now, we can get an instance of this specific propagator like this: ``var propagator = Propagators.DefaultTextMapPropagator``


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

The App2 is a console app so we have to setup the library in a different way. Instead of using an ``IServiceCollection`` extension we’re creating directly a TraceProvider, apart from that, the setup looks almost the same as the previous one.

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

This app retrieves a message from an ActiveMQ RabbitMQ cluster and makes an Http Request to App3.

The HTTP call is being instrumented automatically by the ``AddHttpClientInstrumentation()`` extension method, but the Rabbit part needs to be instrumented manually.

The code is pretty much the same as the one I have showed you on App1, but there are a 2 differences.   
The first one is pretty obvious, this time instead of inject the message we are using the ``XRayPropagator`` to extract the ``X-Amzn-Trace-Id`` header from the message and afterwards link both activies.

The second difference is more subtle, but be careful with this one or the traces sent to X-Ray are going to be malformed.

> Only **Activities of kind Server are converted into X-Ray segments**. All other kind of spans (Internal, Client, etc.) are converted into X-Ray subsegments.

That means that if we don't set Activity to be of  ``Kind = ActivityKind.Server`` the App2 is not going to show up correctly on XRay because it will be a subsegment.

In this kind of app is quite important to instantiate the activity like this: ``var activity = Activity.StartActivity("Process Message", ActivityKind.Server, parentContext.ActivityContext))``

So, why we need to set the ``Kind = ActivityKind.Server`` in this app and not on App1?   
That's because when we setup OpenTelemetry on a .NET WebAPI an Activity of ``Kind = ActivityKind.Server`` is created automatically everytime we invoke an API Controllerthanks to the ``AddAspNetCoreInstrumentation()`` extension method

If we take a look at how a trace on App1 looks like, you'll see this:

![x-ray-fulltrace-app1.png](/img/x-ray-fulltrace-app1.png)

- The parent Span is created automatically
- The Second Span ("Publish message span") is the one we have created manually. 

Remember this issue, because we're going to face it again on App4:

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

App3 is a Web API that uses AWS S3 and AWS SQS, and the setup of the OTEL provider looks almost identical as the ones on the App1 and App2 but with one difference.   
We are using the ``AddAWSInstrumentation()`` method from the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package to auto-instrument the calls that are being made by the AWS SDK clients.

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

App3 receives a message from App2 and upload sthe content of the message into an S3 bucket, and as you can see in the next code snippet there is nothing related to OpenTelemetry here.  
We're simple using the ``AWSSDK.S3`` client to upload the contents of the message into the S3 bucket.

That's because the  ``AddAWSInstrumentation()`` will auto-instrument the downstream calls made by the ``AWSSDK.S3`` client and the ``AWSSDK.SQS`` client.

The library will create an activity of ``Kind=Activity.Client`` foreach downstream call, it will also attach a few tags to each activity, this tags will contain metadata about the AWS  services being invoked.


```csharp
public class S3Repository : IS3Repository
{
    private readonly IAmazonS3 _s3Client;
    private readonly ILogger<S3Repository> _logger;

    public S3Repository(IAmazonS3 s3Client, 
        ILogger<S3Repository> logger)
    {
        _s3Client = s3Client;
        _logger = logger;
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
                BucketName = "vytestevent"
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
App3 will also queue the message received into an Amazon SQS, and as you can see in the next code snippet there is also nothing related to OpenTelemetry here.

There is one thing worth mentioning when working with Amazon SQS and the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package is that the ``AWSTraceHeader`` attribute is going to added into every message send.     
The ``AWSTraceHeader`` is a message system attribute reserved by Amazon SQS to carry the X-Ray trace header.   

This header is important, more about it in the next section.


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

App3 is a Worker Service that uses Amazon SQS and Amazon DynamoDb.

Nothing to comment here, the setup of the Otel provider is exactly the same as the one on App3.

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

As I stated before the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package auto-instruments the downstream calls that are being made by the AWS SDK clients, but we need more than that, we need to link the Activity on the producer side from App3 with the Activity on the consumer side from App4, and the only way to do that is manually.

In the previous section I talked about how the ``AWSTraceHeader`` attribute was injected into each message send to SQS and the attribute contained the X-Ray trace header, so we're going to use this attribute to link both activities.

Amazon SQS is not push-based like RabbitMQ, instead of that you need to keep pulling the messages from the queue by yourself.  

In the next code snippet the Worker Service runs on an infinite loop and keeps pulling messages using the ``ReceiveMessageAsync(request, stoppingToken)`` method.

When a message is pulled we call the ``ProcessMessage()`` method, this method will store the message received from SQS into DynamoDb and delete it from the SQS queue.   
The ``ProcessMessage()`` method creates a new Activity, uses the ``XRayPropagator`` to extract the ``AWSTraceHeader`` from the message and links the producer and the consumer activities.

The instrumentation of DynamoDb is much more simpler, there is no need to do any extra leg work, the auto-instrumentation from the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package will take care of it.

The last step of delete the message from the SQS queue,  there is no need to do any extra leg work here either, the auto-instrumentation will do it for us.

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
There is a extra thing worth mentioning on App4. Previously I stated that the ``ProcessMessage()`` method creates a new Activity, uses the ``XRayPropagator`` to extract the ``AWSTraceHeader`` from the message and links the producer and the consumer activities.

But the ```XRayPropagator`` is built to try to extract only  the ``X-Amzn-Trace-Id`` header, if you want to extract another header you'll need to specify it on the getter delegate.

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

# XRay output

After instrumenting the 4 apps we are going to generate some traffic and afterwards access XRay to start analyzing the traces that the apps are sending.

First let's invoke the ``/publish-message`` endpoint from App1 and let's take a look at an entire trace, this is how it looks on X-Ray:

![xray-fulltrace](/img/xray-fulltrace.png)

Also we can take a look at the X-Ray ServiceMap:

![xray-service](/img/xray-servicemap.png)

We can toggle between the ServiceMap with icons to the ServiceMap with response times

![xray-service](/img/xray-servicemap-noicons.png)

If we drill down into one of the spans that we're auto-generated by the ``OpenTelemetry.Contrib.Instrumentation.AWS`` package, you can see that some metadata about which resource and which action is performed is also added into the span.

![xray-fulltrace-dynamo-subsegment-resources](/img/xray-fulltrace-dynamo-subsegment-resources.png)

Also, do you remember that in the App1 we added some extra Tags on the Activity?

```csharp
activity?.SetTag("messaging.system", "rabbitmq");
activity?.SetTag("messaging.destination_kind", "queue");
activity?.SetTag("messaging.rabbitmq.queue", "sample");
```
You'll find them on XRay if you drill down on the _"Publish Message"_ span

![x-ray-fulltrace-tags-annotations](/img/x-ray-fulltrace-tags-annotations.png)

To end this, let's also invoke the ``/http`` endpoint from App1 and see an entire trace on XRay.   
Here it is:

![xray-fulltrace-http-endpoint](/img/xray-fulltrace-http-endpoint.png)

Every thing seems to be working fine, but there is one little important problem...

# AWS auto-instrumentation additional tracing noise on App4

Taking a look at the traces sent to X-Ray we see a problem on App4.




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
        .AddSource("SQS.DeleteMessageAsync")
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("App4"))
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri(hostContext.Configuration["Otlp:Endpoint"]);
        });
});

Sdk.SetDefaultTextMapPropagator(new AWSXRayPropagator());
```

```csharp
public class Worker : BackgroundService
{
    private static readonly ActivitySource Activity = new(nameof(Worker));
    private static readonly ActivitySource ActivityDynamo = new("Dynamo.PutItem");
    private static readonly ActivitySource ActivitySQSDelete = new("SQS.DeleteMessageAsync");

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
                _logger.LogInformation("Add item to DynamoDb");

                dynamoActivity?.SetTag("aws.service", "Amazon.DynamoDbV2");
                dynamoActivity?.SetTag("aws.operation", "PutItemAsync");
                dynamoActivity?.SetTag("aws.table_name", "Items");

                var items = Table.LoadTable(_dynamoDb, "Items");

                var doc = new Document
                {
                    ["Id"] = Guid.NewGuid(),
                    ["Message"] = msg.Body
                };

                await items.PutItemAsync(doc, cancellationToken);

            }

            using (var sqsActivity = Activity.StartActivity("SQS.DeleteMessageAsync",
                        ActivityKind.Client,
                        activity.Context))
            {

                sqsActivity?.SetTag("aws.service", "Amazon.SQS");
                sqsActivity?.SetTag("aws.operation", "DeleMessageAsync");
                sqsActivity?.SetTag("aws.queue_url", _configuration["SQS:URI"]);
                await _sqs.DeleteMessageAsync(_configuration["SQS:URI"], msg.ReceiptHandle, cancellationToken);

            }
        }
    }
}
```

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
      S3_BUCKET_NAME: aws-otel-demo-s3-bucket-17988
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
