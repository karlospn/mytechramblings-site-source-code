---
title: "How to centrally manage nuget versions within a solution"
date: 2020-08-28T16:57:20+02:00
tags: ["csharp", "dotnet", "nuget"]
draft: true
---

A pretty common scenario in _line of business_ (LOB) applications is having multiple csharp projects inside a single solution, even with the proliferation of microservices is pretty common seeing the client project and the business logic decoupled in multiple projects inside a single solution.   
And it's even more common if you're using some kind of software architecture  like _"Domain Driven Design"_, "_Hexagonal Architecture"_ or _"Clean Architecture"_.   

A common problem that usually appears when the number of projects begins to pile up inside a single solution is trying to mantain consistency between which nuget version is using each project.   

And it's even worse if there are multiple applications inside a single solution, because i'm pretty sure that in some in the time these applications will start sharing projects between them and then it becomes a nightmare trying to consolidate nuget versions. You update a nuget version inside a project because you are modifying an application and you're affecting the other applications without even knowing.

That's why in most cases I say: "Fuck it! I'm going to manage the nuget package versions in a central point inside a solution.". And if my solution contains multiple applications then every application inside the solution is affected but I don't need to try to pinpoint the project mesh and if I want to update only one application I grab the app and put it in a new solution.

Anyways, in these post I want to show some of the different options available when you want to centrally manage nuget versions within a solution.


# Microsoft.Build.CentralPackageVersions 

# MSBuild

# Visual Studio

That's the most cumbersome option, you need to open the solution in Visual Studio, wait for the IDE to load, go to "Manage nuget package in the solution", wait for the nugets to load and finally begin to manage the nugets one by one.
It's the most manual and tedious option and in my opinion it's worse than having a plain file that describe the nuget package versions. But to be honest it's probably the option that most people uses or has used in the past. 
Everyone has done it, me included.

# ManagePackageVersionsCentrally
