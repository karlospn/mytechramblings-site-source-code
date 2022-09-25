---
title: "Getting started with OpenTelemetry Metrics in .NET"
date: 2022-09-23T11:13:32+02:00
tags: ["opentelemetry", "dotnet", "csharp", "metrics", "prometheus", "grafana"]
description: "OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as traces, metrics, and logs. In this post I’m going to show you how you can start using OpenTelemetry to generate metrics with .NET Core."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/opentelemetry-metrics-demo).

OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as **traces**, **metrics**, and **logs**. 

In one of my [previous posts](https://www.mytechramblings.com/posts/getting-started-with-opentelemetry-and-dotnet-core/) I talked about how to get started with OpenTelemetry and distributed tracing, today I want to **focus on metrics**.   

Instead of showing you how to instrument a "Hello World" .NET application using OpenTelemetry metrics, I decided to use a real life application.    
At the end of the post we will end up with an application that generates a series of custom metrics, those metrics will be ingested by a Prometheus Server and we will use Grafana to visualize them on a dashboard.

Before jumping to the practical part there are a few concepts OpenTelemetry and metrics that are worth talking about.

## **Metrics API**

The Metrics API allows users to capture measurements at runtime. The Metrics API is designed to process raw measurements, generally with the intent to produce continuous summaries of those measurements.

The Metrics API has three main components:

- **MeterProvider**: The entry point of the API, provides access to Meters.
- **Meter**: Responsible for creating Instruments.
- **Instrument**: Responsible for reporting Measurements.


Metrics in OpenTelemetry .NET are a somewhat unique implementation of the OpenTelemetry project, as the Metrics API is incorporated directly into the .NET runtime itself, as part of the ``System.Diagnostics.DiagnosticSource`` package. This means, users can instrument their applications and libraries to emit metrics by simply using the ``System.Diagnostics.DiagnosticSource`` package.

## **MeterProvider**

OpenTelemetry Metrics works by using the ``MeterProvider`` to create a ``Meter`` and associating it with one or more ``Instruments``, each of which is used to create a series of ``Measurements``.

The ``MeterProvider`` must be configured to collect metrics using the OpenTelemetry .NET SDK, to setup properly you need to use the ``AddOpenTelemetryMetrics()`` extension method from the ``OpenTelemetry.Extensions.Hosting`` NuGet package.  
The ``MeterProvider`` will hold all the configuration for metrics like Meter names, readers, etc.  

Here's a quick example of how to use the ``MeterProvider`` on .NET:

```csharp
builder.Services.AddOpenTelemetryMetrics(opts => opts
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("BookStore.WebApi"))
    .AddPrometheusExporter());
```

## **Meter, Instruments and Measurements**

A ``Meter`` is responsible for creating ``Instruments``, it must provide functions to create new ``Instruments``.

``Instruments`` are used to report ``Measurements``. 

``Measurements`` are what we create or observe in our applications.

The next code snippet shows:
- How to create a ``Meter``
- Use the ``Meter`` to create an ``Instrument`` of type ``Counter`` 
- How to report ``Measurements`` with it.   

```csharp
public class Program
{
    static Meter meter = new Meter("MyCounter");
    static Counter<int> myCounter = meter.CreateCounter<int>("my-counter");

    static void Main(string[] args)
    {
        while(true)
        {
            myCounter.Add(1);
        }
    }
}
```

## **Types of Instruments**

The OpenTelemetry specification provides 6 types of ``Instruments`` that we can capture ``Measurements`` with.   

### **Counter**

A ``Counter`` is a synchronous instrument that is always increasing, and only accepts non–negative values.

When using a ``Counter``, an ``Add`` operation will be available in the SDK, which must be provided with the non–negative number to increment the ``Counter`` by, along with an optional set of attributes to be attached.

### **Asynchronous Counter**

An Asynchronous Counter is an asynchronous Instrument which reports monotonically increasing value(s) when the instrument is being observed.

It differs from the Counter by operating via callback rather than the add function. When the instrument is observed, the callback is executed and will pass back one or more measurements expressed as absolute values (not delta values). Once the values have been passed, they will be changed into delta values internally.

### **Up Down Counter**

An UpDownCounter is a similar synchronous instrument to a Counter, but it allows negative delta values to be passed (it’s not monotonic). 

Where a Counter would be suited to represent the number of jobs that had been submitted, an UpDownCounter would be perfect to represent the current number of active jobs being processed (it can move up and down). 

An UpDownCounter presents an add operation that is identical to the Counter operation—with the exception that it accepts negative data values.

### **Asynchronous Up Down Counter**

Asynchronous UpDownCounter is an asynchronous Instrument which reports additive value when the instrument is being observed.

It provides a callback interface that returns one or more measurements, expressing each measurement as an absolute value which will be changed to a delta value internally.

### **Histogram**

A Histogram is a synchronous instrument which allows the recording of multiple values that are statistically relevant to each other. 

You would choose a Histogram when you don't want to analyze data points in isolation, but would rather generate statistical information about their distribution by tracking the number of values that fall in each predefined bucket, as well as the minimum and the maximum value.

Histograms have a single method that is exposed: record. Record takes a non–negative observation value and an optional set of attributes to be attached.

### **Asynchronous Gauge**

An Asynchronous Gauge is designed to represent values that do not make sense to sum, even if they share attribute data.

 An example of this would be the temperature in various rooms of a house. This is common data, but it does not make any sense to report it as a total value—you’d potentially want an average or maximum, but never a sum.

 In the same manner, as all Asynchronous instruments, a callback is passed when creating an Asynchronous Gauge, which can return one or more (in this case completely discrete) measurements.

## **Types of Instruments available on .NET**

In the above section we have seen the differents types of ``Instruments`` available in the OpenTelemetry specification. 

.NET 6 only supports 4 of the 6 ``Instruments`` types: 
- Counter
- Asynchronous Counter (in .NET is called ObservableCounter)
- Asynchronous Gauge (in .NET is called ObservableGauge)
- Histogram
 
.NET 7



They can be grouped into two categories: synchronous and asynchronous.


## **Exporters**