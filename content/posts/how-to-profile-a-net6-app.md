---
title: "Profiling a .NET6 app running in a docker container with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio"
date: 2022-02-22T18:49:37+01:00
description: "This post will demonstrate how you can do profiling with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio on a .NET6 application."
tags: ["csharp", "dotnet", "profiling"]
draft: true
---

# Introduction

Every software developer at some time or another has stumbled with an application that has performance issues. And when it happens I find that a lot of people are quite unfamiliar about what steps must they take to solve them.    
And that's what I decided to write this beginner's article. 

I have built a .NET6 application that has some performance issues and in this post we're going to try to spot them.

The issues on the demo are oversimplified and on a real scenario they will not be so easy to spot on, but the objective of this post is to serve as a beginner's guide, so when you face a real performance problem you know what are the basic steps that you should follow.

As you might have guessed, most of the topics I’m going to tackle are basic stuff, so if you’re a performance veteran this post is probably not for you.


# .NET CLI tools

A couple years ago Microsoft introduced a series of new diagnostic tools:

- ``dotnet-counters`` to view Performance Counters.
- ``dotnet-dump`` to capture and analyze Dumps.
- ``dotnet-trace`` to capture runtime events, GC collections and sample CPU stacks.




## Running .NET CLI tools in a container

