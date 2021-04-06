---
title: "Getting started with OpenTelemetry and distributed tracing in .NET Core"
date: 2021-04-04T22:31:51+02:00
draft: true
---

> **Show me the code**   
> If you don't care about my ramblings I have upload the code on my [Github](https://github.com/karlospn/opentelemetry-tracing-demo)

A few months ago the first stable version of the OpenTelemetry SDK for dotnet was released and since then I have wanted to write a little bit about it.   

OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as traces, metrics, and logs. Im my post I'm going to focus solely on tracing. 

From my standpoint I think that there are at least four main concepts about tracing and OpenTelemetry that you need to know. 

## **Span and activities** 
  
A span is the building block that forms a trace. It has a unique identifier and represents a  a piece of the workflow in the distributed system. 

Multiple spans are pieced together to create a trace.  

Traces are often viewed as a "tree" of spans that reflects the time that each span started and completed.

In .NET is represented by an **Activity**.   
The OpenTelemetry SDK for dotnet is reusing the existing Activity and associated classes to represent the OpenTelemetry Span. 
This means, users can instrument their applications/libraries to emit OpenTelemetry compatible traces by using just the .NET Runtime.

To create a Span in .NET we must first create a new activity:

```csharp
private static readonly ActivitySource Activity = new(nameof(Program));
```
And then call "StartActivity" to begin recording, everything that happens inside the using block will be recorded into that Span.

```csharp
using (var activity = Activity.StartActivity("Process Message", ActivityKind.Consumer, parentContext.ActivityContext)){}
```

## **Propagators** 

Allows us to extract and inject context across process boundaries. 

This is typically required if you are not using any of the .NET communication libraries which has instrumentations already available which does the propagation (eg: HttpClient). In such cases, context extraction and propagation is the responsibility of the library itself.

To create a propagator in .NET we must first create a TextMapPropagator

```csharp
private static readonly TextMapPropagator Propagator = new TraceContextPropagator();
```

Use the Inject method to inject the current context:

```csharp
Propagator.Inject(new PropagationContext(activity.Context, Baggage.Current), props, InjectContextIntoHeader);
```

And the Extract method to extract it:

```csharp
 var parentContext = Propagator.Extract(default, ea.BasicProperties, ExtractTraceContextFromBasicProperties);
```

## **Exporters**

Let's be honest emiting traces is pretty useless if you don't have a backend capable of aggregating the traces and displaying those traces in a friendly manner. 

An exporter is how data gets sent to those different back-ends. 

Generally, an exporter translates the internal format into another defined format.

Those are the most well-known available trace exporters:
- Jaeger
- OTLP gRPC
- OTLP HTTP
- Zipkin

> In this post I'm going to use **Jaeger**.

## **Attributes**

Attributes are key:value pairs that provide additional information to a trace.

In .NET those are called Tags. We can add an attribute into an activity like this:

```csharp
  activity?.SetTag("messaging.system", "rabbitmq");
  activity?.SetTag("messaging.destination_kind", "queue");
  activity?.SetTag("messaging.rabbitmq.queue", "sample");
```

I'm going to stop talking about OpenTelemetry theory because I could end up writing and entire post about it, and to be honest I would end up paraphrasing quite a bit of the official docs, so instead of that I'm going to point you towards the official documentation: https://opentelemetry.io/docs/concepts/


After talking a little bit about OpenTelemetry let me refocus on the **real** purpose of this post.   
In this post I want to show you **a practical example about how you can start using OpenTelemetry when you have a series of .NET microservices that communicate among them.**


# Starting point

I have built 4 apps beforehand, these apps are not doing any business logic operations because that's not the goal here.    
The important part is that all the apps are talking among them and also using other external services like Redis, MSSQL or Rabbit.   

Here's the diagram:

![otel-diagram](/img/otel-components-diagram.png)


- **App1.WebApi** is a **NET5 Web API** with 2 endpoints:
    - **/http** endpoint : makes an HTTP call to App2 _"dummy"_ endpoint.
    - **/publish-message** endpoint : publishes a message into a RabbitMq queue named _"sample"_.
    
- **App2.WebApi** is a **NET5 Web API** with 2 endpoints
    - **/dummy** endpoint : returns a fixed _"Ok"_ response.
    - **/sql-to-event** endpoint : receives a message via HTTP POST and stores it in a MSSQL Server.   
    After that, it publishes an event into a RabbitMq queue named _"sample_2"_.

- **App3.RabbitConsumer.Console** is a **NET5 console** application. 
  - Reads the messages from the Rabbitmq _"sample"_ queue and makes a HTTP call to **App2.WebApi** _"/sql-to-event"_ endpoint with the content of the message.

- **App4.RabbitConsumer.HostedService** is a **NET5 Worker Service**.
  - The Hosted Service reads the messages from the Rabbitmq _"sample_2"_ queue and stores it into a Redis cache database.

Those apps are not using OpenTelemetry right now, so in the next sections we're going to do a step-by-step guide about how to setup and use the OpenTelemetry SDK for dotnet and also how to start collecting and visualizing the traces.

# OpenTelemetry .NET Client

To get started with OpenTelemetry we're going to install the following packages.

```xml
<PackageReference Include="OpenTelemetry" Version="1.1.0-beta1" />
<PackageReference Include="OpenTelemetry.Exporter.Jaeger" Version="1.1.0-beta1" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.0.0-rc3" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.0.0-rc3" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.0.0-rc3" />
<PackageReference Include="OpenTelemetry.Instrumentation.SqlClient" Version="1.0.0-rc3" />
<PackageReference Include="OpenTelemetry.Instrumentation.StackExchangeRedis" Version="1.0.0-rc3" />
```

- The _**OpenTelemetry**_ package is the core library.
- The _**OpenTelemetry.Exporter.Jaeger**_ package allows us to export the traces from the application to Jaeger.
- The _**OpenTelemetry.Extensions.Hosting**_ package contains some _"Startup"_ extensions that allows us to register OpenTelemetry into the application DI container.
- The _**OpenTelemetry.Instrumentation.***_ packages are instrumentation libraries for third parties libraries.   
These packages are instrumenting common libraries/functionalities/classes so we don't have to do all the heavy lifting by ourselves.   
In our case we're using the following ones:
  - The _**OpenTelemetry.Instrumentation.AspNetCore**_ instruments ASP.NET Core and collect telemetry about incoming web requests. This instrumentation also collects incoming gRPC requests using Grpc.AspNetCore.
  - The _**OpenTelemetry.Instrumentation.Http**_ instruments System.Net.Http.HttpClient and System.Net.HttpWebRequest and collects telemetry about outgoing HTTP requests. 
  - The _**OpenTelemetry.Instrumentation.SqlClient**_ instruments Microsoft.Data.SqlClient and System.Data.SqlClient and collects telemetry about database operations.
  - The _**OpenTelemetry.Instrumentation.StackExchangeRedis**_ instruments StackExchange.Redis and collects telemetry about outgoing calls to Redis.


## Install and run OpenTelemetry on the App1


## Install and run OpenTelemetry on the App2


## Install and run OpenTelemetry on the App3


## Install and run OpenTelemetry on the App4