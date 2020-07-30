---
title: "A practical example of GitOps using Azure DevOps, Azure ACR, Helm, Flux and Kubernetes"
date: 2020-07-29T22:10:21+02:00
tags: ["azure", "acr", "kubernetes", "gitops", "helm"]
draft: true
---


GitOps is a term introduced by WeaveWorks a couple years ago (https://www.weave.works/blog/gitops-operations-by-pull-request)

GitOps is a way of implementing Continuous Deployment. The core idea of GitOps is having a Git repository that always contains declarative descriptions of the infrastructure currently desired in the production environment and an automated process to make the production environment match the described state in the repository.   
If you want to deploy a new application or update an existing one, you only need to update the repository - the automated process handles everything else.

In these post I want to build a practical example of GitOps, it's not going to be a production-ready example but nonetheless it's going to be an interesting proof of concept, so let's try it.

## Components
---

I'm going to use the following components:

- **Azure DevOps** as my VCS (Version control system) 
- **Azure ACR** as my container registry and also as my helm chart repository.
- **Kubernetes** on-premise for hosting the applications.
- **Helm** to pack the application Kubernetes files.
- **Flux** and the **Helm-Operator** to deploy the applications in an automatic way into the cluster.
  
That's a diagram of how the components are going to interact.

![diagram](/img/gitops-ci.png)


Let me explain a little bit about how it is going to work:

1- The **Developer** creates a new application an pushes the code into an Azure DevOps repository. 

2- The **code pushed** triggers an Azure Pipeline that creates a container image and also a helm chart and pushes everything into an existing Azure Container Registry.  

3- The **DevOps Team** has a **centralized repository** where they maintain a declarative description of all the components deployed into the Kubernetes cluster.    

4- The **DevOps Team** pushes a new file into the **centralized repository**, that file contains the description of the new application that the developer has created on step 1.  
**Flux** is keeping watch for changes in the centralized repository and when it sees a new file, it picks them up and it deploys the new application using the helm chart that we created in step 2.  

_**In my example I'm using 2 different actors: a developer and a devOps teams, but there are a lot of viable combinations possible, just to enumerate a few:_ 
- _It could be the same person that does all the steps by himself, you might not need two actors._
- _It could that the developer creates the application, the automatic pipeline creates the container and the helm chart and when it finishes it triggers an automatic process that creates or updates the configuration file on the centralized repository._


Anyways, let's start building that scenario


## Step 0 - Initial setup
---

1- On my Azure DevOps account I'm just going to create:   

- A new Team Project called "**Provisioning**", and inside it I will create:
  - An Git repository named **"AppA"**: 
    - That's where the application source code is going to be. 
  - A Git repository named **"Manifests"**: 
    - That's the centralized repository that Flux is going to monitor.
- An Azure Service connection with permissions to push a docker image and  a helm chart into ACR.

2 - Create an Azure Container Registry. 


## Step 1 - Creating an application
---

It's going to be a .NET Core 3.1 WebAPI.  The application per se it's not important, it could be anything.  

There are a couple of things I want to remark: 
- The app needs to have a _DockerFile_, because I'm using Kubernetes and Kubernetes likes containers...
- The app also needs to have a _/chart_ folder. That folder is going to contain all the Kubernetes definition files that the application needs to run.   
I need that folder because I'm going to pack all the Kubernetes files into a Helm Chart and use it afterwards to deploy the application inside the cluster.

An example of the application structure would be something like this:

```bash
.
│   .gitignore
│   azure-pipelines.yml
│   README.md
│
└───AppA.WebApi
    │   .dockerignore
    │   AppA.WebApi.csproj
    │   AppA.WebApi.sln
    │   appsettings.Development.json
    │   appsettings.json
    │   Dockerfile
    │   Dockerfile.develop
    │   Program.cs
    │   Startup.cs
    │   WeatherForecast.cs
    │
    ├───chart
    │   │   .helmignore
    │   │   Chart.yaml
    │   │   values.yaml
    │   │
    │   └───templates
    │           deployment.yaml
    │           ingress.yaml
    │           secrets.yaml
    │           service.yaml
    │           _helpers.tpl
    │
    ├───Controllers
    │       WeatherForecastController.cs
    │
    └───Properties
            launchSettings.json

```

As I pointed out the app contains a Dockerfile and also a _/chart_ folder.

After seeing that you could argue that maybe I should not place the Kubernetes files / Chart files within the application code and maybe I should place it directly in the centralized repository, and that's totally legit but I prefer to have the application code and the application kubernetes definitions all in the same place.


## Step 2 - Build the Azure Pipeline to create the docker image and helm chart
---

We are going to build an Azure Pipeline that is going to do those 4 steps:

- Build the docker image
- Push the docker image into an existing ACR
- Package the helm chart 
- Push the helm chart into an existing ACR


That's the pipeline:

```YAML
trigger:
- master

resources:
- repo: self

variables:
  HELM_EXPERIMENTAL_OCI: 1
  registryName: 'acrgitopsdev'
  imageName: 'appawebapi'
  BuildId: '$(Build.BuildId)'
  tag: '$(Build.BuildId)'

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr login --name $(registryName)'
    
- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr build --image $(imageName):$(BuildId) --registry $(registryName) --file Dockerfile .'
    useGlobalConfig: true
    workingDirectory: '$(Build.SourcesDirectory)/AppA.WebApi'

- task: HelmInstaller@1
  inputs:
    helmVersionToInstall: 'latest'

- task: replacetokens@3
  inputs:
    targetFiles: '**/*.yaml'
    encoding: 'auto'
    writeBOM: true
    actionOnMissing: 'warn'
    keepToken: false
    tokenPrefix: '#{'
    tokenSuffix: '}#'
    useLegacyPattern: false
    enableTelemetry: true
       
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |
      helm package --destination $(Build.ArtifactStagingDirectory)/ --version $(BuildId).0.0 ./chart/ 
    workingDirectory: '$(Build.SourcesDirectory)/AppA.WebApi'

- task: AzureCLI@2
  inputs:
    azureSubscription: 'azure-dev-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: 'az acr helm push -n $(registryName) $(Build.ArtifactStagingDirectory)/*.tgz'
    useGlobalConfig: true

```

Helm 3 needs the environment variable "HELM_EXPERIMENTAL_OCI: 1" defined or it won't work, so just put it there...

I'm using the Azure Pipeline BuildId to tag the docker image and also to set the Helm Chart version so the developer doesn't need to maintain an strict control about which version is which.
That's a simple Helm versioning strategy. Using a 1-1 versioning just keep the chart version in sync with your actual application.   
This approach makes version bumping very easy (you bump everything up) and also allows you to quickly track what application version is deployed on your cluster (same as chart version).  
If you're interested in helm versioning strategies you can read it more here: https://codefresh.io/docs/docs/new-helm/helm-best-practices/

Let me detail a little bit what are the steps I'm doing inside the pipeline:

- Login to an Azure Container Registry.
- Build and push the image into the ACR, I'm doing both things which just one command using the AZ CLI: _az acr build_. 
- Install Helm 3. 
- Replace tokens. 
- Create the Helm Chart from the application _/chart_ folder using the Helm CLI.
- Push the Helm Chart into ACR using the AZ CLI.


> _Why I'm using the Replace Tokens task in my pipeline?_ (https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens)   
> That's because in the Kubernetes Deployment file I defined the image i'm using like this: 
```yaml
    containers:
    - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:#{tag}#" 
```
> As I previously said I'm using the Azure Pipelines BuildId as my image tag and I only know the value after the pipeline has started, that's why I'm using the replace tokens task. Basically I'm replacing the _#{tag}#_ placeholder with the correct value while the Azure Pipeline is running.


## Step 3 - Install Flux and the Helm Operator
---

What is Flux? Flux is a tool that automatically ensures that the state of your Kubernetes cluster matches the configuration you’ve supplied in Git. It uses an operator in the cluster to trigger deployments inside Kubernetes, which means that you don’t need a separate continuous delivery tool.  

What is the Helm Operator?  The Helm Operator is a Kubernetes operator, allowing one to declaratively manage Helm chart releases. Combined with Flux this can be utilized to automate helm charts releases in a GitOps manner.

_You can read more about Flux and Helm Operator here: https://docs.fluxcd.io/en/1.20.0/_

That's how it works:

![flux-helm-diagram](/img/fluxcd-helm-operator-diagram.png)


Let's enumerate the steps to install both Flux and the Helm Operator into the Kubernetes cluster.

1 - Create the Flux namespace:

```bash
kubectl create ns flux
```

2 - Add the Flux repository

```bash
helm repo add fluxcd https://charts.fluxcd.io
```

3 - Apply the Helm Operator CRD

```bash
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```

4 - Install Flux

```bash
helm upgrade -i flux fluxcd/flux --set git.url=git@ssh.dev.azure.com:v3/cpn/Provisioning/Manifests  --set registry.acr.enabled=true --namespace flux
```
- The **git.url** attribute needs to be the **Azure DevOps Repository URL** that we want Flux to monitor. Flux is going to connect to the Git repo through **SSH**.   
In our example that would be our **centralized repository.**

- The **registry.acr.enabled** attribute is needed because we are using images hosted in ACR.

5 - Giving Flux read and write access to the Azure DevOps repository

In the previous step we have configured Flux to monitor a concrete Azure DevOps git repository.
Now we need to give Flux access to the git repository.   
To facilitate that you will need to add an SSH key into your Azure DevOps account.
This is pretty straight-forward as Flux generates a SSH key and logs the public key at startup. Find the SSH public key by running:

asdasd

In order to sync your cluster state with git you need to copy the public key and add it as an SSH key in your Azure DevOps account.

<FOTO>

6 - Install the Flux Operator

That step is the most problematic one, that's because there are some authentication problems between the Flux Operator and ACR.

I'm not the only one facing the same problem, so I have stolen the solution found here:
https://gaunacode.com/configuring-flux-to-use-helm-charts-from-azure-container-registry 

Basically when we install the Helm-Operator we have to provide the following attributes:

- The ACR name.
- The client id of a service principal that has at least AcrPull rights to your ACR.
- The client secret of that service principal.
- The url of your ACR Helm endpoint. It must end with a slash.

So the command is going to be something like:

```bash
helm upgrade -i helm-operator fluxcd/helm-operator --set git.ssh.secretName=flux-git-deploy --set helm.versions=v3 --namespace flux --set configureRepositories.enable=true --set configureRepositories.repositories[0].name=acrgitopsdev,configureRepositories.repositories[0].url=https://acrgitopsdev.azurecr.io/helm/v1/repo,configureRepositories.repositories[0].username=put-your-sp-id-here,configureRepositories.repositories[0].password=put-your-sp-password-here
```

> You can do all those steps manually or create an Azure Pipelines to orchestrate all those commands.   
To be honest it's really easy to build the pipeline once you know all the commands you need to execute, that's why for now i'm not going to waste time doing it.


## Step 4 - Create the application config file
---

Right now we have: 
- The application Helm Chart hosted in ACR
- Flux monitoring a repository named Manifests in the Provisioning project.

The last step is to create the application config file and push it into the "Manifests" repository and see if Flux is capable of deploying our application in our Kubernetes cluster.


```yaml

apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: appa
spec:
  chart:
    repository: https://acrgitopsdev.azurecr.io/helm/v1/repo/
    name: appawebapi
    version: "158.0.0"
  values:
    imagePullSecrets:
      - name: acr-gitdevops

```

In the most basic file the only thing I need to specify in the config file is the Chart repository URL, the Chart name and the Chart version I want to deploy.
There're some more options available to tinker with it, if you're interested you can read about it here: https://docs.fluxcd.io/projects/helm-operator/en/1.0.0-rc9/references/helmrelease-custom-resource.html

In my case I'm defining also a secret and that's because Kubernetes uses an image pull secret to store information needed to authenticate to the container registry. **If you're using AKS you don't need to do it.**   
If you want to read more about it: https://docs.microsoft.com/es-es/azure/container-registry/container-registry-auth-kubernetes

And that's it! We're done!


## Conclusion
---

As I said before what I have built is more a proof of concept than a production ready GitOps process.
But I have achieve what I wanted to build with ease.
With the GitOps flow we have built every application created is containerized and pushed into ACR using Azure Pipelines and then we have a centralized repository where we define what applications we want to deploy into our Kubernetes cluster.
Helm and Flux are our main tools to automatically deploy and keep the configuration in-sync with the apps deployed in the kubernetes cluster.

