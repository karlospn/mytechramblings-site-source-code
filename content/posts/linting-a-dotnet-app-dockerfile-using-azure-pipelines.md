---
title: "Linting a .NET 6 app dockerfile using Hadolint, dockerfile_lint and Azure Pipelines"
date: 2022-05-10T11:26:01+02:00
draft: true
tags: ["dotnet", "csharp", "devops", "containers", "docker"]
---

A few months back I wrote a post about image scanning security and how important it is in a Secure DevOps workflow, also I showed you how you could use some of the most well-known image scanners alongside with your Azure DevOps CI/CD YAML Pipelines.    
If you're interested, the post is right [here](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/)

Another important step, that is skipped quite frequently, is linting the application Dockerfile. 

Like any other language, Dockerfiles can and should be linted for updated best practices and
code quality checks.   
Docker is no exception to the rule, and good practices are always moving, getting updates, and might also be a little different between communities.

A Dockerfile linter is a tool that analyses and parses the Dockerfile and warns when it doesn’t match best practices or guidelines. This gives us an automated way of helping developers to write Dockerfiles which always meet a reasonable standard.   

Incorporating a linter into our Secure DevOps workflow ensures our Dockerfiles are always readable, understandable and maintainable.

Using a Dockerfile linter in you Secure DevOps workflow is really easy and in this post I will be covering how you can use them with your Azure DevOps CI/CD Pipelines.

In my container CI/CD workflows I tend to use those 2 dockerfile linters:

- Hadolint
- dockerfile_lint

More info about them and why I use two linters instead of one in the next sections.

# Hadolint

Hadolint (https://github.com/hadolint/hadolint) is probably the most popular and used Dockerfile linter right now. 

This linter validate the Docker [best practice](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). 

- Here's a [complete list](https://github.com/hadolint/hadolint#rules) of the rules that Hadolint validates

## How to use it

To run hadolint locally:

```bash
hadolint <Dockerfile>
hadolint --ignore DL3003 --ignore DL3006 <Dockerfile> # exclude specific rules
hadolint --trusted-registry my-company.com:500 <Dockerfile> # Warn when using untrusted FROM images
```

Or using the official docker image, you just pipe your Dockerfile to docker run:

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
```

If you want to override the severity or ignore specific rules, you can do that using a config file. Like this:

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
To pass a custom configuration file (using relative or absolute path) to a container, use the following command:

```yaml
docker run --rm -i -v /your/path/to/hadolint.yaml:/.config/hadolint.yaml hadolint/hadolint < Dockerfile
# OR
docker run --rm -i -v /your/path/to/hadolint.yaml:/.config/hadolint.yaml ghcr.io/hadolint/hadolint < Dockerfile
```
In addition to config files, Hadolint can be configured with environment variables.

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

The easiest way to integrate with Azure Pipelines is using the Hadolint docker image and pass the application Dockerfile into the container.

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

dockerfile_lint (https://github.com/projectatomic/dockerfile_lint) is a rule based semantic 'linter'. The linter rules can be used to check file syntax as well as arbitrary semantic and best practice attributes determined by the rule file writer.

With dockerfile_lint I can validate that the application is configured exactly as I desire.   

For example, on a .NET 6 app I can validate that the base image used in the dockerfile are coming from the official Microsoft registry (mcr.microsoft.com) and the developer is not using another unofficial image from docker hub.

That why Hadolint and dockerfile_lint are a pretty good match. The first one validates that the Dockerfile is following the Docker best practices, and the second one validates that the app is properly setup.

One really important thing that you need to know about dockerfile_lint is that is somewhat abandoned by Red Hat, they killed it when Project Atomic was abandoned. So as it is right now it works great, but do not expect any new releases or bug fixes.



#  Example linting a .NET6 app dockerfile

In this section, we'll be linting a .NET6 Dockerfile with some issues and end up with a fully double-checked linted file.


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
        regex: /dotnet publish(?!.+--no-restore)/g
        level: "error"
        message: "When running dotnet publish you must use the no-build flag"
        description: "The --no-restore option is used to disable implicit restore, when using the dotnet restore command you don't need to restore the packages a second time"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish"
      - 
        label: "no_publish_without_trimming"
        regex: /dotnet publish(?!.+PublishTrimmed=true)/g
        level: "error"
        message: "Application artifact must be trimmed when published"
        description: "The trim-self-contained deployment model is a specialized version of the self-contained deployment model that is optimized to reduce deployment size"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/trim-self-contained"
      - 
        label: "no_publish_without_self_contained"
        regex: /dotnet publish(?!.+self-contained true)/g
        level: "error"
        message: "The application must be published as a self contained artifact"
        description: "Publishing your app as self-contained produces a platform-specific executable. The output publishing folder contains all components of the app, including the .NET libraries and target runtime"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/"
      - 
        label: "no_publish_without_single_executable"
        regex: /dotnet publish(?!.+PublishSingleFile=true)/g
        level: "error"
        message: "Application artifact must be published as a single self contained executable"
        description: "Bundling all application-dependent files into a single binary provides an application developer with the attractive option to deploy and distribute the application as a single file"
        reference_url: 
          - "https://docs.microsoft.com/en-us/dotnet/core/deploying/single-file"
  MAINTAINER:
    paramSyntaxRegex: /.+/
    rules:
      -
        label: "maintainer_deprecated"
        regex: /.+/
        level: "info"
        message: "the MAINTAINER command is deprecated"
        description: "MAINTAINER is deprecated in favor of using LABEL since Docker v1.13.0"
        reference_url:
          - "https://github.com/docker/cli/blob/master/docs/deprecated.md"
          - "#maintainer-in-dockerfile"        
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