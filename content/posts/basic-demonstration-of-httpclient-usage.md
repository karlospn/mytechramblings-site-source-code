---
title: "Back to .NET basics: How you should be using HttpClient"
date: 2023-07-02T16:07:52+02:00
draft: true
tags: ["dotnet", "networking", "http", "basic"]
description: "TBD"
---

> **If you are a .NET veteran, this post is not intended for you**.

I'm well aware that there are a ton of great articles (and probably better than this one) on the Internet, explaining exactly how you should properly use HttpClient with .NET. However, the truth is, even with so many resources available, I still come across many cases where its usage is incorrect when I start reviewing code.

Therefore, I have decided to write a quick post about the different options available nowadays for using HttpClient.

I don't intend to provide a theoretical explanation of how HttpClient works internally, as there are other posts on the Internet that have done a much better job than I could. Instead, my goal is to create **a concise, simple, and practical post that highlights various scenarios where HttpClient is utilized and discuss the reasons behind its correct or incorrect usage.**

# **Netstat command**

The ``netstat`` command is a great networking tool that allows us to investigate active network connections on a system. 

``HttpClient`` is used to make HTTP requests to web servers and APIs, and it relies on underlying network connections to perform these tasks. By leveraging the ``netstat`` command, we can gain insights into the active connections created by ``HttpClient``, helping us identify potential issues.

To investigate the active connections that HttpClient creates using ``netstat``, you can open a command prompt or terminal and enter ``netstat -an`` (the '-a' flag shows all connections, and the '-n' flag displays IP addresses and port numbers instead of resolving hostnames).   
The output will provide a list of all active connections, along with their status, local and remote IP addresses, and associated port numbers.

Monitoring HttpClient connections using ``netstat`` can help you identify if your application is properly closing connections after use or if there are lingering connections that may lead to resource leaks. It can also reveal if there are connection failures, such as connections in a ``TIME_WAIT`` state, which might indicate issues with connection pooling or DNS resolution.

The next list show the ``netstat`` states and their meanings:

- ``ESTABLISHED``: This state indicates that a connection is active and data is being exchanged between the local and remote systems. It signifies a successful connection between the client and server.
- ``TIME_WAIT``: After the connection is closed, it enters the ``TIME_WAIT`` state. This state ensures that any delayed packets from the previous connection are handled properly. It typically lasts for a few minutes before the connection is fully closed.
- ``CLOSE_WAIT``: This state occurs when the local application has closed the connection, but the remote system has not acknowledged the closure yet. It usually implies that the local application is waiting for the remote system to release the connection.
- ``FIN_WAIT_1``, ``FIN_WAIT_2``: These states occur during the process of closing a connection. ``FIN_WAIT_1`` means the local system has initiated the closure, while ``FIN_WAIT_2`` indicates the remote system has acknowledged the closure, and the local system is waiting for a final acknowledgment.
- ``LAST_ACK``: This state appears when the local system has initiated the closure, sent a FIN packet, and is waiting for the final acknowledgment from the remote system before the connection is fully closed.
- ``SYN_SENT``: In this state, the local system has sent a synchronization (SYN) packet to initiate a connection with the remote system but has not received a response yet.
- ``SYN_RECEIVED``: The SYN_RECEIVED state occurs on the server side when it receives a SYN packet from the client and sends back its SYN-ACK packet to acknowledge the connection request.
- ``LISTEN``: When a server application is in the LISTEN state, it is waiting and ready to accept incoming connection requests from clients.
- ``CLOSING``: This state occurs when the local system has initiated the closure of the connection, but the remote system is also trying to close the connection simultaneously.


# **Scenario 1: Create a new HttpClient instance everytime**

## Source code

- A new HttpClient is instantiated everytime a new request comes in.
- The HttpClient is not disposed.

```csharp

```

## netstat output

- Every new request creates a new TCP connection.
- TCP connections are not being neither reused nor closed after being used, which means that they will hang for some time waiting for incoming connections.
- Idle connections will be closed after 2 minutes by the OS.

{{< video src="/videos/asd.mp4" type="video/mp4" preload="auto" >}}


## Pros & cons
### Pros
- None

### Cons
- A new HttpClient is being created everytime a new request comes in, which means that the application has a unnecessary overhead from establishing a new TCP connection every single time.
- A new TCP connection is created every single time, which means that if the application is under heavy load this approach can lead to the accumulation of idle TCP connections and a socket exhaustion scenario.


# **Scenario 2: Create a new HttpClient everytime and dispose of it immediately after finishing using it**

- A new HttpClient is instantiated everytime a new request comes in.
- The HttpClient is disposed immediately after being used.

```csharp

```

## netstat output

- Every new request creates a new TCP connection.
- The HttpClient gets disposed right away thanks to the ``using`` statement, which causes the TCP connections to move to a ``TIME_WAIT`` state.

{{< video src="/videos/asd.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons
### Pros: 
- It becomes harder for your app to suffer from a socket exhaustion scenario.

### Cons
- TCP connections are not released immediately after connection closure, so if the rate of requests is high, the app might still end up in a socket exhaustion scenario.
- A new HttpClient is still being created everytime a new request comes in, which means that the application has a unnecessary overhead from establishing a new TCP connection every single time.

# **Scenario 3: Create a static or singleton HttpClient instance and reuse it**

## Source code
- A ``static`` HttpClient instance is created and used by any requests received.

```csharp

```
## netstat output

- TCP connection are being reused.
- If the application is being idle for some time, then TCP connection will get closed by the OS. The next request will create a new TCP connection.

{{< video src="/videos/asd.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons
### Pros: 
- TCP connections are being reused.

### Cons
- HttpClient only resolves DNS entries when a TCP connection is created. If DNS entries changes regularly, then the client won't notice those updates.

# **Scenario 4: Create a static or singleton HttpClient instance with PooledConnectionLifetime and reuse it**

## Source code
- A ``static`` HttpClient instance with ``PooledConnectionLifetime`` set to the desired interval is created.

```csharp

```
## netstat output

- TCP connection are being reused.
- If the application is being idle for some time, then TCP connection will get closed by the OS. The next request will create a new TCP connection.
- The ``PooledConnectionLifetime`` allow us to  limit the lifetime of the TCP connection, so that DNS lookup is repeated when the connection is replaced.

{{< video src="/videos/asd.mp4" type="video/mp4" preload="auto" >}}

## Pros & cons
### Pros: 
- TCP connections are being reused.
- It solves the DNS change problems, from scenario 3.

### Cons
- None.

# **Scenario 5: Use IHttpClientFactory**

# **Scenario 6: Using HttpClient with .NET Framework and Autofac** 


