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

With the AWS Distro for OpenTelemetry you can instrument your application to send metrics and traces to multiple AWS solutions such as AWS CloudWatch, Amazon Managed Service for Prometheus, AWS X-Ray or Amazon OpenSearch Service.   

It also can collect metadata from your AWS resources and managed services, so you can correlate application performance data with underlying infrastructure data, reducing the mean time to problem resolution.

As I stated before the goal for this post is testing distributed tracing with AWS Distro for OpenTelemetry in .NET, so during this post I'll be tackling the following topics:
- How to setup the AWS OTEL Collector.
- How to configure properly a .NET app and how to use XRay to view them.

But first of all, let's review the demo I'm going to build during this post.

# Demo

The demo has 4 apps. These apps are not doing any business logic operations because that's not the goal here.
The important part is that all the apps are talking with AWS services like S3, DynamoDb, AmazonMQ RabbitMQ or SQS.

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

# AWS OpenTelemetry Collector

The collector in our distro is built using the upstream OpenTelemetry collector. We have added AWS-specific exporters to the upstream collector to send data to AWS services including AWS X-Ray, Amazon CloudWatch and Amazon Managed Service for Prometheus. 


# Adding OpenTelemetry on App1

# Adding OpenTelemetry on App2

# Adding OpenTelemetry on App3

# Adding OpenTelemetry on App4

# AWS instrumentation tracing noise

# XRay output

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
