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

If you don't use any tool for scanning your dependencies right now, you should know that the dotnet CLI has a NuGet dependency scan feature ready to use. 

# How to use the .NET CLI to check if your app has any vulnerable NuGet dependency

You can list any known vulnerabilities in your dependencies within your application with the `dotnet list package --vulnerable` command.

This command gets the security information from the centralized GitHub Advisory Database. This database provides two main listings of vulnerabilities:

- A CVE is Common Vulnerabilities and Exposures. This is a list of publicly disclosed computer security flaws.
- A GHSA is a GitHub Security Advisory. GitHub is a CVE Numbering Authority (CNA) and is authorized to assign CVE identification numbers.

To scan for vulnerabilities within your projects using the dotnet CLI you'll need to have installed the **.NET SDK version 5.0.200** (or higher).

The `dotnet list package --vulnerable` command only works with projects that are using the ``PackageReference `` format, you won't be able to use it if your project still uses the ``packages.config`` format.

Here's how the output of the ``dotnet list package --vulnerable`` command  looks like on an application that has multiple projects.

```bash
The following sources were used:
   https://api.nuget.org/v3/index.json

The given project `VulnerableApp.Library.Contracts` has no vulnerable packages given the current sources.
The given project `VulnerableApp.Library.Impl` has no vulnerable packages given the current sources.
The given project `VulnerableApp.Repository.Contracts` has no vulnerable packages given the current sources.
The given project `VulnerableApp.Repository.Impl` has no vulnerable packages given the current sources.
Project `VulnerableApp.Core.Extensions` has the following vulnerable packages
   [netstandard2.0]: 
   Top-level Package      Requested   Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json      12.0.3      12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

Project `VulnerableApp.WebApi` has the following vulnerable packages
   [net5.0]: 
   Top-level Package                                    Requested   Resolved   Severity   Advisory URL                                     
   > Microsoft.AspNetCore.Authentication.JwtBearer      5.0.6       5.0.6      Moderate   https://github.com/advisories/GHSA-q7cg-43mg-qp69
   > Newtonsoft.Json                                    12.0.3      12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

The given project `VulnerableApp.WebApi.IntegrationTest` has no vulnerable packages given the current sources.
The given project `VulnerableApp.Library.Impl.UnitTest` has no vulnerable packages given the current sources.
```

As you can see the ``dotnet list package --vulnerable`` command will show if there is any package that contains a vulnerability, in which version the vulnerability has been resolved, the severity of the vulnerability, and a link to the advisory for you to view.

However, the ``dotnet list package --vulnerable`` command **ONLY** checks dependencies that are directly installed on your app (top-level packages), if you are interested in seeing vulnerabilities within your dependencies you'll need to use the `--include-transitive` parameter, like this `dotnet list package --vulnerable --include-transitive`.

Let's execute the ``dotnet list package --vulnerable --include-transitive`` command on the same app than before.

```bash
The following sources were used:
   https://api.nuget.org/v3/index.json

Project `VulnerableApp.Library.Contracts` has the following vulnerable packages
   [netstandard2.0]: 
   Transitive Package      Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json       12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

Project `VulnerableApp.Library.Impl` has the following vulnerable packages
   [netstandard2.0]: 
   Transitive Package      Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json       12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

The given project `VulnerableApp.Repository.Contracts` has no vulnerable packages given the current sources.
The given project `VulnerableApp.Repository.Impl` has no vulnerable packages given the current sources.
Project `VulnerableApp.Core.Extensions` has the following vulnerable packages
   [netstandard2.0]: 
   Top-level Package      Requested   Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json      12.0.3      12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

Project `VulnerableApp.WebApi` has the following vulnerable packages
   [net5.0]: 
   Top-level Package                                    Requested   Resolved   Severity   Advisory URL                                     
   > Microsoft.AspNetCore.Authentication.JwtBearer      5.0.6       5.0.6      Moderate   https://github.com/advisories/GHSA-q7cg-43mg-qp69
   > Newtonsoft.Json                                    12.0.3      12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr

Project `VulnerableApp.WebApi.IntegrationTest` has the following vulnerable packages
   [net5.0]: 
   Transitive Package                                   Resolved   Severity   Advisory URL                                     
   > Microsoft.AspNetCore.Authentication.JwtBearer      5.0.6      Moderate   https://github.com/advisories/GHSA-q7cg-43mg-qp69
   > Newtonsoft.Json                                    12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr
   > System.Net.Http                                    4.3.0      High       https://github.com/advisories/GHSA-7jgj-8wvc-jh57
   > System.Text.RegularExpressions                     4.3.0      Moderate   https://github.com/advisories/GHSA-cmhx-cq75-c4mj

Project `VulnerableApp.Library.Impl.UnitTest` has the following vulnerable packages
   [net5.0]: 
   Transitive Package                    Resolved   Severity   Advisory URL                                     
   > Newtonsoft.Json                     12.0.3     High       https://github.com/advisories/GHSA-5crp-9r3c-p9vr
   > System.Net.Http                     4.3.0      High       https://github.com/advisories/GHSA-7jgj-8wvc-jh57
   > System.Text.RegularExpressions      4.3.0      Moderate   https://github.com/advisories/GHSA-cmhx-cq75-c4mj
```

As you can see the list of vulnerable dependencies has grown quite a bit from the previous execution, so when using this feature use **ALWAYS** the ``--use-transitive`` parameter.

# How to integrate the .NET CLI vulnerability scan feature with your CI/CD pipelines

You can check for any known NuGet vulnerability and break the pipeline execution with just a couple of lines of bash script, so there is little to no excuse for you not to do it.


# How to check if your app has outdated or deprecated NuGet versions

To ensure a secure supply chain of dependencies, you will want to ensure that all of your dependencies & tooling are regularly updated to the latest stable version as they will often include the latest functionality and security patches to known vulnerabilities

You can use the dotnet CLI to list any known deprecated  dependencies you may have inside your project or solution. You can use the command `dotnet list package --deprecated` to provide you a list of any known deprecations.