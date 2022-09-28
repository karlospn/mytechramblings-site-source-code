---
title: "Getting started with OpenTelemetry Metrics in .NET. Part 1: Key concepts"
date: 2022-09-27T10:01:32+02:00
tags: ["opentelemetry", "dotnet", "csharp", "metrics", "prometheus", "grafana"]
description: "In this 2 part series-post I’m going to show you how to use OpenTelemetry to generate custom metrics and how to visualize those metrics using Prometheus and Grafana. In part 1 I’ll be talking about some key concepts that you should know when using OpenTelemetry Metrics with .NET."
draft: false
---

> This is a 2 part-series post.
> - **Part 1**: Key concepts that you should know when using OpenTelemetry Metrics with .NET.
> - **Part 2**: A practical example about how to add OpenTelemetry Metrics on a real life .NET app and how to visualize those metrics using Prometheus and Grafana. If you want to read it, click [here](https://www.mytechramblings.com/posts/getting-started-with-opentelemetry-metrics-and-dotnet-part-2/)



OpenTelemetry is a set of APIs, SDKs, tooling and integrations that are designed for the creation and management of telemetry data such as **traces**, **metrics**, and **logs**. 

In one of my [previous posts](https://www.mytechramblings.com/posts/getting-started-with-opentelemetry-and-dotnet-core/) I talked about how to get started with OpenTelemetry and distributed tracing, today I want to **focus on metrics**.   

At the end of this 2 part-series post we will have a .NET6 app that emits a series of metrics, those metrics will be send to the OpenTelemetry Collector, a Prometheus Server will receive the metrics from the OTEL Collector and we will have a Grafana dashboard to visualize them.

![otel-metrics-app-diagram](/img/otel-metrics-app-diagram.png)

But before jumping to the practical part there are a few key concepts about using OpenTelemetry Metrics with .NET that are worth talking about.

# **Metrics API**

The Metrics API allows users to capture measurements at runtime. The Metrics API is designed to process raw measurements, generally with the intent to produce continuous summaries of those measurements.

The Metrics API has three main components:

- **MeterProvider**: The entry point of the API, provides access to Meters.
- **Meter**: Responsible for creating Instruments.
- **Instrument**: Responsible for reporting Measurements.


Metrics in OpenTelemetry .NET are a somewhat unique implementation of the OpenTelemetry project, as the Metrics API is incorporated directly into the .NET runtime itself, as part of the ``System.Diagnostics.DiagnosticSource`` package. This means, users can instrument their applications and libraries to emit metrics by simply using the ``System.Diagnostics.DiagnosticSource`` package.

# **MeterProvider**

OpenTelemetry Metrics works by using the ``MeterProvider`` to create a ``Meter`` and associating it with one or more ``Instruments``, each of which is used to create a series of ``Measurements``.

The ``MeterProvider`` must be configured to collect metrics using the OpenTelemetry .NET SDK, to set it up properly you need to use the ``AddOpenTelemetryMetrics()`` extension method from the ``OpenTelemetry.Extensions.Hosting`` NuGet package.  
The ``MeterProvider`` will hold all the configuration for metrics like Meter names, readers, etc.  

Here's a simple example of how to setup the ``MeterProvider`` on .NET:

```csharp
builder.Services.AddOpenTelemetryMetrics(opts => opts
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("BookStore.WebApi"))
    .AddPrometheusExporter());
```

# **Meter, Instruments and Measurements**

A ``Meter`` is responsible for creating ``Instruments`` and it must provide a series of functions to create new ``Instruments``.

``Instruments`` are used to report ``Measurements``. 

``Measurements`` are what we create or observe in our applications.


Here is a quick example for a better understading. The next code snippet shows:
- How to create a ``Meter``.
- How to use the ``Meter`` to create an ``Instrument`` of type ``Counter``.
- How to report ``Measurements`` with it.   

```csharp
public class Program
{
    static Meter meter = new Meter("MyMeter");
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

# **Types of Instruments**

The OpenTelemetry specification provides 6 types of instruments that we can capture measurements with. 

This 6 types of instruments can be grouped into two categories: synchronous and asynchronous.

- ## **Counter**

A ``Counter`` is a synchronous instrument that is always increasing, and only accepts non–negative values.

When using a ``Counter``, an ``Add`` operation will be available in the .NET SDK, which must be provided with the non–negative number to increment the ``Counter`` by, along with an optional set of attributes to be attached.

Here's a quick example of how to create and use a ``Counter`` instrument:

```csharp
Counter<int> BooksAddedCounter = meter.CreateCounter<int>("books-added", "Book", "Amount of books");
BooksAddedCounter.Add(1);
```
- ``books-added`` is the name of the ``Counter``.
- ``Book`` represents the unit of measure. The unit of measure is an optional string provided by the author of the instrument.
- ``Amount of books`` represents the instrument description. The description is an optional free-form text provided by the author of the instrument.

- ## **Asynchronous Counter**

An ``Asynchronous Counter`` is an asynchronous instrument which reports monotonically increasing value(s) when the instrument is being observed.

It differs from the ``Counter`` by operating via callback rather than the ``Add`` function.    
When the instrument is observed, the callback is executed and will pass back one or more measurements expressed as absolute values.

Here's a quick example of how to create and use an ``Asynchronous Counter`` instrument (aka ``ObservableCounter`` on .NET):

```csharp
ObservableCounter<int> OrdersCanceledCounter = meter.CreateObservableCounter<int>("orders-canceled", () => GetOrdersCanceled(), "Order", "Amount of orders cancelled");
```

- ``orders-canceled`` is the name of the ``Counter``.
- ``() => GetOrdersCanceled()`` is the callback function responsible for reporting ``Measurements``.
- ``Order`` represents the unit of measure. The unit of measure is an optional string provided by the author of the instrument.
- ``Amount of orders cancelled`` represents the instrument description. The description is an optional free-form text provided by the author of the instrument.

- ## **Histogram**

A ``Histogram`` is a synchronous instrument which allows the recording of multiple values that are statistically relevant to each other. 

You would choose a ``Histogram`` when you don't want to analyze data points in isolation, but would rather generate statistical information about their distribution by tracking the number of values that fall in each predefined bucket, as well as the minimum and the maximum value.

``Histograms`` have a single method that is exposed: ``Record``. ``Record`` takes a non–negative observation value and an optional set of attributes to be attached.

Here's a quick example of how to create and use an ``Histogram`` instrument:

```csharp
 Histogram<int> NumberOfBooksPerOrderHistogram = meter.CreateHistogram<int>("orders-number-of-books", "Book", "Number of books per order");
 NumberOfBooksPerOrderHistogram.Record(amount);
```
- ``orders-number-of-books`` is the name of the ``Histogram``.
- ``Book`` represents the unit of measure. The unit of measure is an optional string provided by the author of the instrument.
- ``Number of books per order`` represents the instrument description. The description is an optional free-form text provided by the author of the instrument.


- ## **Asynchronous Gauge**

An ``Asynchronous Gauge`` is designed to represent values that do not make sense to sum, even if they share attribute data.

An example of this would be the temperature in various rooms of a house. This is common data, but it does not make any sense to report it as a total value—you’d potentially want an average or maximum, but never a sum.

In the same manner, as all asynchronous instruments, a callback is passed when creating an ``Asynchronous Gauge``, which can return one or more measurements.

Here's a quick example of how to create and use an ``Asynchronous Gauge`` instrument (aka ``ObservableGauge`` on .NET):

```csharp
ObservableGauge<int> TotalCategoriesGauge = meter.CreateObservableGauge<int>("total-categories", () => GetTotalCategories(), "Category", "Get total amount of categories");
```
- ``total-categories`` is the name of the ``Gauge``.
- ``() => GetTotalCategories()`` is the callback function responsible for reporting ``Measurements``.
- ``Category`` represents the unit of measure. The unit of measure is an optional string provided by the author of the instrument.
- ``Get total amount of categories`` represents the instrument description. The description is an optional free-form text provided by the author of the instrument.


- ## **Up Down Counter**

An ``UpDown Counter`` is a similar synchronous instrument to a ``Counter``, but it allows negative delta values to be passed. 

Where a ``Counter`` would be suited to represent the number of jobs that had been submitted, a ``UpDown Counter`` would be perfect to represent the current number of active jobs being processed (it can move up and down). 

An ``UpDown Counter`` presents an ``Add`` operation that is identical to the ``Counter`` operation—with the exception that it accepts negative data values.

_Not available on .NET right now. More info about it in the next section._

- ## **Asynchronous Up Down Counter**

``Asynchronous UpDown Counter`` is an asynchronous instrument which reports additive value when the instrument is being observed.

It provides a callback interface that returns one or more measurements, expressing each measurement as an absolute value which will be changed to a delta value internally.

_Not available on .NET right now. More info about it in the next section._

# **Types of Instruments available on .NET**

In the above section we have seen the differents types of instruments available in the OpenTelemetry specification, but **.NET only supports 4 of the 6 instruments**

The supported instruments on .NET6 are the following ones:
- ``Counter``.
- ``ObservableCounter`` (aka ``Asynchronous Counter`` on the OpenTelemetry specification).
- ``ObservableGauge`` (aka ``Asynchronous Gauge`` on the OpenTelemetry specification).
- ``Histogram``.
 
The ``UpDown Counter`` and ``Asynchronous UpDown Counter`` instruments are **NOT** available right now on .NET. The ``Metrics API`` is incorporated directly into the .NET runtime itself, as part of the ``System.Diagnostics.DiagnosticSource`` package and that package does **NOT** support ``UpDown Counters`` nowadays.

**The support for the ``UpDown Counter`` is expected with the release of the stable version of .NET7**.   
More info about it here:
- https://github.com/open-telemetry/opentelemetry-dotnet/issues/2362

# **Choosing the correct instrument**

Choosing the correct instrument to report measurements is critical to achieving better efficiency, easing consumption for the user, and maintaining clarity in the semantics of the metric stream.

**I want to count something**
- If the value is monotonically increasing (the delta value is always non-negative), use a ``Counter``
- If the value is NOT monotonically increasing (the delta value can be positive, negative or zero), use an ``Asynchronous Gauge*``. 

**I want to record or time something**
- If you expect that the collected statistics are meaningful, use a ``Histogram``

**I want to measure something**
- If it makes NO sense to add up the values across different sets of attributes, use an ``Asynchronous Gauge``.
- If it makes sense to add up the values across different sets of attributes and the value is monotonically increasing, use an ``Asynchronous Counter``.

_*The correct instrument to use here is an ``UpDown Counter`` but .NET does not support this kind of instrument, so you'll have to use an ``Asynchronous Gauge`` as a workaround._

# **Exporters**

Let’s be honest emiting metrics is kind of pointless if you don’t have a backend capable of aggregating the metrics and displaying them in a friendly manner.

There are 2 ways to exporting data on OpenTelemetry:

- Using the OpenTelemetry Collector.
- Exporting the data directly into a back-end (like Prometheus, Jaeger, Zipkin, Elastic APM, Azure Monitor, etc).

## **Using the OpenTelemetry Collector**

The OpenTelemetry Collector is a standalone process designed to receive, process and export telemetry data.   

It removes the need to run, operate and maintain multiple agents/collectors in order to support open-source telemetry data formats (e.g. Jaeger, Prometheus, Zipkin, etc.) sending to multiple open-source or commercial back-ends.  

It eases the integration with your apps because you only need to export your data to a single endpoint, the collector endpoint, using the OTLP protocol.

![otel-metrics-exporter-otel-collector](/img/otel-metrics-exporter-otel-collector.png)

To send metrics to the OpenTelemetry Collector in .NET, you'll need to install the ``OpenTelemetry.Exporter.OpenTelemetryProtocol`` NuGet package on your application and configure the ``MeterProvider`` using the ``AddOtlpExporter`` extension method, like this:

```csharp
builder.Services.AddOpenTelemetryMetrics(opts => opts
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("BookStore.WebApi"))
    .AddMeter(meters.MetricName)
    .AddOtlpExporter(opts =>
    {
        opts.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"]);
    }));
```

## **Exporting the data directly into a backend**

You can export the metrics directly to a backend using the ``OpenTelemetry.Exporter.*
`` NuGet packages

![otel-metrics-exporter-backend](/img/otel-metrics-exporter-backend.png)

Here's an example of how to send the metrics data directly to Prometheus using the ``OpenTelemetry.Exporter.Prometheus.AspNetCore`` NuGet package:

```csharp
builder.Services.AddOpenTelemetryMetrics(opts => opts
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("BookStore.WebApi"))
    .AddMeter(meters.MetricName)
    .AddPrometheusExporter());
```

## **When to use the OpenTelemetry Collector**

Under what circumstances does one use a collector to send data, as opposed to having each service send it directly to the backend?

For trying out and getting started with OpenTelemetry, sending your data directly to a backend is a great way to get value quickly. Also, in a development or small-scale environment you can get decent results without a collector.

However, in general it's recommended to use the collector alongside your service, since it allows your service to offload data quickly and the collector can take care of additional handling like retries, batching, encryption or even sensitive data filtering.

Also using the collector eases the integration with your apps because you only need to export data to a single service using the OTLP protocol.    
If you send the data directly to a backend, you probably will end up with multiples configurations: a configuration to export the application traces into Jaeger or Zipkin, or whatever. Another configuration to export the metrics, another for logs, and so forth and so on.
