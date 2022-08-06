---
title: "How to easily check on your CI/CD pipeline if your app contains a NuGet package with a security vulnerability"
date: 2022-08-06T17:06:21+02:00
tags: ["devops", "dotnet", "nuget"]
description: "ToDo"
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/check-nuget-packages-for-security-vulnerabilities).

Almost every dotnet application contains a few NuGet dependencies, and those dependencies may have their own dependencies, and so on and so forth.
That means that your application ends up depending on dozens of dependencies.

What if any of those dependencies you're using are vulnerable? That could pose a risk of a supply chain attack.   
What is software supply chain attack?  It is an upstream vulnerability in one of your dependencies that can be fatal, making your app vulnerable to a potential compromise.

If you don't live under a rock, then you've likely heard about last year Log4j vulnerability, this was a software supply chain vulnerability in which having the popular Java logging framework installed on your app could compromise your business.

The term software supply chain is used to refer to everything that goes into your software and where it comes from. It is the dependencies and properties of your dependencies that your software supply chain depends on. A dependency can be code, binaries, or other components, and where they come from, such as a repository or package manager.   

One of the most important things you can do to protect your supply chain is to patch your vulnerable dependencies and keep track of the outdated ones.

There are quite a few tools that allow you to scan you appn for vulnerable dependencies, one of my favorites is (Snyk)[https://snyk.io/], also if your project is hosted on GitHub, you can leverage GitHub Security to find vulnerable depedencies in your project and Dependabot will fix them by opening up a pull request against your codebase.

If you don't use tool for scanning your dependencies right now, you should know that the dotnet CLI has a NuGet dependency scanner feature ready to use. 

# How to use the .NET CLI to check if your app has any vulnerable NuGet dependency

You can list any known vulnerabilities in your dependencies within your application with the `dotnet list package --vulnerable` command.

This command gets the security information from the centralized GitHub Advisory Database. This database provides two main listings of vulnerabilities:

- A CVE is Common Vulnerabilities and Exposures. This is a list of publicly disclosed computer security flaws.
- A GHSA is a GitHub Security Advisory. GitHub is a CVE Numbering Authority (CNA) and is authorized to assign CVE identification numbers.

To scan for vulnerabilities within your projects using the dotnet CLI you'll need to have installed the **.NET SDK version 5.0.200** (or a higher version) or **Visual Studio 2019 16.9** (or a higher version).

# How to integrate it with your CI/CD pipelines

You can check for any known NuGet vulnerability and break the pipeline execution with just a couple of lines of bash script, so there is little to no excuse for you not to do it.


# How to check if your app has outdated or deprecated NuGet versions

To ensure a secure supply chain of dependencies, you will want to ensure that all of your dependencies & tooling are regularly updated to the latest stable version as they will often include the latest functionality and security patches to known vulnerabilities

You can use the dotnet CLI to list any known deprecated  dependencies you may have inside your project or solution. You can use the command `dotnet list package --deprecated` to provide you a list of any known deprecations.