---
title: "Linting a .NET 6 app dockerfile using Hadolint, dockerfile_lint and Azure Pipelines"
date: 2022-05-10T11:26:01+02:00
draft: true
tags: ["dotnet", "csharp", "devops", "containers", "docker"]
description: "Like any other language, Dockerfiles can and should be linted for updated best practices and code quality checks. Docker is no exception to the rule, because good practices are always moving and getting new updates. In this post I will be covering how you can integrate them on your CI/CD pipelines."
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/linting-a-dockerfile-net6-app-with-azure-pipelines).

A few months back I wrote a post about image scanning security and how important it is in a Secure DevOps workflow, also I showed you how you could use some of the most well-known image scanners alongside with your Azure DevOps CI/CD YAML Pipelines.    
If you're interested, the post is right [here](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/)

Another important step, that is skipped quite frequently, is linting the application Dockerfile. 

Like any other language, Dockerfiles can and should be linted for updated best practices and
code quality checks.   
Docker is no exception to the rule, and good practices are always moving and getting new updates.

A Dockerfile linter is a tool that analyses and parses the Dockerfile and warns when it doesn’t match best practices or guidelines. This gives us an automated way of helping developers to write Dockerfiles which always meet a reasonable standard.   

Incorporating a linter into our Secure DevOps workflow ensures our Dockerfiles are always readable, understandable and maintainable.

Using a Dockerfile linter is really easy and in this post I will be covering how you can integrate them on your CI/CD pipelines.   

To show you how to use them I'll be using Azure DevOps Pipelines, but they can be integrated with whatever CI/CD tool you use (Github Actions, Bitbucket Pipelines, Jenkins, ...)

In my projects I tend to use those 2 linters:

- Hadolint
- dockerfile_lint

More info about them and why I use two linters instead of one in the next sections.

# Hadolint

Hadolint (https://github.com/hadolint/hadolint) is probably the most popular and used Dockerfile linter right now.   
It validates that your Dockerfile is following [Docker best practice](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). 

Here's a [complete list](https://github.com/hadolint/hadolint#rules) of the rules that Hadolint validates.

## How to use it

To run it locally:

```bash
hadolint <Dockerfile>
hadolint --ignore DL3003 --ignore DL3006 <Dockerfile> # exclude specific rules
```
To run it using the official docker image you just need to pipe your Dockerfile to the docker run command, like this:

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
```

Also if you want to override the severity or ignore specific rules, you can do it using a config file.

```yaml
ignored:
  - SC1010

override:
  error:
    - DL3001
    - DL3002
  warning:
    - DL3042
    - DL3033
  info:
    - DL3032
  style:
    - DL3015
```
To pass a config file (using relative or absolute path) to  the hadolint container use the following command:

```yaml
docker run --rm -i -v /your/path/to/hadolint.yaml:/.config/hadolint.yaml hadolint/hadolint < Dockerfile
# OR
docker run --rm -i -v /your/path/to/hadolint.yaml:/.config/hadolint.yaml ghcr.io/hadolint/hadolint < Dockerfile
```
In addition to config files, Hadolint can also be configured with environment variables.

```bash
NO_COLOR=1                               # Set or unset. See https://no-color.org
HADOLINT_NOFAIL=1                        # Truthy value e.g. 1, true or yes
HADOLINT_VERBOSE=1                       # Truthy value e.g. 1, true or yes
HADOLINT_FORMAT=json                     # Output format (tty | json | checkstyle | codeclimate | gitlab_codeclimate | gnu | codacy | sarif )
HADOLINT_FAILURE_THRESHOLD=info          # threshold level (error | warning | info | style | ignore | none)
HADOLINT_OVERRIDE_ERROR=DL3010,DL3020    # comma separated list of rule codes
HADOLINT_OVERRIDE_WARNING=DL3010,DL3020  # comma separated list of rule codes
HADOLINT_OVERRIDE_INFO=DL3010,DL3020     # comma separated list of rule codes
HADOLINT_OVERRIDE_STYLE=DL3010,DL3020    # comma separated list of rule codes
HADOLINT_IGNORE=DL3010,DL3020            # comma separated list of rule codes
HADOLINT_STRICT_LABELS=1                 # Truthy value e.g. 1, true or yes
HADOLINT_DISABLE_IGNORE_PRAGMA=1         # Truthy value e.g. 1, true or yes
HADOLINT_TRUSTED_REGISTRIES=docker.io    # comma separated list of registry urls
HADOLINT_REQUIRE_LABELS=maintainer:text  # comma separated list of label schema items
```

## Integrate it with Azure Pipelines

The easiest way to integrate it with Azure Pipelines is using the docker image and pipe the app Dockerfile.

```yaml
trigger: none

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Bash@3
  displayName: Run hadolint linter
  inputs:
    targetType: 'inline'
    script: |
      docker run --rm -i --env HADOLINT_FAILURE_THRESHOLD=warning hadolint/hadolint:latest < Dockerfile
    workingDirectory: '$(System.DefaultWorkingDirectory)'
```


# dockerfile_lint

Hadolint already validates that your Dockerfile follows the Docker best practices, so why you need another linter?

dockerfile_lint (https://github.com/projectatomic/dockerfile_lint) is a rule based semantic linter. The linter rules can be used to check file syntax as well as arbitrary semantic and best practices attributes determined by the rule file writer.

dockerfile_lint can validate that the application is configured properly on the Dockerfile.   

For example, if someone is writing a .NET app I want to validate that the base images used in the dockerfile are coming from the official Microsoft registry (mcr.microsoft.com) and is not using some unofficial image from docker hub or another place.

That's why Hadolint and dockerfile_lint are a pretty good match, the first one validates that the Dockerfile is following the best practices, and the second one validates that the app is properly setup using a syntactic analisys.

One really important thing that you need to know about dockerfile_lint is that is somewhat abandoned by Red Hat, they killed it when Project Atomic was abandoned. As it is right now it works great, but do not expect any new releases or bug fixes.

## How to use it

To run it locally:

```bash
npm install -g dockerfile_lint
dockerfile_lint -f <Dockerfile>  -r <linting_rules>.yml
```

To run it from the official docker image:

```bash
docker run -it --rm -v $PWD:/root/ \
           projectatomic/dockerfile-lint \
           dockerfile_lint [-f Dockerfile]
```

Rule files are written in YAML and implememented using regular expressions, the linter runs on one instruction of the dockerfile at a time. The rule file has 4 sections, a profile section, a general section, a line rule section and a required instruction section.

More info about how to write rules [here](https://github.com/projectatomic/dockerfile_lint#extending-and-customizing-rule-files) and also a few examples [here](https://github.com/projectatomic/dockerfile_lint/tree/master/sample_rules).


#  Practical example

In the previous section I talked a little bit about Hadolint and dockerfile_lint. 

Now, let's me show you a practical example, I'll be linting a .NET6 app Dockerfile that has some issues and end up with a fully double-checked linted file.

The next code snippet shows dockerfile that I'm going to lint.

```yaml
#############
## Stage 1 ##
#############
FROM bitnami/dotnet-sdk:latest AS build
WORKDIR /app

## Specify maintainer
MAINTAINER mytechramblings.com

## Copy the applications .csproj
COPY /src/WebApp/*.csproj ./src/WebApp/

## Restore packages
RUN dotnet restore "./src/WebApp/WebApp.csproj" \
	-s "https://api.nuget.org/v3/index.json" \
	--runtime rhel-x64

## Copy everything else
COPY . ./

## Build the app
RUN dotnet build "./src/WebApp/WebApp.csproj" \
	-c Release \
	--runtime rhel-x64 \
	--self-contained true \
	/p:PublishSingleFile=true

## Run unit tests
RUN dotnet test "./test/WebApp.Tests/WebApp.Tests.csproj" \
	--no-restore

## Publish the app
RUN dotnet publish "./src/WebApp/WebApp.csproj" \
	-c Release \
	-o /app/publish \
	--runtime rhel-x64 \
	--self-contained true \
	/p:PublishSingleFile=true

#############
## Stage 2 ##
#############
FROM mcr.microsoft.com/dotnet/runtime-deps:6.0-bullseye-slim
WORKDIR /app

## Copy artifact
COPY --from=build /app/publish .

## Set entrypoint
ENTRYPOINT ["./WebApp"]
```
It is a multi-stage dockerfile. 

- Stage one restores, builds and publishes the application artifact. The application is setup to be published as a single file.

- Stage two grabs the artifact from stage one and defines the entrypoint.

### **1. Execute the Hadolint linter**

### **2. Create the dockerfile_lint rules**

First step is to create the dockerfile_lint linter rules file. 

The Dockerfile I showed you above creates a totally working image that can be executed without any problem at all. But nonetheless I want to enforce a series of validations, because having a running image doesn't mean that it is a compliant one with my company requirements.

Those are the rules I want to enforce on any Dockerfile.

- Do not use the "latest" tag on any base image declared on the Dockerfile.
- You need to specify a tag for any base images declared on the Dockerfile.
- The base images should come from the official Microsoft repository (mcr.microsoft.com)
- You should restore the NuGet packages explicitly using the ``dotnet restore`` command, that means that there is not need for ``dotnet build``, ``dotnet test`` and  ``dotnet publish``  to restore them again.
- We want the artifact size to be as small as possible.
- We want to publish the app as a single file executable.
- We want to publish the app as a self-contained application. 
- The app will be running on a machine running Debian, so we want to use a ``linux-x64`` runtime when generating the artifact.
- There should be at least one ``EXPOSE``, ``ENTRYPOINT`` and ``COPY`` command.

The next code snippet shows the resulting rules file:

```yaml
profile:
  name: "WebApp"
  description: "Linting profile for WebApp application. Checks dockerfile semantically."
line_rules:
  FROM:
    paramSyntaxRegex: /.+/
    rules:
      -
        label: "is_latest_tag"
        regex: /latest/
        level: "info"
        message: "Base image uses 'latest' tag"
        description: "Using the 'latest' tag may cause unpredictable builds. It is recommended that a specific tag is used in the FROM line"
        reference_url:
          - "https://docs.docker.com/engine/reference/builder/"
          - "#from"
      -
        label: "no_tag"
        regex: /[:]/
        level: "warn"
        inverse_rule: true
        message: "No tag is used"
        description: "No tag is used"
        reference_url:
          - "https://docs.docker.com/engine/reference/builder/"
          - "#from"
      -
        label: "using_mcr_official_repository"
        regex: /mcr.microsoft.com/
        inverse_rule: true
        level: "error"
        message: "Base Image must be from the official Microsoft registry"
        description: "The Official .NET Docker images are Docker images created and optimized by Microsoft. They are publicly available in the Microsoft MCR repository"
        reference_url:
          - "https://github.com/microsoft/containerregistry"
  RUN: 
    paramSyntaxRegex: /.+/
    rules: 
      - 
        label: "use_no_restore_flag_when_running_dotnet_build"
        regex: /dotnet build(?!.+--no-restore)/g
        level: "error"
        message: "When using dotnet build you must use the no-restore flag"
        description: "The --no-restore option is used to disable implicit restore, when using the dotnet restore command you don't need to restore the packages a second time" 
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-build"
      - 
        label: "use_no_restore_flag_when_running_dotnet_test"
        regex: /dotnet test(?!.+--no-restore)/g
        level: "error"
        message: "When using dotnet test you must use the no-restore flag"
        description: "The --no-restore option is used to disable implicit restore, when using the dotnet restore command you don't need to restore the packages a second time"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-build"
      - 
        label: "use_no_restore_flag_when_running_dotnet_publish"
        regex: /dotnet publish(?!.+--no-restore)/g
        level: "error"
        message: "When running dotnet publish you must use the no-restore flag"
        description: "The --no-restore option is used to disable implicit restore, when using the dotnet restore command you don't need to restore the packages a second time"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish"
      - 
        label: "use_no_build_flag_when_running_dotnet_publish"
        regex: /dotnet publish(?!.+--no-build)/g
        level: "error"
        message: "When running dotnet publish you must use the no-build flag"
        description: "The --no-restore option is used to disable implicit restore, when using the dotnet restore command you don't need to restore the packages a second time"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish"
      - 
        label: "no_build_without_single_executable"
        regex: /dotnet build(?!.+PublishSingleFile=true)/g
        level: "error"
        message: "Application artifact must be published as a single self contained executable"
        description: "Bundling all application-dependent files into a single binary provides an application developer with the attractive option to deploy and distribute the application as a single file"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/single-file"
      - 
        label: "no_publish_without_single_executable"
        regex: /dotnet publish(?!.+PublishSingleFile=true)/g
        level: "error"
        message: "Application artifact must be published as a single self contained executable"
        description: "Bundling all application-dependent files into a single binary provides an application developer with the attractive option to deploy and distribute the application as a single file"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/single-file"
      - 
        label: "no_build_without_self_contained"
        regex: /dotnet build(?!.+self-contained true)/g
        level: "error"
        message: "The application must be published as a self contained artifact"
        description: "Publishing your app as self-contained produces a platform-specific executable. The output publishing folder contains all components of the app, including the .NET libraries and target runtime"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/"
      - 
        label: "no_publish_without_self_contained"
        regex: /dotnet publish(?!.+self-contained true)/g
        level: "error"
        message: "The application must be published as a self contained artifact"
        description: "Publishing your app as self-contained produces a platform-specific executable. The output publishing folder contains all components of the app, including the .NET libraries and target runtime"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/"
      - 
        label: "no_publish_without_trimming"
        regex: /dotnet publish(?!.+PublishTrimmed=true)/g
        level: "error"
        message: "Application artifact must be trimmed when published"
        description: "The trim-self-contained deployment model is a specialized version of the self-contained deployment model that is optimized to reduce deployment size"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/trim-self-contained"   
      - 
        label: "no_restore_without_using_linux_x64_runtime"
        regex: /dotnet restore(?!.+runtime linux-x64)/g
        level: "error"
        message: "The application must be publish as a linux-64 platform-specific artifact"
        description: "The runtime attribute is used to identify the target platforms where the application runs."
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/rid-catalog"   
      - 
        label: "no_build_without_using_linux_x64_runtime"
        regex: /dotnet build(?!.+runtime linux-x64)/g
        level: "error"
        message: "The application must be publish as a linux-64 platform-specific artifact"
        description: "The runtime attribute is used to identify the target platforms where the application runs."
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/rid-catalog" 
      - 
        label: "no_publish_without_using_linux_x64_runtime"
        regex: /dotnet publish(?!.+runtime linux-x64)/g
        level: "error"
        message: "The application must be publish as a linux-64 platform-specific artifact"
        description: "The runtime attribute is used to identify the target platforms where the application runs."
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/rid-catalog"     
required_instructions: 
    - 
      instruction: "EXPOSE"
      count: 1
      level: "error"
      message: "There is no 'EXPOSE' instruction"
      description: "Without exposed ports how will the service of the container be accessed?"
      reference_url: 
        - "https://docs.docker.com/engine/reference/builder/"
        - "#expose"
    - 
      instruction: "ENTRYPOINT"
      count: 1
      level: "error"
      message: "There is no 'ENTRYPOINT' instruction"
      description: "None"
      reference_url: 
        - "https://docs.docker.com/engine/reference/builder/"
        - "#entrypoint"
    - 
      instruction: "COPY"
      count: 1
      level: "error"
      message: "There is no 'COPY' instruction"
      description: "None"
      reference_url: 
        - "https://docs.docker.com/engine/reference/builder/"
        - "#copy"
```
### **3. Execute the dockerfile_lint linter**



# Linting a dockerfile using Azure Pipelines 

```yaml
trigger:
  branches:
    include:
    - main
  paths:
    exclude:
    - pipelines/*
    - test/*
    - README.md
    - .dockerignore
    - .gitignore

variables:
- name: appName
  value: 'WebApp'
- name: tag
  value: '$(Build.BuildId)' 


pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Bash@3
  displayName: Run hadolint linter
  inputs:
    targetType: 'inline'
    script: |
      docker run --rm -i --env HADOLINT_FAILURE_THRESHOLD=warning hadolint/hadolint:latest < Dockerfile
    workingDirectory: '$(System.DefaultWorkingDirectory)'

- task: Bash@3
  displayName: Install semantic linter
  inputs:
    targetType: 'inline'
    script: |
      npm install -g dockerfile_lint
- task: Bash@3
  displayName: Run semantic linter
  inputs:
    targetType: 'inline'
    script: |
      dockerfile_lint -f Dockerfile  -r pipelines/linting_rules.yml
    workingDirectory: '$(System.DefaultWorkingDirectory)'

- task: Bash@3
  displayName: Create application image
  inputs:
    targetType: 'inline'
    script: |
      docker build -t  ${{ lower(variables.appName) }}:$(tag) .
    workingDirectory: '$(System.DefaultWorkingDirectory)'

- task: Bash@3
  displayName: Download Trivy Scanner
  inputs:
    targetType: 'inline'
    script: |
      curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
- task: Bash@3
  displayName: Run security scanner
  inputs:
    targetType: 'inline'
    script: |
      trivy image --exit-code 0 --severity LOW,MEDIUM,HIGH --no-progress ${{ lower(variables.appName) }}:$(tag)
      trivy image --exit-code 1 --severity CRITICAL --no-progress ${{ lower(variables.appName) }}:$(tag)
```