---
title: "Building an Async HTTP Api with Azure Durable Functions and Python"
date: 2021-12-15T09:00:04+01:00
draft: true
tags: ["azure", "functions", "serverless", "python"]
description: "The async HTTP API pattern addresses the problem of coordinating the state of long-running operations with external clients. Azure Durable Functions provides built-in support for this pattern and in this post I'm going to show you how to implement it using Python."
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/az-durable-func-async-http-api-python).


In today's post I'll be showing you how to implement the Async HTTP Api pattern with Azure Durable functions. 

Also, most of the online examples and documentation about Durable Functions are written with csharp so instead of that I have decided to use Python.   

To begin with, let's do a brief explanation about a few key concepts around Azure Durable Functions.

# 1. Azure Durable Functions

For those unaware of Azure Durable Functions, it is an extension of Azure Functions that lets you write stateful functions in a serverless compute environment.   
The extension lets you define stateful workflows by writing orchestrator functions and stateful entities.

The 4 main concepts you should know about are:

- **Client functions**

The client function is responsible for starting, stopping and monitoring the orchestrator functions.

- **Orchestrator functions**

This function is the heart when building a durable function solution. In this function you write your workflow in code. The workflow can consist of code statements calling other functions like activity functions, other orchestration functions or even wait for other events to occur.

An orchestration is started by a client function, a function that on its turn can be triggered by a message in a queue, an HTTP request, or any other trigger mechanism you are familiar with. 

Every instance of an orchestration function will have an instance identifier, which can be auto-generated or user-generated. 

- **Activity functions**

 These activity functions are the basic unit of work in a durable function solution.    
 Each activity function executes one task and can be anything you want. 

- **Entity functions**

Entity functions define operations for reading and updating small pieces of state. We often refer to these stateful entities as durable entities.   

Like orchestrator functions, entity functions are functions with a special trigger type, entity trigger. They can also be invoked from client functions or from orchestrator functions. 

Unlike orchestrator functions, entity functions do not have any specific code constraints. Entity functions also manage state explicitly rather than implicitly representing state via control flow.



# 2. Problem trying to solve with Azure Durable functions

In this section I'll like to talk a little bit about what I was trying to solve using Durable functions and why Durable functions was a good fit.

One of my clients has an HTTP API that uses an Azure Function as its backend, the function takes a few parameters via querystring, runs a query against an Azure Storage Table and returns a result. It's a really simple setup.

![audit-api](/img/audit-function-durable.png)

The problem with this approach lies in the fact that the query takes between 3 and 7 minutes to complete, so most of the time the HTTP function times out because 230 seconds is the maximum amount of time that an HTTP triggered function can take to respond to a request. This is because of the idle timeout of Azure Load Balancer. 

For longer processing times, I needed to consider using an async pattern or defer the actual work and return an immediate response.   
And Azure Durable Functions provides built-in support for the async http api pattern.

# 3. Async HTTP API Pattern

The async HTTP API pattern addresses the problem of coordinating the state of long-running operations with external clients.  

A common way to implement this pattern is by having an HTTP endpoint trigger the long-running action. Then, redirect the client to a status endpoint that the client polls to learn when the operation is finished.

![async-http-pattern](/img/async-http-api-pattern.png)

# 4. Scenario
The idea behind what we're going to build is the following one:

- The customer submits a job by calling an Azure Function that has an HTTP endpoint trigger. This is the Client Function.
- The submitted job is going to be picked up by the Orchestrator Function and it will call the Activity Function.
- The Activity Function is going to run the query against the Azure Storage Table and return the result.
- There is going to be an extra Azure Function that queries the orchestrator function and retrieves the status and the result of a given job.

Here's a diagram:

![async-http-api](/img/durable-function-async-http-api.png)

- ``client-function``: submits the job that needs to be executed.
- ``get-status-function``: it is used to retrieve the status and the result of a given job.
- ``orchestrator-function``: Unwraps the parameters from the submitted job and calls the activity function.
- ``query-storage-account-activity-function``: Runs a custom query in an Azure Storage Table.


# 5. Implementation

The components of the solution are described in the previous section. Now let's start building them.

# client-function

This is an Http triggered function. 

The function receives 2 parameters via QueryString (those parameters are needed to build the query against the Storage Table) and validates both of them.

Afterwards, it creates a ``DurableOrchestrationClient`` and starts a new orchestration instance using the ``start_new`` method.   
The ``start_new`` method takes 3 parameters:
  - Name: The name of the orchestrator function to schedule.
  - InstanceId: (Optional) The unique ID of the instance. If you don't specify this parameter, the method uses a random ID.
  - Input: Any JSON-serializable data that should be passed as the input to the orchestrator function.

And lastly, it builds the URL from where the client can retrieve the status and the result of the submitted job.


```python
import azure.functions as func
import azure.durable_functions as df
import json
import dateutil.parser
from urllib.parse import urlparse

async def main(req: func.HttpRequest, starter: str) -> func.HttpResponse:

    response = { }
    headers = { "Content-Type": "application/json" }

    start_date = req.params.get('s')
    end_date = req.params.get('e')
    
    if start_date:
        try:
          start_date = dateutil.parser.parse(start_date, dayfirst=True)
        except:
          response['error'] = "Invalid start date format."
          return func.HttpResponse(json.dumps(response) ,headers=headers, status_code=400 )
    else:
        response['error'] = "Empty start date."
        return func.HttpResponse(json.dumps(response) ,headers=headers, status_code=400 )

    if end_date:
        try:
          end_date = dateutil.parser.parse(end_date, dayfirst=True)
        except:
          response['error'] = "Invalid end date format."
          return func.HttpResponse(json.dumps(response) ,headers=headers, status_code=400 )
    else:
        response['error'] = "Empty end date."
        return func.HttpResponse(json.dumps(response) ,headers=headers, status_code=400 )

    delta = end_date - start_date
    if delta.days < 1:
        response['error'] = "Invalid date range."
        return func.HttpResponse(json.dumps(response) ,headers=headers, status_code=400 )

    
    client = df.DurableOrchestrationClient(starter)

    parameters = {
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d")
    }

    instance_id = await client.start_new('orchestrator-function', None, parameters)

    status_uri = build_api_url(urlparse(req.url).scheme, req.headers.get("host"), instance_id)
    response["statusUri"] = status_uri
    return func.HttpResponse(json.dumps(response), headers=headers, status_code=200 )

def build_api_url(scheme, host, instance_id):
    return f"{scheme}://{host}/api/status/{instance_id}"
```
# get-status-function

## Is this function really needed?

Before discussing the implementation of this Azure Function, I think it's important to talk a bit about if this function is really needed or not.

Most of the examples that you are going to find online don't use a custom function to retrieve the status of the jobs. Instead of that, they use the ``create_check_status_response`` method on the Client Function.  

That means that in the previous section I could have built the client function like this:

```python
    ...
    client = df.DurableOrchestrationClient(starter)

    parameters = {
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d")
    }

    instance_id = await client.start_new('orchestrator-function', None, parameters)
    return client.create_check_status_response(req, instance_id)
```

And that's how the function would have responded:

```javascript
{
  "id": "b78244c9e19f43e89e7b1578f711940d",
  "statusQueryGetUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d?taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw==",
  "sendEventPostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d/raiseEvent/{eventName}?taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw==",
  "terminatePostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d/terminate?reason={text}&taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw==",
  "rewindPostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d/rewind?reason={text}&taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw==",
  "purgeHistoryDeleteUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d?taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw==",
  "restartPostUri": "http://localhost:7071/runtime/webhooks/durabletask/instances/b78244c9e19f43e89e7b1578f711940d/restart?taskHub=TestHubName&connection=Storage&code=V/xX4rraZaheGNwbMePOhhGtyHxGsgks4Q36WKBdS3WfiOjD5Sb5lw=="
}
```
As you can see the ``create_check_status_response`` method returns a response that contains links to the various management operations that can be invoked on an orchestration instance.   
These operations include querying the orchestration status, raising events or terminating the orchestration.

>So, why I'm building an extra Azure Function to retrieve the status of a job instead of directly using the ``create_check_status_response`` method?

This is because the general public shouldn’t be able to query the Orchestration status endpoint directly.   
As you can see the response payload includes the orchestrator management endpoints with their corresponding keys, by providing that, you would allow the clients not only to get the status of a running instance, but also to get the execution history, send external events to the orchestration instance or terminate that instance.   

Basically if you don’t want your clients messing around with the heart of the function, you need to expose another azure function that returns only the minimum information required.

## Implementation

The ``get-status-function`` is an Http triggered function.

This function needs the ``instance_id`` that the Client function generated when submitting the job to the orchestrator function.   

To retrieve the status and the result of a submitted job you can use the ``get_status`` method. This method queries the orchestrator function to obtain the status of the job.

The ``get_status``  returns an object with the following properties:

- Name: The name of the orchestrator function.
- InstanceId: The instance ID of the orchestration (should be the same as the instanceId input).
- CreatedTime: The time at which the orchestrator function started running.
- LastUpdatedTime: The time at which the orchestration last checkpointed.
- Input: The input of the function as a JSON value. This field isn't populated if showInput is false.
- CustomStatus: Custom orchestration status in JSON format.
- Output: The output of the function as a JSON value (if the function has completed). If the orchestrator function failed, this property includes the failure details. If the orchestrator function was terminated, this property includes the reason for the termination (if any).
- RuntimeStatus: One of the following values:
  - Pending: The instance has been scheduled but has not yet started running.
  - Running: The instance has started running.
  - Completed: The instance has completed normally.
  - ContinuedAsNew: The instance has restarted itself with a new history. This state is a transient state.
  - Failed: The instance failed with an error.
  - Terminated: The instance was stopped abruptly.
- History: The execution history of the orchestration. This field is only populated if showHistory is set to true.

```python
import azure.functions as func
import azure.durable_functions as df
import json


async def main(req: func.HttpRequest, starter: str) -> func.HttpResponse:
    client = df.DurableOrchestrationClient(starter)

    instance_id = req.route_params["id"]

    response = await client.get_status(instance_id)

    if response.instance_id is None:
        return func.HttpResponse("Job not found", status_code=404)

    return func.HttpResponse(json.dumps({
        "id": response.instance_id,
        "status": response.runtime_status.value,
        "result": response.output
    }))
```

# orchestrator-function

Nothing fancy here. The orchestrator function in this application is really simple because there is no need to orchestrate multiple activies nor build a complex workflow.

The function does the following steps:

- Retrieves the inputs send from the Client Function (These parameters are needed to build the query against the Azure Table Storage)
- Calls the activity function passing the retrieved inputs.
- Waits for the activity function to end and returns the result.

```python
import azure.durable_functions as df


def orchestrator_function(context: df.DurableOrchestrationContext):
    input = context.get_input()
    result = yield context.call_activity('query-storage-account-activity-function', {'start': input['start'], 'end': input['end']})
    return result

main = df.Orchestrator.create(orchestrator_function)
```

# query-storage-account-activity-function

This Activity Function is responsible to run the query against the Azure Table Storage.

This function retrieves the Storage Table connection string from an App Configuration with a Key Vault reference (https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview), runs the query and returns the result.

As you can see this function is a single unit of work and does not contain any reference to the durable-functions library.

```python
import json
import os
from azure.appconfiguration import AzureAppConfigurationClient
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from azure.data.tables import TableClient
from pathlib import Path

def main(request: dict) -> int:
    start = request['start']
    end   = request['end']
    
    table_conn_str = get_azure_table_connection_string()
    response = run_query(start, end, table_conn_str)

    return response

def run_query(start, end, table_conn_str) -> int:

    items = []
    client =  TableClient.from_connection_string(conn_str=table_conn_str, table_name="audit") 
    
    parameters = {
        "start": start,
        "end": end
    }
           
    query_filter = "PartitionKey ge @start and PartitionKey le @end and CertNumber ne ''"
    entities = client.query_entities(query_filter, parameters=parameters, select='identificationNumber')
    
    for entity in entities:
        items.append(entity['identificationNumber'])
        
    return len(set(items))

def get_azure_table_connection_string() -> str:
    
    defaultAzureCredential = DefaultAzureCredential()

    app_config_base_url = os.getenv('AppConfigEndpoint')
    app_config_client = AzureAppConfigurationClient(base_url=app_config_base_url, credential=defaultAzureCredential)

    keyvault_value = app_config_client.get_configuration_setting(key="storage-account-connection-string", label="async-http-api")
    url_parts = Path(json.loads(keyvault_value.value)["uri"]).parts
    vault_url = "//".join(url_parts[:2])
    kv_secret = url_parts[-1]
    kv_client = SecretClient(vault_url, defaultAzureCredential)
    secret_val = kv_client.get_secret(kv_secret).value

    return secret_val
```

# 6. Test

Everything is put in place, now let's test it.

If I try to submit a new job with a few invalid parameters, it responds with an error:

```bash
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/submit/query"
{"error": "Empty start date."}
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/submit/query?s=10/08/2021&e=abcd"
{"error": "Invalid end date format."}
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/submit/query?s=10/08/2021&e=05/08/2021"
{"error": "Invalid date range."}
```

If I try to submit a new job with valid parameters, the client function responds with the status endpoint.
```bash
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/submit/query?s=10/10/2021&e=10/12/2021"
{"statusUri": "https://func-sa-table-durable-dev.azurewebsites.net/api/status/433ebcfe85ec4012abe94dcda2aa6b00"}
```

If I query the get-status function right away, it returns that the job is still being executed.
```bash
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/status/433ebcfe85ec4012abe94dcda2aa6b00"
{"id": "433ebcfe85ec4012abe94dcda2aa6b00", "status": "Running", "result": null}
```

If I query the get-status function after a few minutes, the job has completed and we can see the result.
```bash
curl "https://func-sa-table-durable-dev.azurewebsites.net/api/status/433ebcfe85ec4012abe94dcda2aa6b00"
{"id": "433ebcfe85ec4012abe94dcda2aa6b00", "status": "Completed", "result": 119}
```

# 7. Deployment to Azure

I didn't plan to write about how to deploy these functions to Azure, but it might be useful to someone.

Here's how you can deploy them using:
- Azure DevOps pipelines
- Github Actions

## Using Azure Pipelines

```yaml
trigger: none

pool:
  vmImage: 'ubuntu-latest'

variables:
- name: azureSubscription
  value: 'cpons-demos-dev'
- name: functionAppName
  value: 'func-staccount-report-query-dev'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.8'
  displayName: 'Use Python 3.8'

- script: |
    python -m pip install --upgrade pip
    pip install --target="./.python_packages/lib/site-packages" -r ./requirements.txt
  displayName: 'Install dependencies'

- task: CopyFiles@2
  inputs:
    SourceFolder: '$(Build.SourcesDirectory)'
    Contents: '**'
    TargetFolder: '$(Build.ArtifactStagingDirectory)'

- task: ArchiveFiles@2
  inputs:
    rootFolderOrFile: '$(Build.ArtifactStagingDirectory)'
    includeRootFolder: false
    archiveType: 'zip'
    archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
    replaceExistingArchive: true

- task: AzureFunctionApp@1
  inputs:
    azureSubscription: '$(azureSubscription)'
    appType: 'functionAppLinux'
    appName: '$(functionAppName)'
    package: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
    runtimeStack: 'PYTHON|3.8'
```
## Using Github Action

```yaml
name: Deploy Durable Functions to Azure Function App

on:
  push:
    branches: [ main ]

  workflow_dispatch:

env:
  AZURE_FUNCTIONAPP_NAME: func-staccount-report-query-dev
  AZURE_FUNCTIONAPP_PACKAGE_PATH: '.'   
  PYTHON_VERSION: '3.8'
                
jobs:
  build-and-deploy:
    environment: dev
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@master

    - name: Setup Python ${{ env.PYTHON_VERSION }} Environment
      uses: actions/setup-python@v1
      with:
        python-version: ${{ env.PYTHON_VERSION }}

    - name: 'Resolve Project Dependencies Using Pip'
      shell: bash
      run: |
        pushd './${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}'
        python -m pip install --upgrade pip
        pip install -r requirements.txt --target=".python_packages/lib/site-packages"
        popd
    - name: 'Run Azure Functions Action'
      uses: Azure/functions-action@v1
      id: fa
      with:
        app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
        package: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
        publish-profile: ${{ secrets.SCM_CREDENTIALS }}
```