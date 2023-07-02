---
title: "A basic demonstration of how you should use HttpClient in .NET"
date: 2023-07-02T16:07:52+02:00
draft: true
tags: ["dotnet", "httpclient", "http"]
description: "TBD"
---

> **Before you continue reading, let me tell you that if you are a .NET veteran, this post is not intended for you. So, I'll see you next time**.

I'm aware that there are a ton of good articles (and probably better than this one) on the Internet telling exactly how you should use properly HttpClient with .NET, but the truth is that when I start reviewing code, I still find many cases where its use is not the correct one.    
Therefore, I have decided to write a post as simple as possible about the different options available today for using HttpClient.

I don't want to go into a theoretical explanation of how HttpClient works internally, as there are other posts on the Internet that have done a much better job than I could. Instead, my goal is to create **a concise, simple and practical post that highlights various scenarios where HttpClient is utilized and discuss the reasons behind its correct or incorrect usage.**

Without further ado, let's get down to business.


# **Netstat command**

In this post, I will be making extensive use of the ``netstat`` command-line utility. 

This tool grants us a detailed insight into all the active connections established from our application to any remote API using ``HttpClient``, enabling us to examine the status details of each connection.

Here's a quick example.




# **Scenario 1: Create a new HttpClient instance everytime**

# **Scenario 2: Create a new HttpClient everytime and dispose of it immediately after finishing using it**

# **Scenario 3: Create a static or singleton HttpClient instance and reuse it**

# **Scenario 4: Create a static or singleton HttpClient instance with PooledConnectionLifetime and reuse it**

# **Scenario 5: Use a IHttpClientFactory named client**

# **Scenario 6: Use a IHttpClientFactory typed client**

# **Using IHttpClientFactory with .NET Framework and Autofac** 


