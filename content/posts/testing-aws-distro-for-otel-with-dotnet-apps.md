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

I'm not going to repeat myself talking about what's an activity, what's an span or a propagator, you can go read my other post.   
In this one I want to go straight to the point, so I'm planning to be tackling the following topics:
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
  - **OTEL GRPC Receiver**: The .NET apps will have to send the tracing data to the collector and to achieve it we're going to use the OTLP GRPC receiver. This receiver receives data via gRPC using OTLP format.    
  Instead of the gRPC receiver we could use the HTTP receiver, this one receives data via HTTP/JSON.
- **Exporters**:
  - **XRay Exporter**: This exporter converts OTLP traces into the AWS X-Ray format and then exports the data to the AWS X-Ray service.
  - **Logging Exporter**: This exporter writes the OTLP traces to the console. This one is not needed, but I find it quite useful to review what data are the apps sending to the collector.
- **Processors**:

If you're curious about the logging exporter, here's an example of how output of the exporter looks like:

![otel-collector-logging-output](/img/aws-otel-collector-logging-output.png)

And here's the configuration file:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

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

In the previous section we have talked about what is the AWS OTEL Collector and how to set it up, now it's time to focus on building the .NET apps.

The demo we're going to build will contain 4 apps, these apps are not doing any business logic operations because that's not the goal here.    
The key part of this demo is that the apps are communicating between them and also with some AWS services like S3, DynamoDb, AmazonMQ RabbitMQ or SQS.

Here's the diagram:

![components-diagram]((/img/aws-otel-demo-components-diagram.png)

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

# Adding OpenTelemetry on App1

# Adding OpenTelemetry on App2

# Adding OpenTelemetry on App3

# Adding OpenTelemetry on App4

# XRay output

After instrumenting the 4 apps we are going to generate some traffic and afterwards access XRay  to start analyzing the traces that the apps are sending.

If we take a look at an entire trace, this is how it looks on X-Ray:

![xray-fulltrace](/img/xray-fulltrace.png)

Also we can take a look at the X-Ray ServiceMap:

![xray-service](/img/xray-servicemap.png)

If we want we can toggle between the ServiceMap with icons to the ServiceMap with response times

![xray-service](/img/xray-servicemap-noicons.png)

# AWS auto-instrumentation tracing noise


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
