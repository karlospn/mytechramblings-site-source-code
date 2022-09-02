---
title: "Keep your .NET platform images up to date using AWS ECR with Azure Pipelines"
date: 2022-08-23T15:36:41+02:00
tags: ["aws", "dotnet", "devops", "ecr", "containers"]
description: "When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images will be the base for your company applications. And in this post I'm going to show you an opinionated implementation of how to automate the creation and update of your .NET platform images."
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/keep-your-platform-images-updated-when-using-aws-ecr-and-azure-pipelines).

In this post I'm going to show you an opinionated implementation of how to automate the creation and update of your .NET platform images.

This implementation is going to be built to work solely with **AWS Elastic Container Registry** (AWS ECR) as image registry and **Azure Pipelines** as orchestrator for the creation or update of the platform images.  

But first, let's talk a little bit about what's a platform image.

# **1. What's a platform image?**

When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images will be the base for your company applications.

![platform-images-diagram](/img/platform-images.png)

The point of having and using a platform image instead of a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.

If your enterprise applications need some kind of software baked into the image to run properly, it is far better to install it only on the platform image, instead of having to install it in each and every one of the application images, this way it is far easier to roll updates.

For dotnet there are a few official docker images avalaible on the Microsoft registry (https://hub.docker.com/_/microsoft-dotnet/), but we don't want to use those images directly on our enterprise applications. Instead of that we are going to use those official images from Microsft as a base image to build a platform image that is going to be own by our company.   

The Microsoft official dotnet images are not inherently bad and often include many security best practices, but a more secure way is to not rely solely on third party images, because you lose the ability to control scanning, patching, and hardening across your organization. The recommended way is using these images as the base for building any of your company platform images.

# **2. Create and update a dotnet platform image**

Create a new platform image is an easy, but you also want to keep the image up to date.   
Every time the base image gets a new update from Microsoft you'll want to update your platform image, because the latest version it's usually the most secure.

The Microsoft image update policy for dotnet image is the following one:
- The .NET base images are updated within 12 hours of any updates to the underlying OS (e.g. debian:buster-slim, windows/nanoserver:ltsc2022, buildpack-deps:bionic-scm, etc.).
- When a new version of .NET (includes major/minor and servicing versions) gets released the dotnet base images get updated.

The .NET base images are updated quite freqüently, which means that automating the update of our platform image is paramount.

In this section we have talked about creating a new platform image and also the importance of keep it up to date when a new update on the base image is avalaible, but it is also possible that you might want to change something on an existing platform image (for example, install some new software, update some existing one, set some permissions, etc) and after doing that you will want to create a new version of the platform image right away.

I see 3 main things we'll need to implement to automate the creation/update of a dotnet platform image:
- Creation of a new platform image.
- Update of an existing platform image because the base image has a new updated version available.
- Update of an existing platform image because we have modified something and want to create a new version of the image.

# **3. Platform image creation/update automated process diagram**

## **Diagram**

The following diagram shows the necessary steps we're going to build to automate the creation or update of a platform image.  

The platform image creation/update process is going to be executed on an **Azure DevOps Pipeline**.   
Every platform image will have its own Azure DevOps pipeline.

![pipeline-diagram](/img/update-platform-images-pipeline.png)

The pipeline gets triggered on a scheduled basis (every Monday at 6:00 UTC), it uses a scheduled trigger because we want to periodically poll the Microsoft container registry (https://mcr.microsoft.com/) to check if there is any update available for the base image.

When trying to update an image if there is NO new update available on the base image then the pipeline just ends there.   

If there is a new updated base image on the Microsoft registry then the pipeline will create a new version of the platform image, test it, store it into ECR and notify the update into a Teams channel.

Also if you want to change something on an existing platform image (for example, install some new software, update some existing one, set some permissions, etc) you will want to create a new version of the platform image right away, you can do it setting the ``force_update`` pipeline parameter to ``true``, that parameter will skip the Microsoft container registry update check and go straight into creating a new platform image.

## **How to know if there is a new updated base image?**

If you're using Azure Container Registry (ACR) as your container registry you can use ACR Tasks to automate the creation of a new platform image when a container's base image is updated.
- _More info here about it: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-base-image-update_

But this kind of functionality does not exist for AWS ECR, so you'll need to build something on the side.

There are some third party tools like Diun or image-watch that allows us to receive notifications when a Docker image is updated on a Docker registry, but there is a much easier approach that using those tools:
- Use Azure Pipelines (or any other CI/CD service like GitHub Action) with a Cron Trigger that runs a pipelines thats checks if the base image has been updated.

But, how do we know if a dotnet base image has been updated?     
We can query the Microsoft Artifact Registry (https://mcr.microsoft.com) for any dotnet image to obtain when was pushed last time.   
For example, if we want to know when the "mcr.microsoft.com/dotnet/runtime:6.0-bullseye-slim" image whas pushed last image we can go fetch that information from here: "https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags".

From this point forward we can fetch the last time our platform image was pushed into AWS ECR and if this value is inferior then it means we're not using the most up to date base image and a new platform image needs to be built.

## **What's the test step all about?**

In the diagram above, you'll see a "Test Platform Image" step, but what does it do?

When updating a platform image we need to validate that the updated version are not going to break anything that could potentially affect our enterprise applications.   

The pipeline test step validates that you can run the platform image version that you're building in a real application without any unexpected error.

## **What tagging strategy are we going to use?**

A tagging strategy is important for managing multiple versions of platform images and make consuming it easier across your organization.

The platform images will be tagged using the upstream version of the base image with some minor tweaks.

This is how Microsoft tags their images:

- _{dotnet-version}-{os-version}-{architecture-os}_

Here's a few example of how a .NET runtime image is tagged:

- 6.0.8-bullseye-slim-amd64
- 6.0-bullseye-slim-amd64
- 6.0.8-bullseye-slim
- 6.0-bullseye-slim
- 6.0.8
- 6.0

One thing about the Microsoft tagging strategy is that the tags are NOT immutable, let's take the above example, when a new .NET runtime image is release there are some tags like "6.0" or "6.0-bullseye-slim" tags are going to be removed from the previous image version and applied to the new one.   
That's an interesting strategy because when you rebuild your application image if you're the "6.0" tag as you're base image you're always getting the most up to date .NET runtime image avalaible.

For my platform images I'm going to use a similar approach with a couple of minor changes.

- _{dotnet-version}-{os-version}-{versioning}_

A version is added at the end of the tag so if there is the necessity to use a previous platform image we can use it specifying the concrete version. It will look like this:

- 6.0-bullseye-slim-1.0.0  
- 6.0-bullseye-slim-1.1.0  

# **4. Building the platform image creation/update automated process.**

## **Key points**
From the previous section I can extract the following key points to keep in mind when building this solution:

- Platform images are going to be stored in AWS ECR.
- Azure Pipelines to run the process of creation/update the platform images.
- An Azure Pipeline scheduled trigger will be used to periodically check if any dotnet base image contains a new update.
- The ``Dockerfile`` for any platform image will be hosted on Azure DevOps Repos.
- Every time a new version of a platform is created
- Every time a new version of a platform is created a notification to a Teams Channel needs to be sent.

## **Repository structure**

The repository has 2 main folder: ``shared`` folder and ``platform-images`` folder.

### **/shared folder**

This folder contains a set of shared scripts and assets used by the platform images pipelines.

- The ``/scripts`` folder contains a set of shell scripts used by the Azure Pipelines YAML template.
- The ``/templates`` folder contains an Azure Pipeline YAML template. This template is used by every platform images pipeline.
- The ``/integration-tests/{dotnet version}`` folder contains an application that is going to be used as an integration test.

### **/platform-images folder**

The ``/platform-images/{dotnet version}`` folder will contain all your platform images, the platform images are segregated by dotnet version (net5, net6, net7, etc).   

For any platform image you'll need:
- A ``Dockerfile`` which contains a set of instructions and commands that will be used to create/build the platform image.
- An ``azure-pipelines.yml`` pipeline, it will be use to create/update the corresponding platform image.   

Here's an example of how the repository will look like:
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
## **Integration Test step**

When updating a platform image we need to validate that the update doesn't break anything that could potentially affect our applications.   

The ``/shared/integration-tests`` folder contains an series of applications that are going to be used as an integration test application. Every dotnet version will have its own integration test application.

When building a new version of a platform image, we will test that this new platform image doesn't break anything by using it in this integration test app.

The validation steps runs the following commands: 

- Retrieves the test application from the ``/integration-tests/{dotnet version}``
- Overrides the ``FROM`` statement from the integration test ``Dockerfile`` with the platform image we're building.
- Checks that the ``docker build`` process didn't thrown any warning or error.
- Starts the application using the ``docker run`` command.
- Sends an Http Request to the running application using cURL and expects that the response is a 200 OK status Code.

If all those steps run without any problem then the integration test is a success, if any of those steps fails the integration test fails and breaks the pipeline.

## **Building the pipeline**

Now it's time to build the pipeline. Let me remind you again what I'll be building.

![pipeline-diagram](/img/update-platform-images-pipeline.png)

I decided to use shell script to implement every step of the pipeline. Let's get to it.

### **force-update-process script**

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

echo "Contaimer with name: $aws_ecr_repository_name was pushed on Vy ECR last time at: $aws_image_pushed_at"

if [[ "$aws_image_pushed_at" < "$mcr_push_date_cleaned" ]] ;
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

## **Put every script inside a pipeline**

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
