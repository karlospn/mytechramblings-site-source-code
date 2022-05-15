---
title: "Linting a .NET 6 app Dockerfile using Hadolint, dockerfile_lint and Azure Pipelines"
date: 2022-05-10T11:26:01+02:00
draft: true
tags: ["dotnet", "csharp", "devops", "containers", "docker"]
description: "Like any other language, Dockerfiles can and should be linted for updated best practices and code quality checks. In this post I will show you how to incorporate a couple of Dockerfile linters into our Secure DevOps workflow to ensure our Dockerfiles are always readable, understandable and maintainable."
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/linting-a-dockerfile-net6-app-with-azure-pipelines).

A few months back I wrote a post about security image scanning and how important it is in a Secure DevOps workflow, also I showed you how you could use some of the most well-known image scanners alongside with your Azure DevOps CI/CD YAML Pipelines.    
If you're interested, the post is right [here](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/)

Another important step, that is skipped quite frequently, is linting the application Dockerfile. 

Like any other language, Dockerfiles can and should be linted for updated best practices and
code quality checks.   
Docker is no exception to the rule, and good practices are always moving and getting new updates.

A Dockerfile linter is a tool that analyses and parses the Dockerfile and warns when it doesn’t match best practices or guidelines. This gives us an automated way of helping developers to write Dockerfiles which always meet a reasonable standard.   

Incorporating a linter into our Secure DevOps workflow ensures our Dockerfiles are always readable, understandable and maintainable.

In this post I will be covering how you can use them and also how you can integrate them on your CI/CD pipelines.   

To show you how to integrate them with a CI/CD pipeline I'll be using Azure DevOps Pipelines, but the process is practically the same is you want to integrate the with whatever CI/CD tool you use (Github Actions, Bitbucket Pipelines, Jenkins, ...).

In this post I'll focus on those 2 linters:

- Hadolint (https://github.com/hadolint/hadolint)
- dockerfile_lint (https://github.com/projectatomic/dockerfile_lint)

More info about them and why I use two linters instead of a single one in the next sections.

# Hadolint

Hadolint is probably the most popular and used Dockerfile linter right now, it validates that your Dockerfile is following [Docker best practice](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). 

Here's a [complete list](https://github.com/hadolint/hadolint#rules) of the rules that Hadolint validates.

## How to use it

To run it locally:

```bash
hadolint <Dockerfile>
hadolint --ignore DL3003 --ignore DL3006 <Dockerfile> # exclude specific rules
```
To run it using the official docker image, you just need to pipe your Dockerfile to the docker run command, like this:

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

dockerfile_lint is a rule based semantic linter. The linter rules can be used to check file syntax as well as arbitrary semantic and best practices attributes determined by the rule file writer.

I tend to use dockerfile_lint to validate that the application is configured properly on the Dockerfile.   
For example, if someone is writing a .NET app Dockerfile I want to validate that the base images used are coming from the official Microsoft registry (mcr.microsoft.com) and is not using some unofficial images from docker hub or somewhere else.

That's why Hadolint and dockerfile_lint are a pretty good match, the first one validates that the Dockerfile is following the best practices, and the second one validates that the app is properly setup using a syntactic analisys.

One **really important** thing that you need to know about dockerfile_lint is that is somewhat abandoned by Red Hat, they killed it when Project Atomic was abandoned. As it is right now, it works good enough, but do not expect any new releases or bug fixes.

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

Rule files are written in YAML and implememented using regular expressions, the linter runs one instruction of the dockerfile at a time.    
The rule file has 4 sections, a profile section, a general section, a line rule section and a required instruction section.

- More info about how to write rules [here](https://github.com/projectatomic/dockerfile_lint#extending-and-customizing-rule-files).
- A few rules examples [here](https://github.com/projectatomic/dockerfile_lint/tree/master/sample_rules).

## Integrate it with Azure Pipelines

The easiest way to integrate it with Azure Pipelines is using NPM to install it and then just run it.

```yaml
trigger: none

pool:
  vmImage: 'ubuntu-latest'

steps:
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
```

# Practical example

In the previous section I gave you a brief introduction about how to use both linters: Hadolint and dockerfile_lint. 

Now, let's make a practical example. I have a .NET6 app Dockerfile that has some issues, I'm going to run both linters, fix every linting error and end up with a fully double-checked linted file.

The next code snippet shows the Dockerfile that I'm going to lint.

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

- Stage one restores, builds and generates the app artifact. As you can see the app is setup to be published as a single file executable.

- Stage two grabs the app artifact from the previous stage and sets the entrypoint.

## **1. Executing the Hadolint linter**

The first step is executing Hadolint to check if the Dockerfile follows Docker best practices.   
After running it, this is the output:

```bash
-:4 DL3007 warning: Using latest is prone to errors if the image will ever update. Pin the version explicitly to a release tag
-:8 DL4000 error: MAINTAINER is deprecated
-:29 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
-:33 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
```

Using the latest tag or no tag at all is not a good practice, to solve the ``DL3007`` issue I'm going to change:
- The ``FROM bitnami/dotnet-sdk:latest`` command to ``FROM bitnami/dotnet-sdk:6``

The ``MAINTAINER`` instruction is used to define the author of the generated images, but this instruction is deprecated. The ``LABEL`` instruction is a much more flexible version of this and you should use it instead. To solve the ``DL4000``  issue I'm going to change:
- The ``MAINTAINER mytechramblings.com`` instruction to ``LABEL maintaner=mytechramblings.com``

The ``DL3059`` rule violation is a false positive error, let me explain why.   
Each ``RUN`` instruction creates a new layer in the resulting image. Therefore consolidating consecutive ``RUN`` instructions reduces the layer count of an image.   
In our case this is a multi-stage Dockerfile, so we don't care how many layers are being created on stage one, this is mainly because the stage one image is going to be discarded and the only thing that we're going to reuse from that stage is the application artifact.  
To solve this rule I'm going to tell Hadolint to ignore it, like this:
- ``docker run --rm -i --env HADOLINT_IGNORE=DL3059 hadolint/hadolint < Dockerfile``

After solving those 3 errors Hadolint returns no error at all.

## **2. Creating the dockerfile_lint rules**

The next step is executing dockefile_lint, but first we need to create a rules file. 

The Dockerfile I showed you above creates an image that runs just fine, but it doesn't mean that the application is setup properly or that the Dockerfile is compliant with some of my own or my company constraints.

To keep this example going I made up a few rules that I want to enforce on my Dockerfile, here's the complete list:

- The base images should come from the official Microsoft repository (mcr.microsoft.com)
- You should restore the NuGet packages explicitly using the ``dotnet restore`` command, that means that there is not need for ``dotnet build``, ``dotnet test`` and  ``dotnet publish``  to restore them again.
- The artifact size should be as small as possible.
- The app must be published as a single file executable.
- The app must be published as a self-contained application. 
- The app will be running on a machine running Debian, that means that the ``linux-x64`` runtime needs to be set when generating the app artifact.
- There should be at least one ``EXPOSE``, ``ENTRYPOINT`` and ``COPY`` command on the Dockerfile.

The next code snippet shows the resulting rules file that I'll be using to enforce the above list.

```yaml
profile:
  name: "WebApp"
  description: "Linting profile for WebApp application. Checks dockerfile semantically."
line_rules:
  FROM:
    paramSyntaxRegex: /.+/
    rules:
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
        label: "use_no_build_flag_when_running_dotnet_publish"
        regex: /dotnet publish(?!.+--no-build)/g
        level: "error"
        message: "When running dotnet publish you must use the no-build flag"
        description: "The --no-build is used to avoid building the project before publishing. It also implicitly sets the --no-restore flag"
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
        description: "The runtime attribute is used to identify the target platforms where the application runs"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/rid-catalog"   
      - 
        label: "no_build_without_using_linux_x64_runtime"
        regex: /dotnet build(?!.+runtime linux-x64)/g
        level: "error"
        message: "The application must be publish as a linux-64 platform-specific artifact"
        description: "The runtime attribute is used to identify the target platforms where the application runs"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/rid-catalog" 
      - 
        label: "no_publish_without_using_linux_x64_runtime"
        regex: /dotnet publish(?!.+runtime linux-x64)/g
        level: "error"
        message: "The application must be publish as a linux-64 platform-specific artifact"
        description: "The runtime attribute is used to identify the target platforms where the application runs"
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
## **3. Executing the dockerfile_lint linter**

After running it using the rules list from the previous section, this is the output:

```bash
# Analyzing Dockerfile

--------ERRORS---------

Line 7: -> FROM bitnami/dotnet-sdk:6 AS build
ERROR: Base Image must be from the official Microsoft registry. The Official .NET Docker images are Docker images created and optimized by Microsoft. They are publicly available in the Microsoft MCR repository.
Reference -> https://github.com/microsoft/containerregistry


Line 27: -> RUN dotnet restore "./src/WebApp/WebApp.csproj"     -s "https://api.nuget.org/v3/index.json"        --runtime rhel-x64
ERROR: The application must be publish as a linux-64 platform-specific artifact. The runtime attribute is used to identify the target platforms where the application runs.
Reference -> https://docs.microsoft.com/en-us/dotnet/core/rid-catalog


Line 43: -> RUN dotnet build "./src/WebApp/WebApp.csproj"       -c Release      --runtime rhel-x64      --self-contained true   /p:PublishSingleFile=true
ERROR: When using dotnet build you must use the no-restore flag. The --no-restore option is used to disable implicit restore, when using the dotnet restore command you do not need to restore the packages a second time.
Reference -> https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-build


Line 43: -> RUN dotnet build "./src/WebApp/WebApp.csproj"       -c Release      --runtime rhel-x64      --self-contained true   /p:PublishSingleFile=true
ERROR: The application must be publish as a linux-64 platform-specific artifact. The runtime attribute is used to identify the target platforms where the application runs.
Reference -> https://docs.microsoft.com/en-us/dotnet/core/rid-catalog


Line 65: -> RUN dotnet publish "./src/WebApp/WebApp.csproj"     -c Release      -o /app/publish         --runtime rhel-x64      --self-contained true   /p:PublishSingleFile=true
ERROR: When running dotnet publish you must use the no-build flag. The --no-build is used to avoid building the project before publishing. It also implicitly sets the --no-restore flag.
Reference -> https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish


Line 65: -> RUN dotnet publish "./src/WebApp/WebApp.csproj"     -c Release      -o /app/publish         --runtime rhel-x64      --self-contained true   /p:PublishSingleFile=true
ERROR: Application artifact must be trimmed when published. The trim-self-contained deployment model is a specialized version of the self-contained deployment model that is optimized to reduce deployment size.

Reference -> https://docs.microsoft.com/en-us/dotnet/core/deploying/trim-self-contained


Line 65: -> RUN dotnet publish "./src/WebApp/WebApp.csproj"     -c Release      -o /app/publish         --runtime rhel-x64      --self-contained true   /p:PublishSingleFile=true
ERROR: The application must be publish as a linux-64 platform-specific artifact. The runtime attribute is used to identify the target platforms where the application runs.
Reference -> https://docs.microsoft.com/en-us/dotnet/core/rid-catalog


ERROR: There is no 'EXPOSE' instruction. Without exposed ports how will the service of the container be accessed?.
Reference -> https://docs.docker.com/engine/reference/builder/#expose
```

As I said earlier, the Dockerfile I showed you at the beginning of this example creates an image that runs just fine, but as you can see having a working and running image doesn't mean that the Dockerfile is free of problems or the application is tuned up correctly.

Let's start fixing the errors:

- The base image should come from the official Microsoft repository, which means changing this instruction from ``FROM bitnami/dotnet-sdk:6 AS build`` to ``FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build``

- The application must use a ``linux-x64`` runtime instead of the ``rhel-x64`` runtime, which means changing the runtime attribute on the ``dotnet restore``, ``dotnet build`` and ``dotnet publish`` commands.

- There is no need to restore the NuGet packages on each dotnet command, which means setting the ``--no-restore`` attribute on the ``dotnet build`` command. 

- There is no need to restore the NuGet packages and build the project again when running the ``dotnet publish`` command, which means setting the ``--no-build`` attribute on the ``dotnet publish`` command.

- One strategy to reduce the size of the artifact is to trim the artifact. To do it we need to set the ``/p:PublishTrimmed=true`` attribute when running the ``dotnet publish`` command.

- It might not be needed, but it is always a good practice to ``EXPOSE`` which ports are going to be used, so on stage 2 we're going to add the ``EXPOSE 80`` and ``EXPOSE 443`` instructions.

Here's the resulting Dockerfile after applying all this fixes.

```yaml
#############
## Stage 1 ##
#############
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

## Set the maintainer
LABEL maintaner=mytechramblings.com

## Copy the applications .csproj
COPY /src/WebApp/*.csproj ./src/WebApp/

## Restore packages
RUN dotnet restore "./src/WebApp/WebApp.csproj" \
	-s "https://api.nuget.org/v3/index.json" \
	--runtime linux-x64

## Copy everything else
COPY . ./

## Build the app
RUN dotnet build "./src/WebApp/WebApp.csproj" \
	-c Release \
	--runtime linux-x64 \
	--no-restore \
	--self-contained true \
	/p:PublishSingleFile=true

## Run unit tests
RUN dotnet test "./test/WebApp.Tests/WebApp.Tests.csproj" \
	--no-restore

## Publish the app
RUN dotnet publish "./src/WebApp/WebApp.csproj" \
	-c Release \
	-o /app/publish \
	--runtime linux-x64 \
	--no-restore \
	--no-build \
	--self-contained true \
	/p:PublishSingleFile=true \
	/p:PublishTrimmed=true

#############
## Stage 2 ##
#############
FROM mcr.microsoft.com/dotnet/runtime-deps:6.0-bullseye-slim
WORKDIR /app

## Expose ports
EXPOSE 80
EXPOSE 443

## Copy artifact
COPY --from=build /app/publish .

## Set entrypoint
ENTRYPOINT ["./WebApp"]
```

If we try to execute dockerfile_lint again, there are no errors.

```bash
# Analyzing Dockerfile

Check passed!
```

# Linting the Dockerfile using Azure Pipelines 

Now it is time to put everything together on an Azure DevOps CI/CD pipeline.

I'm going to put together a pipeline that not only runs the linters, but it also builds the application image if there are no linting errors and finally scans the resulting image using a security scanner.

Here's how it looks:

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
      docker run --rm -i --env HADOLINT_FAILURE_THRESHOLD=info --env HADOLINT_IGNORE=DL3059 hadolint/hadolint:latest < Dockerfile
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