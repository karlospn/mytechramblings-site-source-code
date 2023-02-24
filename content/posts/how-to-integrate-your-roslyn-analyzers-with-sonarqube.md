---
title: "How to integrate your Roslyn Analyzers with Sonarqube"
date: 2023-02-20T13:30:55+01:00
tags: ["dotnet", "roslyn", "devops", "sonarqube"]
description: "TBD"
draft: true
---

> **Show me the code**   
> As always, if you don't care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/how-to-integrate-roslyn-analyzers-with-sonarqube)

If you work regularly with .NET you're probably heard about Roslyn Analyzers, and if you don't know anything about them you're being using them without knowing it.

Starting with Visual Studio 2019 16.8 and .NET 5.0, a bunch of Roslyn Analyzers are included with the .NET SDK. Nowadays when you create a new .NET API in Visual Studio, it comes with quite a few a Roslyn Analyzers packages already preconfigured.

<add-img>

Currently you don't see much talk about Roslyn Analyzers, but they are still the de facto way when you want to perform some kind of custom static analysis in your codebase. 
A good example of custom Roslyn analyzer can be found in multiple library packages, that uses an analyzer to enforce the correct usage and setup of the library.

SonarQube is one of the most well-known tools that performs static analysis of code to detect bugs, code smells and security vulnerabilities.

