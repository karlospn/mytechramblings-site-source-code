---
title: "Testing AWS Distro for OpenTelemetry and distributed tracing with .NET apps"
date: 2022-01-24T11:19:12+01:00
tags: ["AWS", "dotnet", "OpenTelemetry"]
description: "TBD"
draft: true
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/aws-otel-tracing-demo).

# Introduction

AWS OTEL is a secure, production-ready, AWS-supported distribution of the OpenTelemetry project.

With AWS Distro for OpenTelemetry, you can instrument your applications just once to send correlated metrics and traces to multiple AWS and Partner monitoring solutions.

It also collects metadata from your AWS resources and managed services, so you can correlate application performance data with underlying infrastructure data, reducing the mean time to problem resolution.

And this repository contains a practical example about how to use AWS OTEL for tracing when we have a series of distributed .NET apps.

# Demo

# AWS OpenTelemetry Collector

# OpenTelemetry .NET Client

# Adding OpenTelemetry on App1

# Adding OpenTelemetry on App2

# Adding OpenTelemetry on App3

# Adding OpenTelemetry on App4

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
