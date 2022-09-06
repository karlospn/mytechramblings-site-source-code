---
title: "Keep your .NET platform images up to date using AWS ECR and Azure Pipelines"
date: 2022-08-23T15:36:41+02:00
tags: ["aws", "dotnet", "devops", "ecr", "containers", "docker"]
description: "When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images are the base for your company applications. In this post I'm going to show you an opinionated implementation of how to automate the creation and update of your own .NET platform images using Azure Pipelines."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/keep-your-platform-images-updated-when-using-aws-ecr-and-azure-pipelines).

In this post I'm going to show you an opinionated implementation of how to automate the creation and update your own .NET platform images.

This implementation is going to be built to work solely with **AWS Elastic Container Registry** (AWS ECR) as image registry, **Azure Repos** as VCS (Version Control Sofware) and **Azure Pipelines** as a runner to execute the process of creation and update of your own platform images.  

But first, let's talk a little bit about what's a platform image.

# **1. What's a platform image?**

When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images are the base for your company applications.

![platform-images-diagram](/img/platform-images.png)

The point of having a platform image instead of directly using a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.   
Also if your enterprise applications need some kind of software baked into the images to run properly, it is far better to install it on the platform images, instead of having to install it in each and every one of the application images.

When working with .NET, there are quite a few official base images availables (https://hub.docker.com/_/microsoft-dotnet/), but we don't want to use those images directly on our enterprise applications, instead we want to use those official images from Microsoft as a base image to build a platform image that is going to be own by our company.    
The official Microsoft .NET images are not inherently bad and often include many security best practices, but a more secure way is to not rely solely on third party images, because you lose the ability to control scanning, patching, and hardening across your organization. The recommended way is using the official Microsoft images as the base for building everyone of your company platform images.

# **2. Create and update a .NET platform image**

Create a new platform image is an easy task, but you also want to keep the image up to date. 

Every time the base image gets a new update from Microsoft you'll want to update your platform image, because the latest version it's usually the most secure one.

The Microsoft image update policy for dotnet images is the following one:
- The .NET base images are updated within 12 hours of any updates to the underlying OS (e.g. debian:buster-slim, windows/nanoserver:ltsc2022, buildpack-deps:bionic-scm, etc.).
- When a new version of .NET (includes major/minor and servicing versions) gets released the dotnet base images gets updated.

The .NET base images might get updated quite frequently according to these policies, which means that automating the creation and update of our platform images is paramount, but automation is not the only important step, it is also important being able to manually update an existing platform image (maybe because we have installed a new piece of software on the platform image or updated some existing one or set some permissions).

There are 3 main flows to implement when trying to automate the creation/update of a platform image:
- Create a new platform image from scratch.
- Update an existing platform image because the base image has a new update available.
- Update an existing platform image because we have modified something on the platform image itself and we need to create a new version of the image.

# **3. .NET Platform image creation/update process diagram**

## **Diagram**

The following diagram shows the necessary steps we're going to build to automate the creation/update of a .NET platform image.  

The platform image creation/update process is going to be executed on an **Azure DevOps Pipeline**.   
Every platform image will have its own Azure DevOps pipeline.

![pipeline-diagram](/img/update-platform-images-pipeline.png)

The pipeline uses a scheduled trigger because we need to periodically poll the Microsoft container registry (https://mcr.microsoft.com/) to check if there is any update available for the base image.   

When trying to update a platform image if there is NO new update available on the base image then the pipeline needs to end there.   
If there is a new updated base image on the Microsoft registry then the pipeline needs to create a new version of the platform image, test it, store it into ECR and notify that an update has ocurred via Teams channel.   
If you modify an existing platform image (for example, install some new software, update some existing one, set some permissions, etc) and want to create a new version of the platform image right away, you will be able to do it setting the ``force_update`` pipeline parameter to ``true``, that parameter will skip the Microsoft container registry update check and go straight into creating a new platform image.

## **How to know if a base image update is available**

If you're using Azure Container Registry (ACR) as your container registry you can use ACR Tasks to automate the creation of a new platform image when a container's base image is updated.
- More info about it: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-base-image-update

But this kind of functionality does not exist for AWS ECR, so you'll need to build something on the side.

There are some third party tools like Diun or image-watch that allows us to receive notifications when a Docker image is updated on a Docker registry, but there is a much easier approach that using those tools:
- Using Azure Pipelines (or any other CI/CD service like GitHub Action) with a Cron Trigger that runs a pipeline thats checks if the base image has a new updated version.

But, how do we know if a dotnet base image has been updated?     
We can query the Microsoft Artifact Registry (https://mcr.microsoft.com) for any dotnet image to obtain when was the last time it was pushed into the registry.   

Let me show you an example:

If we want to know when the "mcr.microsoft.com/dotnet/runtime:6.0-bullseye-slim" image was pushed last image we can fetch that information from here: 
- https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags

From this point forward we can fetch the last time our platform image was pushed into AWS ECR and if the value is inferior than the one from the Microsoft Register it means we're not using the most up to date base image and a new platform image needs to be built.

## **What's the test step about?**

In the diagram above, you'll see a "Test Platform Image" step, but what does it do?

When updating a platform image we need to validate that the new image is not going to break anything that could potentially affect our enterprise applications.   

The pipeline test step validates that you can successfully run the new platform image that you're building in a real application without any unexpected error.

## **What tagging strategy are we going to use?**

A tagging strategy is important for managing multiple versions of platform images and make consuming it easier across your organization.

The platform images will be tagged using the upstream version of the base image with some minor tweaks.

This is how Microsoft tags their images:

- _{dotnet-version}-{os-version}-{architecture-os}_

Here's an example of how the .NET6 runtime image is tagged:

- 6.0.8-bullseye-slim-amd64
- 6.0-bullseye-slim-amd64
- 6.0.8-bullseye-slim
- 6.0-bullseye-slim
- 6.0.8
- 6.0

One thing about the Microsoft tagging strategy is that the tags are NOT immutable, let's take the above example, when a new .NET6 runtime image is release there are some tags like "6.0" or "6.0-bullseye-slim" tags that are going to be removed from the current image and applied to the new one.   
That's an interesting strategy because every time you rebuild your enterprise application image if you're the "6.0" tag on the ``FROM`` statement you'll get always the most up to date .NET6 runtime image available.

For my platform images I'm going to use a similar approach with a couple of minor changes.

- _{dotnet-version}-{os-version}-{versioning}_

I have added a version at the end of the tag, that helps if the necessity to use a previous platform image arises in the future.    
This is how I will tag the platform images:

- 6.0-bullseye-slim-1.0.0  
- 6.0-bullseye-slim-1.1.0  

# **4. Building the .NET platform image creation/update process**

## **Key points**
From the previous section I can extract the following key points to keep in mind when building this solution:

- Platform images are going to be stored in AWS ECR.
- Azure Pipelines to run the process of creation/update the platform images.
- An Azure Pipeline scheduled trigger will be used to periodically check if any dotnet base image contains a new update.
- The ``Dockerfile`` for any platform image will be hosted on Azure DevOps Repos.
- Every time a new version of a platform is created a notification to a Teams Channel needs to be sent.

## **Repository structure**

The repository will have 2 main folder: A ``shared`` folder and a ``platform-images`` folder.   
Here's an example about how the repository will look like with a few platform images:
```bash
|
+---platform-images
|   +---net6
|   |   +---runtime
|   |   |       azure-pipelines.yml
|   |   |       Dockerfile
|   |   |
|   |   \---sdk
|   |           azure-pipelines.yml
|   |           Dockerfile
|   |
|   \---net7
|       +---runtime
|       |       azure-pipelines.yml
|       |       Dockerfile
|       |
|       +---runtime -deps
|       |       azure-pipelines.yml
|       |       Dockerfile
|       |
|       \---sdk
|               azure-pipelines.yml
|               Dockerfile
|
\---shared
    +---integration-tests
    |   +---net6
    |   |   |   .dockerignore
    |   |   |   appsettings.Development.json
    |   |   |   appsettings.json
    |   |   |   Dockerfile
    |   |   |   IntegrationTest.WebApi.csproj
    |   |   |   IntegrationTest.WebApi.sln
    |   |   |   Program.cs
    |   |   |
    |   |   +---Controllers
    |   |   |       MyServiceController.cs
    |   |   |
    |   |   +---DTO
    |   |   |       FooRqDto.cs
    |   |   |       FooRsDto.cs
    |   |   |
    |   |   \---Properties
    |   |           launchSettings.json
    |   |
    |   \---net7
    |       |   .dockerignore
    |       |   appsettings.Development.json
    |       |   appsettings.json
    |       |   Dockerfile
    |       |   IntegrationTest.WebApi.csproj
    |       |   IntegrationTest.WebApi.sln
    |       |   Program.cs
    |       |
    |       +---Controllers
    |       |       MyServiceController.cs
    |       |
    |       +---DTO
    |       |       FooRqDto.cs
    |       |       FooRsDto.cs
    |       |
    |       \---Properties
    |               launchSettings.json
    |
    +---scripts
    |       check-if-ecr-repository-exists.sh
    |       check-if-update-is-needed.sh
    |       docker-build.sh
    |       force-update-process.sh
    |       integration-test.sh
    |       push-to-private-ecr.sh
    |       send-teams-notification.sh
    |
    \---templates
            automatic-update-platform-image-template.yml
```

### **/shared folder**

This folder contains a set of shared scripts and assets used by the platform images pipelines.

- The ``/scripts`` folder contains a set of shell scripts used by the Azure Pipelines YAML template.
- The ``/templates`` folder contains an Azure Pipeline YAML template. This template is used by every platform images pipeline.
- The ``/integration-tests/{dotnet version}`` folder contains an application that is going to be used as an integration test.

### **/platform-images folder**

The ``/platform-images/{dotnet version}`` folder will contain all your platform images, the platform images will be segregated by dotnet version (net5, net6, net7, etc).   

For any platform image you'll need:
- A ``Dockerfile`` which contains a set of instructions and commands that will be used to build the platform image.
- An ``azure-pipelines.yml`` pipeline that it will be used to create and update the corresponding platform image.   

## **Integration Test step**

When updating a platform image we need to validate that the update doesn't break anything that could potentially affect our applications.   

The ``/shared/integration-tests`` folder will contain a series of applications that are going to be used as an integration test application. Every dotnet version will have its own integration test application.

When building a new version of a platform image, we will test that the new platform image doesn't break anything by building a real world application that is going to use the platform image as base image.


The integration test step runs the following commands: 

- Retrieves the test application from the ``/integration-tests/{dotnet version}``
- Overrides the ``FROM`` statement from the integration test ``Dockerfile`` with the platform image we're building.
- Builds an image of the integration test app  and checks that the ``docker build`` process didn't thrown any warning or error.
- Starts the application using the ``docker run`` command.
- Sends an Http Request to the running application using cURL and expects that the response is a 200 OK status Code.

If all those steps run without any problem then the integration test is a success, if any of those steps fails then the integration test fails and breaks the pipeline.

## **Building the pipeline**

Now it's time to build the pipeline. Let me remind you again what I'll be building.

![pipeline-diagram](/img/update-platform-images-pipeline.png)

I decided to use shell script to implement every step of the pipeline. Let's get to it.

### **force-update-process script**

This particular script runs only if the ``force_update`` parameter is set to ``true`` in the pipeline.

This parameter is used if you want to manually force the creation of a new platform image version. It skips the Microsoft container registry update check and goes straight into creating a new platform image.

This script does the following steps:
- It retrieves the version of the last platform image, increments the version value and stores it in a pipeline variable.    
Here's an example, if our latest platform image stored in ECR has this tag: _"6.0-bullseye-slim-1.3.0"_, we need to create the tag _"6.0-bullseye-slim-1.4.0"_ for the platform image we're creating right now.

> The image tag version that this script stores in a pipeline variable will be used later on when it's time to push the image into ECR.

```bash
#!/bin/bash

if [ -z "$1" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$1
fi

if [ -z "$2" ]; then
    echo "'aws_ecr_tag_without_version' parameter not found."
    exit 1
else
    aws_ecr_tag_without_version=$2
fi

echo "=========================================="
echo "Script parameters summary"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "aws_ecr_tag_without_version: $aws_ecr_tag_without_version"
echo "=========================================="

aws_image=$(aws ecr describe-images --repository-name $aws_ecr_repository_name --image-ids imageTag=latest)

echo "Checking image tags from the ECR image..."

for row  in $(echo $aws_image | jq -r '.imageDetails[0].imageTags'); do   
    sub=$aws_ecr_tag_without_version
    if [[ "$row" == *"$sub"* ]]; then
        aws_image_tag=$row
        break
    fi
done

if [ -z "$aws_image_tag" ]; then
    echo "Tag was not found on the ECR image."
    exit 1
fi

echo "Image tag found: $aws_image_tag"
echo "Now calculating new image tag..."

version=$(echo $aws_image_tag | sed 's/.*-//')
if [ -z "$version" ]; then
    echo "Tag version was not found on the ECR image."
    exit 1
fi

major=$(echo $version |  tr -d '"' | cut -d '.' -f 1)
if [ -z "$major" ]; then
    echo "Tag major version was not found on the ECR image."
    exit 1
fi

minor=$(echo $version |  tr -d '"' | cut -d '.' -f 2)
if [ -z "$minor" ]; then
    echo "Tag minor version was not found on the ECR image."
    exit 1
fi

patch=$(echo $version |  tr -d '"' | tr -d ',' | cut -d '.' -f 3)
if [ -z "$patch" ]; then
    echo "Tag patch version was not found on the ECR image."
    exit 1
fi

update_minor=$(($minor+1))
new_tag_version="$aws_ecr_tag_without_version-$major.$update_minor.0"

echo "New image tag is: $new_tag_version"
echo "Now setting this as a pipeline environment variable."
echo "##vso[task.setvariable variable=aws_tag_version]$new_tag_version"
```

### **check-if-ecr-repository-exists script**

This script checks if the ECR repository exists using the ``aws ecr describe-repositories`` command.

- If the repository doesn't exist:
    - It sets the platform image tag version to 1.0.0 and stores it in a pipeline variable. 
    - It sets the ``skip_update_check`` pipeline variable to ``true``. 
      - This pipeline variable is used to skip the step that validates if there is a new update available in the Microsoft registry. If the repository doesn't exists it means that we're building a new platform image from the ground up, so it makes no sense to check if there is a new base image update available.

> The image tag version that the script stored in a pipeline variable will be used later on when it's time to push the image into ECR.


```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$1
fi

if [ -z "$2" ]; then
    echo "'aws_ecr_tag_without_version' parameter not found."
    exit 1
else
    aws_ecr_tag_without_version=$2
fi

echo "=========================================="
echo "Script parameters summary"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "aws_ecr_tag_without_version: $aws_ecr_tag_without_version"
echo "=========================================="

echo "Check if private ECR Repository exists..."
output=$(aws ecr describe-repositories --repository-names $aws_ecr_repository_name 2>&1)

if [ $? -ne 0 ]; then
  if echo ${output} | grep -q RepositoryNotFoundException; then
    echo "ECR Repository not found"
    echo "Set environment variables to skip update check step"
    new_tag_version="$aws_ecr_tag_without_version-1.0.0"
    echo "Set environment variables tag version: $new_tag_version"
    echo "##vso[task.setvariable variable=aws_tag_version]$new_tag_version"
    echo "##vso[task.setvariable variable=skip_update_check]true"
    exit 0
  else
    echo "Error running the 'ecr describe-repositories' command"
    exit 1
  fi
fi

echo "Private ECR Repository found"
```

### **check-if-update-is-needed script**

This script checks if there is a new base image available.

How we do that? The script does the following steps:

- Queries the Microsoft Artifact Registry (https://mcr.microsoft.com) to obtain the ``lastModifiedDate`` property of the base image.
- Fetches when was the last time that our platform image was pushed into AWS ECR, and if this value is inferior than the one retrieved from the Microsoft Artifact Registry then it means we're not using the most up to date base image and a new platform image needs to be built.
- It retrieves the tag version of the current platform image, increments the version value and stores it in a pipeline variable.    
For example, if our current platform image stored in ECR has the tag: _"6.0-bullseye-slim-1.3.0"_, we want to create a new tag called _"6.0-bullseye-slim-1.4.0"_ for the platform image we're creating right now.

> The image tag version that this script stores in a pipeline variable will be used later on when it's time to push the image into ECR.

```bash
#!/bin/bash

if [ -z "$1" ]; then
    echo "'mcr_registry_uri' parameter not found."
    exit 1
else
    mcr_registry_uri=$1
fi

if [ -z "$2" ]; then
    echo "'mcr_tag_name' parameter not found."
    exit 1
else
    mcr_tag_name=$2
fi

if [ -z "$3" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$3
fi

if [ -z "$4" ]; then
    echo "'aws_ecr_tag_without_version' parameter not found."
    exit 1
else
    aws_ecr_tag_without_version=$4
fi

echo "=========================================="
echo "Script parameters summary"
echo "mcr_registry_uri: $mcr_registry_uri"
echo "mcr_tag_name: $mcr_tag_name"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "aws_ecr_tag_without_version: $aws_ecr_tag_without_version"
echo "=========================================="

http_response=$(curl -s -o response.txt -w "%{http_code}" $mcr_registry_uri)

if [ $http_response != "200" ]; then
   echo "Error retrieving container image information from Microsoft MCR Registry."
   exit 1
fi

body="$(cat response.txt)"

if [ -z "$body" ]; then
    echo "Container image information response from Microsoft MCR Registry came empty."
    exit 1
fi

mcr_push_date=$(jq --arg mcr_tag_name "$mcr_tag_name" '.[] | select(.name==$mcr_tag_name) | .lastModifiedDate' response.txt)

if [ -z "$mcr_push_date" ]; then
    echo "Container image information not found on Microsoft MCR Registry."
    exit 1
fi

mcr_push_date_cleaned=$(echo $mcr_push_date | tr -d '"' | cut -d "T" -f 1)
echo "Container with name: $mcr_tag_name was pushed last time on the MCR registry at: $mcr_push_date_cleaned"

aws_image=$(aws ecr describe-images --repository-name $aws_ecr_repository_name --image-ids imageTag=latest)

if [ -z "$aws_image" ]; then
    echo "Image $aws_ecr_repository_name not found on ECR."
    exit 1
fi

aws_image_pushed_at=$(echo $aws_image | jq -r '.imageDetails[0].imagePushedAt' | cut -d "T" -f 1)

if [ -z "$aws_image_pushed_at" ]; then
    echo "Image $aws_ecr_repository_name on ECR do not contain a imagePushedAt attribute."
    exit 1
fi

echo "Contaimer with name: $aws_ecr_repository_name was pushed on ECR last time at: $aws_image_pushed_at"

if [[ "$aws_image_pushed_at" > "$mcr_push_date_cleaned" ]] ;
then
    echo "There are no new versions of the image in the MCR registry. Nothing further to do."
    echo "##vso[task.setvariable variable=skip_tasks]true"
    exit 0
fi

echo "Now checking image tags from the ECR image..."

for row  in $(echo $aws_image | jq -r '.imageDetails[0].imageTags'); do   
    sub=$aws_ecr_tag_without_version
    if [[ "$row" == *"$sub"* ]]; then
        aws_image_tag=$row
        break
    fi
done

if [ -z "$aws_image_tag" ]; then
    echo "Tag was not found on the ECR image."
    exit 1
fi

echo "Image tag found: $aws_image_tag"
echo "Now calculating new image tag..."

version=$(echo $aws_image_tag | sed 's/.*-//')
if [ -z "$version" ]; then
    echo "Tag version was not found on the ECR image."
    exit 1
fi

major=$(echo $version |  tr -d '"' | cut -d '.' -f 1)
if [ -z "$major" ]; then
    echo "Tag major version was not found on the ECR image."
    exit 1
fi

minor=$(echo $version |  tr -d '"' | cut -d '.' -f 2)
if [ -z "$minor" ]; then
    echo "Tag minor version was not found on the ECR image."
    exit 1
fi

patch=$(echo $version |  tr -d '"' | tr -d ',' | cut -d '.' -f 3)
if [ -z "$patch" ]; then
    echo "Tag patch version was not found on the ECR image."
    exit 1
fi

update_minor=$(($minor+1))
new_tag_version="$aws_ecr_tag_without_version-$major.$update_minor.0"

echo "New image tag is: $new_tag_version"
echo "Now setting this as a pipeline environment variable."
echo "##vso[task.setvariable variable=aws_tag_version]$new_tag_version"
```

### **docker-build script**

This script builds the platform image. Simple as that.

```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "'image_context' parameter not found."
    exit 1
else
    image_context=$1
fi

echo "=========================================="
echo "Script parameters summary"
echo "image_context: $image_context"
echo "=========================================="

docker build -t platform.image:tmp $image_context
```

### **integration-test script**

This script validates that the platform image created in the script above works in a real world application without any error or warning.

The script does the following steps:
- Retrieves the test application from the ``/integration-tests/{dotnet version}``
- Overrides the ``FROM`` statement from the integration test ``Dockerfile`` with the platform image we're building.
- Checks that the ``docker build`` process didn't thrown any warning or error.
- Starts the application using the ``docker run`` command.
- Sends an Http Request to the running application using cURL and expects that the response is a 200 OK status Code.

```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$1
fi

if [ -z "$2" ]; then
    echo "'integration_test_dockerfile_local_path' parameter not found."
    exit 1
else
    integration_test_dockerfile_local_path=$2
fi

if [ -z "$3" ]; then
    echo "'integration_test_dockerfile_local_overwrite_image_name' parameter not found."
    exit 1
else
    integration_test_dockerfile_local_overwrite_image_name=$3
fi

if [ -z "$4" ]; then
    echo "'integration_test_dockerfile_context' parameter not found."
    exit 1
else
    integration_test_dockerfile_context=$4
fi

echo "=========================================="
echo "Script parameters summary"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "integration_test_dockerfile_local_path: $integration_test_dockerfile_local_path"
echo "integration_test_dockerfile_local_overwrite_image_name: $integration_test_dockerfile_local_overwrite_image_name"
echo "integration_test_dockerfile_context: $integration_test_dockerfile_context"
echo "=========================================="

export DOCKER_BUILDKIT=1

if [[ -f "$integration_test_dockerfile_local_path" ]]; then
    echo "$integration_test_dockerfile_local_path exists."
else
    echo "$integration_test_dockerfile_local_path file does not exist"
    exit 1
fi

echo "Replacing content on Dockerfile..."
sed -i "/$integration_test_dockerfile_local_overwrite_image_name/c\FROM platform.image:tmp" $integration_test_dockerfile_local_path

echo "Output dockerfile content"
cat $integration_test_dockerfile_local_path

echo "Building image..."
docker build -t testapp:latest --progress=plain  -f $integration_test_dockerfile_local_path $integration_test_dockerfile_context 2>&1 | tee build.log

echo "Analyze docker build log output..."
grep -q "warning" build.log; [ $? -eq 0 ] && echo "warnings found on the docker build log" && exit 1
echo "No errors or warnings found on docker build log output"

echo "Run image..."
container_id=$(docker run -d -p 5055:8080 testapp:latest)
sleep 20

echo "Print docker logs..."
docker logs $container_id

echo "Run integration test..."
status_code=$(curl -X 'POST' --write-out %{http_code} -k --silent --output /dev/null 'http://localhost:5055/MyService' -H 'accept: application/json' -H 'Content-Type: application/json-patch+json' -d '{ "data": "string", "color": "Red"}')
echo $status_code

if [[ "$status_code" -ne 200 ]] ; then
    echo "Integration Test failed."
    exit 1
else
    echo "Integration Test succeeded."
fi
```

### **push-to-private-ecr script**

This script creates the ECR repository if it doesn't exists and afterwards it pushes the platform image and tags into the repository.

The script does the following steps:
- Creates the ECR repository using the ``aws ecr create-repository`` command, if needed.
- Retrieves the tag version set in the previous scripts and tags the image.
- Tags the image with the ``latest`` tag and and some extra tags we can pass them as a parameter.
- Pushes the tags and the image into ECR.

```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "'aws_account' parameter not found."
    exit 1
else
    aws_account=$1
fi

if [ -z "$2" ]; then
    echo "'aws_region' parameter not found."
    exit 1
else
    aws_region=$2
fi

if [ -z "$3" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$3
fi

if [ -z "$4" ]; then
    echo "'aws_tag_version' environment variable not found."
    exit 1
else
    aws_tag_version=$4
fi

if [ -z "$5" ]; then
    echo "'aws_extra_tags' parameter not found."
    exit 1
else
    aws_extra_tags=$5
fi

echo "=========================================="
echo "Script parameters summary"
echo "aws_account: $aws_account"
echo "aws_region: $aws_region"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "aws_tag_version: $aws_tag_version"
echo "aws_extra_tags: $aws_extra_tags" 
echo "=========================================="

aws ecr get-login-password --region $aws_region | docker login --username AWS --password-stdin $aws_account.dkr.ecr.$aws_region.amazonaws.com
aws ecr create-repository --repository-name $aws_ecr_repository_name --region $aws_region --image-scanning-configuration scanOnPush=true
docker tag platform.image:tmp $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:$aws_tag_version
docker tag platform.image:tmp $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:latest
docker push $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:$aws_tag_version
docker push $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:latest

array_tags=( $aws_extra_tags )
for extra_tag in "${array_tags[@]}"
do
    docker tag platform.image:tmp $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:$extra_tag
    docker push $aws_account.dkr.ecr.$aws_region.amazonaws.com/$aws_ecr_repository_name:$extra_tag
done
```

### **send-teams-notification script**

This script notifies to a Teams Channel that a new platform image is available.

The script does the following steps:
- Builds an adaptative card for Microsoft Teams.
- Sends the card to Microsoft Teams using a HTTP WebHook.

```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "'aws_ecr_repository_name' parameter not found."
    exit 1
else
    aws_ecr_repository_name=$1
fi

if [ -z "$2" ]; then
    echo "'aws_tag_version' environment variable not found."
    exit 1
else
    aws_tag_version=$2
fi

if [ -z "$3" ]; then
    echo "'teams_webhook_uri' environment variable not found."
    exit 1
else
    teams_webhook_uri=$3
fi

echo "=========================================="
echo "Script parameters summary"
echo "aws_ecr_repository_name: $aws_ecr_repository_name"
echo "aws_tag_version: $aws_tag_version"
echo "teams_webhook_uri: $teams_webhook_uri"
echo "=========================================="

date_now=$(date +"%Y-%m-%d %T")
message='{"type":"message","attachments":[{"contentType":"application/vnd.microsoft.card.adaptive","content":{"type":"AdaptiveCard","body":[{"type":"TextBlock","size":"High","weight":"Bolder","text":"New platform image available."},{"type":"TextBlock","size":"Medium","weight":"Bolder","text":"A new platform image is **available** and **ready** to use."},{"type":"FactSet","facts":[{"title":"Image Name:","value":"'"$aws_ecr_repository_name"'"},{"title":"Version Tag:","value":"'"$aws_tag_version"'"},{"title":"Creation Time:","value":"'"$date_now"'"}]}],"$schema":"http://adaptivecards.io/schemas/adaptive-card.json","version":"1.0","msteams":{"width":"Full"}}}]}'

status_code=$(curl --write-out %{http_code} -k --silent --output /dev/null -H "Content-Type: application/json" -H "Accept: application/json" "${teams_webhook_uri}" -d "${message}")
echo $status_code

if [[ "$status_code" -ne 200 ]] ; then
    echo "Webhook post failed."
else
    echo "Webhook post succeed."
fi
```

## **Create the Azure DevOps Pipeline**

### **Create an Azure DevOps YAML template**

In the previous section we have been building the different steps we're going to need, now it's time to put them together.

I decided to build an Azure DevOps Template Pipeline and reuse it in every platform image pipeline.

Here's how the resulting Azure DevOps YAML template looks like:

```yaml
parameters:
- name: aws_credentials
  type: string
- name: aws_account
  type: string
- name: private_ecr_region
  type: string
- name: aws_ecr_repository_name
  type: string
- name: aws_ecr_tag_without_version
  type: string
- name: aws_ecr_extra_tags
  type: object  
- name: mcr_registry_uri
  type: string
- name: mcr_tag_name
  type: string
- name: dockerfile_context_path
  type: string
- name: integration_test_dockerfile_local_path
  type: string
- name: integration_test_dockerfile_local_overwrite_image_name
  type: string
- name: integration_test_dockerfile_context
  type: string
- name: teams_webhook_uri
  type: string
- name: force_update
  type: string
  default: false

variables:
  aws_ecr_extra_tags_stringified: ${{join(' ',parameters.aws_ecr_extra_tags)}}

steps:
- task: AWSShellScript@1
  displayName: 'Check if forced update has been specified'
  condition: and(succeeded(), eq('${{ parameters.force_update }}', 'true'))
  inputs:
    awsCredentials: '${{ parameters.aws_credentials }}'
    regionName: '${{ parameters.private_ecr_region }}'
    arguments: '${{ parameters.aws_ecr_repository_name }} ${{ parameters.aws_ecr_tag_without_version }}'
    scriptType: 'filePath'
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/force-update-process.sh'

- task: AWSShellScript@1
  displayName: 'Check if ECR repository exists'
  condition: and(succeeded(), eq('${{ parameters.force_update }}', 'false'))
  inputs:
    awsCredentials: '${{ parameters.aws_credentials }}'
    regionName: '${{ parameters.private_ecr_region }}'
    arguments: '${{ parameters.aws_ecr_repository_name }} ${{ parameters.aws_ecr_tag_without_version }}'
    scriptType: 'filePath'
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/check-if-ecr-repository-exists.sh'

- task: AWSShellScript@1
  displayName: 'Check if there is a new update'
  condition: and(succeeded(), ne(variables['skip_update_check'], 'true'), eq('${{ parameters.force_update }}', 'false'))
  inputs:
    awsCredentials: '${{ parameters.aws_credentials }}'
    regionName: '${{ parameters.private_ecr_region }}'
    arguments: '${{ parameters.mcr_registry_uri}} ${{ parameters.mcr_tag_name }} ${{ parameters.aws_ecr_repository_name }} ${{ parameters.aws_ecr_tag_without_version }} ${{ parameters.force_update}}'
    scriptType: 'filePath'
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/check-if-update-is-needed.sh'

- task: Bash@3
  displayName: 'Build new image'
  condition: and(succeeded(), ne(variables['skip_tasks'], 'true'))
  inputs:
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/docker-build.sh'
    arguments: '${{ parameters.dockerfile_context_path }} $(Build.BuildNumber)'
  
- task: Bash@3
  displayName: 'Integration Test'
  condition: and(succeeded(), ne(variables['skip_tasks'], 'true'))
  inputs:
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/integration-test.sh'
    arguments: '${{ parameters.aws_ecr_repository_name }} ${{ parameters.integration_test_dockerfile_local_path }} ${{ parameters.integration_test_dockerfile_local_overwrite_image_name }} ${{ parameters.integration_test_dockerfile_context }}'
  
- task: AWSShellScript@1
  displayName: 'Publish new image to private ECR registry'
  condition: and(succeeded(), ne(variables['skip_tasks'], 'true'))
  inputs:
    awsCredentials: '${{ parameters.aws_credentials }}'
    regionName: '${{ parameters.private_ecr_region }}'
    arguments: '${{ parameters.aws_account }} ${{ parameters.private_ecr_region }} ${{ parameters.aws_ecr_repository_name }} $(aws_tag_version) "$(aws_ecr_extra_tags_stringified)"'
    scriptType: 'filePath'
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/push-to-private-ecr.sh'

- task: Bash@3
  displayName: 'Send notification to teams'
  condition: and(succeeded(), ne(variables['skip_tasks'], 'true'))
  inputs:
    filePath: '$(System.DefaultWorkingDirectory)/shared/scripts/send-teams-notification.sh'
    arguments: '${{ parameters.aws_ecr_repository_name }} $(aws_tag_version) $(teams_webhook_uri)'
```
This YAML template has the following parameters:
- ``aws_credentials``: An Azure DevOps Service Connection to AWS.
- ``aws_account``: AWS Account Number.
- ``private_ecr_region``: AWS Region
- ``aws_ecr_repository_name``: Name of the ECR repository where the image is going to be stored.
- ``aws_ecr_tag_without_version``: The main tag we want to add to the platform image. The main tag will be automatically versioned by the pipeline.   
Here's an example, let's say we set it to ``6.0-bullseye-slim``, if the platform image doesn't exist yet a tag named ``6.0-bullseye-slim-1.0.0`` will be added to the image , if the platform image already exists a tag named ``6.0-bullseye-slim-1.1.0`` will be added to the image, and so forth and so on in the following executions.
- ``aws_ecr_extra_tags``: Extra tags we want to add to this platform image, like the build Id, build Number, dotnet version, etc.
- ``mcr_registry_uri``: The URI of the Microsoft Registry which we will be used to check if the base image contains a new update.   
For example, if the platform image is using the ``dotnet/runtime:6.0-bullseye-slim`` image as a base image, then the ``mcr_registry_uri`` has to be  ``https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags``.   
If the platform image is using the ``dotnet/runtime-deps:6.0-bullseye-slim`` image as a base image, then the ``mcr_registry_uri`` has to be ``https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime-deps/tags``.
- ``mcr_tag_name``: The specific image tag from the Microsoft Registry which we will used to search if the base image contains a new update.   
For example, if the platform image is using the ``dotnet/runtime:6.0-bullseye-slim`` image as a base image, then the ``mcr_tag_name`` has to be  ``6.0-bullseye-slim``
- ``dockerfile_context_path``: The location of the platform image build context.
- ``integration_test_dockerfile_local_path``: The location of the integration test dockerfile.
- ``integration_test_dockerfile_local_overwrite_image_name``: Which ``FROM`` statement needs to be overridden on the integration test Dockerfile.
- ``integration_test_dockerfile_context``: The location of the integration test build context.
- ``teams_webhook_uri``: A Teams WebHook Uri. The pipelines notifies to a Teams Channel when a new platform image has been created or updated.
- ``force_update``: If you want to create a new version of the platform image right away, you can do it setting this parameter to ``true`` and it will skip the Microsoft container registry update check and go straight into creating a new platform image.

### **Create an Azure DevOps Pipeline using the YAML Template**

Now it's time to create an actual platform image pipeline.

The pipeline needs to contain the following features:
- The ``trigger`` attribute set to ``none`` because there is not need to create a push trigger. The only way to generate a  platform image will be via the scheduled trigger or using the ``force_update`` parameter.
- An scheduled trigger to periodically execute the pipeline. The schedule functionality will be used to poll the Microsoft Container Registry (https://mcr.microsoft.com/) to check if there is any update available for the base image.
- The ``force_update`` parameter is going to be a runtime parameter (https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters). This parameter is used if you want to manually create a new version of a platform image. 
- Use the pipeline YAML template we have built in the previous section.

Here's a real example of how a Pipeline will look like:

```yaml
trigger: none

schedules:
- cron: "0 6 * * 1"
  displayName: Run At 06:00 UTC every Monday
  branches:
    include:
    - main

parameters:
- name: force_update
  type: boolean
  default: false

pool:
  vmImage: ubuntu-latest

extends:
  template: ../../../shared/templates/automatic-update-platform-image-template.yml
  parameters:
      aws_credentials: 'aws-dev'
      aws_account: '951281301247'
      private_ecr_region: 'eu-west-1'
      aws_ecr_repository_name: 'runtime' 
      aws_ecr_tag_without_version: '6.0-bullseye-slim' 
      aws_ecr_extra_tags:
      - 6.0
      - $(Build.BuildNumber)
      mcr_registry_uri: 'https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags' 
      mcr_tag_name: '6.0-bullseye-slim' 
      dockerfile_context_path: '$(System.DefaultWorkingDirectory)/platform-images/net6/runtime' 
      integration_test_dockerfile_local_path: '$(System.DefaultWorkingDirectory)/shared/integration-tests/net6/Dockerfile' 
      integration_test_dockerfile_local_overwrite_image_name: 'runtime:6.0-bullseye-slim' 
      integration_test_dockerfile_context: '$(System.DefaultWorkingDirectory)/shared/integration-tests/net6' 
      teams_webhook_uri: '$(TEAMS_WEBHOOK_URI)'
      force_update: ${{ parameters.force_update }}
```

# **5. Testing everything out**

Let's start by building a ``Dockerfile`` for our new platform image.

```yaml
FROM mcr.microsoft.com/dotnet/runtime:6.0-bullseye-slim

# Set non-root user
RUN groupadd -r devsecops && useradd -r --uid 1000 -g devsecops devsecops \
    && mkdir /app \
    && mkdir /home/devsecops \
    && chown -R devsecops /app \
    && chown -R devsecops /home/devsecops

# Set the default user
USER devsecops

# Non-user root cannot start on port 80
ENV ASPNETCORE_URLS=http://+:8080
```


This platform image contains the following features:
- Uses the ``dotnet/runtime:6.0-bullseye-slim`` as a base image.
- Uses a non-root container image.
- Sets the starting port as ``8080``

By default a dotnet app runs as root inside a container. Root in a container is not the same as root on the host. Docker restricts users in containers. But to decrease the security-attack surface, you’d want to run the container as an unprivileged user.   
A non-root user cannot run on port 80. A new port needs to be specified.

Now let's create the Azure DevOps pipeline that will be used to create and update this platform image.

- The main tag for this image is going to be ``6.0-bullseye-slim``. This tag will be automatically version by the pipeline, which means that after the first execution the image will have assoaciated the ``6.0-bullseye-slim-1.0.0`` tag.
- The Microsoft Registry Artifact endpoint that we want to monitor to check if a new version of the base image is available is this one: ``https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags``.


```yaml
trigger: none

schedules:
- cron: "0 6 * * 1"
  displayName: Run At 06:00 UTC every Monday
  branches:
    include:
    - main

parameters:
- name: force_update
  type: boolean
  default: false

pool:
  vmImage: ubuntu-latest

extends:
  template: ../../../shared/templates/automatic-update-platform-image-template.yml
  parameters:
      aws_credentials: 'aws-dev'
      aws_account: '951281301247'
      private_ecr_region: 'eu-west-1'
      aws_ecr_repository_name: 'runtime' 
      aws_ecr_tag_without_version: '6.0-bullseye-slim' 
      aws_ecr_extra_tags:
      - 6.0
      - $(Build.BuildNumber)
      mcr_registry_uri: 'https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags' 
      mcr_tag_name: '6.0-bullseye-slim' 
      dockerfile_context_path: '$(System.DefaultWorkingDirectory)/platform-images/net6/runtime' 
      integration_test_dockerfile_local_path: '$(System.DefaultWorkingDirectory)/shared/integration-tests/net6/Dockerfile' 
      integration_test_dockerfile_local_overwrite_image_name: 'runtime:6.0-bullseye-slim' 
      integration_test_dockerfile_context: '$(System.DefaultWorkingDirectory)/shared/integration-tests/net6' 
      teams_webhook_uri: '$(TEAMS_WEBHOOK_URI)'
      force_update: ${{ parameters.force_update }}
```

Now let's run this pipeline a few times to test it out, but first we need to add an "Incoming WebHook" to a Teams Channel. This WebHook will be used by the pipeline to notify that a new version of the platform image is available.

![platform-img-incoming-webhook](/img/platform-img-incoming-webhook.png)

After creating the WebHook we have to add the WebHook URL as a Pipeline variable.

![platform-img-teams-webhook.uri](/img/platform-img-teams-webhook.uri.png)

Let's run the pipeline for the first time. Here's the result:

![platform-img-new-pipeline-execution](/img/platform-img-new-pipeline-execution.png)

As you can see the step that checks if a new update is available has not been executed because we're creating a platform image for the first time.

If we take a look at the Teams Channel where we added the "Incoming WebHook" a new notification has popped up.

![platform-img-channel-notification](/img/platform-img-channel-notification.png)

Now let's wait for the scheduled trigger to execute the pipeline, here's the result of the pipeline execution:

![platform-img-existing-pipeline-execution](/img/platform-img-existing-pipeline-execution.png)

As you can see the pipeline was executed successfully, but the execution stopped on the "Check if there is a new update" step, if we drill-down on this concrete step.

![platform-img-no-new-update](/img/platform-img-no-new-update.png)

In the image below you can see that there were no new update available on the Microsoft Artifact Registry so the pipeline ended there.

Now let's force the creation of a new platform image, to do that we need to check the ``force_update`` parameter in the pipeline.

![platform-img-force-update](/img/platform-img-force-update.png)

Let's take a look at AWS ECR after forcing the creation of a new platform image version a couple more times.    

![platform-img-autoincrement-version-tag](/img/platform-img-autoincrement-version-tag.png)

As you can see in the image above.
- The tag _"6.0-bullseye-slim"_ has been automatically versioned everytime a new version of the platform image has been pushed into ECR.
- The ``latest`` tag points to the latest version of this platform image.
- The ``6`` tag points to the latest version of this platform image.

And that's it! 