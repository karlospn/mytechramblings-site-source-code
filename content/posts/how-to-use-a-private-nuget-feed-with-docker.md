---
title: "How to Restore a Nuget from an Azure DevOps Private Feed within Docker"
date: 2020-07-14T18:12:41+02:00
tags: ["csharp", "docker", "nuget", "containers", "windows"]
draft: true
---

Let's go for a quick one this time. 

If you try to create a Docker image from an app that uses a nuget hosted in a private Azure DevOps feed I'm pretty confident that some of these errors are going to sound pretty familiar to you:  

```bash
/src/MyConsoleApplication.csproj : error NU1101: Unable to find package MyOwn.EmailService. No packages exist with this id in source(s): nuget.org
  Failed to restore /src/MyConsoleApplication.csproj (in 2.37 sec).

```

```bash
C:\Program Files\dotnet\sdk\3.1.301\NuGet.targets(128,5): error : Unable to load the service index for source https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json. [C:\src\MyConsoleApplication.csproj]

```

```bash
C:\Program Files\dotnet\sdk\3.1.301\NuGet.targets(128,5): error :   Response status code does not indicate success: 401 (Unauthorized). [C:\src\MyConsoleApplication.csproj]
```

That's a pretty common pain point when trying to create a Docker image from an app that uses a private Azure DevOps Feed.

To solve this problem there are two possible solutions: 
- Place a NuGet.Config file inside your application and use it when creating the image.
- Use the Azure Artifacts Credentials Provider

The first option is the older one and used to be the way to do it. 
The second one is the most recent one and in my opinion a better way to do it.

Let's show somes examples, but first of all let's set everything up.

1- I create a private Azure DevOps Feed:


```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="my-very-private-feed" value="https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json" />
  </packageSources>
</configuration>

```

2- I pushed a Nuget named **"MyOwn.EmailService"** into my private Azure DevOps Feed

3- I created a .NETCore 3.1 console app that is using the nuget **"MyOwn.EmailService"**


```xml

<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="MyOwn.EmailService" Version="1.0.0" />
  </ItemGroup>

</Project>

```

4 - I built a DockerFile

```docker

FROM mcr.microsoft.com/dotnet/core/runtime:3.1-buster-slim AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build

WORKDIR /src
COPY ["MyConsoleApplication.csproj", ""]
RUN dotnet restore "./MyConsoleApplication.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyConsoleApplication.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsoleApplication.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsoleApplication.dll"]

```

And when I try to create the image it fails miserably with a 401 error. Let's fix it.


## Scenario 1 : Using a NuGet.Config
---

Create a NuGet.Config and place it within the app.  
You also need to create a PAT _(Personal Access Token)_ to authenticate against the private feed.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="my-very-private-feed" value="https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <my-very-private-feed>
      <add key="Username" value="my-user@outlook.com" />
      <add key="ClearTextPassword" value="put-your-pat-here" />
    </my-very-private-feed>
  </packageSourceCredentials>
</configuration>

```

Change the Dockerfile to use the NuGet.Config file when trying to restore the nugets:

```docker

FROM mcr.microsoft.com/dotnet/core/runtime:3.1-buster-slim AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build

WORKDIR /src
COPY ["MyConsoleApplication.csproj", ""]
COPY NuGet.Config ./

RUN dotnet restore --configfile NuGet.Config "./MyConsoleApplication.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyConsoleApplication.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsoleApplication.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsoleApplication.dll"]

```

And it works flawlessly.


## Scenario 2: Using the Artifacts Credential Provider
---

The Azure Artifacts Credentials provider eases the acquisition of credentials needed to restore NuGet packages as part of your .NET development workflow.  
You can read more about it [here](https://github.com/microsoft/artifacts-credprovider)


The Dockerfile looks like this:


```docker

FROM mcr.microsoft.com/dotnet/core/runtime:3.1-buster-slim AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build

ENV DOTNET_SYSTEM_NET_HTTP_USESOCKETSHTTPHANDLER=0
ENV NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED true
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS {\"endpointCredentials\": [{\"endpoint\":\"https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json\", \"username\":\"my-user@outlook.com\", \"password\":\"put-your-pat-here\"}]}


RUN wget -qO- https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | bash

WORKDIR /src
COPY ["MyConsoleApplication.csproj", ""]

RUN dotnet restore "./MyConsoleApplication.csproj" -s "https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json" -s "https://api.nuget.org/v3/index.json"
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyConsoleApplication.csproj"  -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsoleApplication.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsoleApplication.dll"]

```

In that scenario we are using a bash script to install the credential provider, and  also need to specify some environment variables for the provider to work. 

- NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED: Controls whether or not the session token is saved to disk. If false, the Credential Provider will prompt for auth every time.
- VSS_NUGET_EXTERNAL_FEED_ENDPOINTS: A JSON that contains an array of service endpoints, usernames and access tokens to authenticate. 

Also In the _"dotnet restore"_ step we need to specify our private feed. If we don't specify it  the credential provider will try to restore the nugets using only the default feed.



## Windows Containers
---


And what about the "long forgotten" **Windows Containers?**   
Let's try to replicate the same scenarios but these time using Windows Containers.



### Scenario 1 : Using the Nuget.Config with a nanoserver image
---

The only difference is that I'm using a windows nanoserver image instead of a linux image.

```docker
FROM mcr.microsoft.com/dotnet/core/runtime:3.1-nanoserver-1903 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-nanoserver-1903 AS build
WORKDIR /src
COPY ["MyConsoleApplication.csproj", ""]
COPY NuGet.Config ./

RUN dotnet restore --configfile NuGet.Config "./MyConsoleApplication.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyConsoleApplication.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsoleApplication.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsoleApplication.dll"]

```
I works without any hitch.

### Scenario 2: Using the Artifacts Credential Provider

That scenario is a real pain!  

We are running on windows containers so we cannot use the bash script to install the provider, instead of using the bash script we are going to use another script written in Powershell.   
That's not a problem at all because the nanoserver image comes with Powershell Core already installed.  
 
The problem is that the credential provider installer doesn't play nice with the nanoserver image. The solution is to install the credentials provider in a **windows server core** image and copy the output into the nanoserver image...   

```docker

# escape=`

# Install the cred provider in a separate stage based on Windows Server Core
# due to https://github.com/microsoft/artifacts-credprovider/issues/201
FROM mcr.microsoft.com/powershell as credproviderinstaller
SHELL ["pwsh", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
RUN Invoke-WebRequest https://raw.githubusercontent.com/microsoft/artifacts-credprovider/master/helpers/installcredprovider.ps1 -OutFile installcredprovider.ps1; `
    .\installcredprovider.ps1; `
    del installcredprovider.ps1


FROM mcr.microsoft.com/dotnet/core/sdk:3.1-nanoserver-1903 AS build
ENV DOTNET_SYSTEM_NET_HTTP_USESOCKETSHTTPHANDLER=0
ENV NUGET_CREDENTIALPROVIDER_SESSIONTOKENCACHE_ENABLED true
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS `
    "{`"endpointCredentials`": [{`"endpoint`":`"https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json`", `"username`":`"myUser@outlook.com`", `"password`":`"putYourPatHere`"}]}"


# Copy cred provider from installer stage
COPY --from=credproviderinstaller C:\Users\ContainerAdministrator\.nuget\plugins C:\Users\ContainerUser\.nuget\plugins

WORKDIR /src
COPY ["MyConsoleApplication.csproj", ""]
RUN dotnet restore "./MyConsoleApplication.csproj" -s "https://pkgs.dev.azure.com/cpn/_packaging/my-very-private-feed/nuget/v3/index.json" -s "https://api.nuget.org/v3/index.json" 
COPY . .
WORKDIR "/src/."
RUN dotnet build "MyConsoleApplication.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsoleApplication.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/core/runtime:3.1-nanoserver-1903 AS base
WORKDIR /app

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsoleApplication.dll"]

```

At the end of the day, I prefer to use the Credential Provider instead of the Nuget.Config because it one less file to manage in my repository. 
 
Also if you are a men with poor luck and stumble with a project that uses Windows Containers you can also use both scenarios.  
 Probably you're going to lose some time tinkering around if you want to use the credential provider with a windows container but at nonetheless it works.