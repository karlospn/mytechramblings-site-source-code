---
title: "How to update an Azure API Management API that is configured with a remote OpenApi definition using Terraform"
date: 2024-02-04T19:00:06+01:00
draft: true
description: "In this post, you'll learn how to create and update an Azure API Management API configured to fetch the OpenAPI definition from a downstream API using Terraform."
tags: ["azure", "terraform", "iac"]
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/how-to-update-an-azure-api-mgmt-api-that-uses-a-remote-openapi-using-terraform).

Having an endpoint that provides the content of an OpenAPI definition or to have the OpenAPI definition file directly exposed in your API is a common practice nowadays. When integrating this API with Azure API Management, it is preferable to use the remote OpenAPI definition to expose the desired API methods.   

While one could certainly copy and paste the OpenAPI document into the same folder as our Terraform files and use it to create the API Management API, it's far more maintainable to utilize the remote OpenAPI definition/endpoint for creating or updating an Azure API Management API.

But there is one problem depending on how we create the Azure API Management API with Terraform.  
 The next Terraform code snippet shows the simplest way to create an Azure API Management Api configured with a remote OpenAPI definition.

```yml
resource "azurerm_api_management_api" "apim_app_a_api" {
  name                = "app-a-webapi"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "1"
  display_name        = "app-a-webapi"
  path                = "app-a"
  protocols           = ["https"]
  service_url         = "https://${azurerm_container_app.aca_app_a.ingress[0].fqdn}"
  subscription_required = false

  import {
    content_format = "openapi+json-link"
    content_value  = "https://${azurerm_container_app.aca_app_a.ingress[0].fqdn}/swagger/v1/swagger.json"
  }
}
```

As you can see the ``import`` block is configured to retrieve content from the remote API ``/swagger/v1/swagger.json`` endpoint.

However, this scenario poses a problem.    

If someone makes a change to the downstream API that modifies the OpenApi content, such as exposing a new method or altering the signature of an existing one, running the ``terraform plan`` command will reveal that Terraform is unable to detect those changes.    
Consequently, it won't update the API in the Azure API Management to reflect the modifications.

![tf-plan](/img/api-mgmt-api-sync-openapi-no-changes.png)

To force the update on the Azure API Management API we have 2 options available:
- Using the API Management API ``revision`` property.
- Using the Terraform ``Http`` Data Resource.

For this post, I’m utilizing Azure Container Apps to host the remote API, but it could be any other service (such as Azure App Service, ACI, Azure VM, or even a service from another Cloud). The hosting platform is not relevant.

## **1. Using the revision property**

The ``revision`` property is used to make non-breaking API changes to your API, so you can test changes safely. The usual process involves deploying a new ``revision``, conducting tests, and eventually transitioning from the old to the new revision.

However, we can repurpose this property to instigate the destruction and recreation of the current exposed API on the API Management. While the original intent of the ``revision`` property is to maintain multiple revisions simultaneously, updating the ``revision`` property of the current API triggers the recreation of the current resource, serving our purpose in this context, because when the API gets recreated, it will fetch the updated content from the ``/swagger/v1/swagger.json`` endpoint.

In the following code snippet, the ``revision`` property has been modified from ``1`` to ``2``, triggering the destruction of the existing Azure Api Management Api and the creation of a new one with the updated revision property.    

Once the new Azure API Management API is created, it will fetch the latest content from the ``https://${azurerm_container_app.aca_app_a.ingress[0].fqdn}/swagger/v1/swagger.json`` endpoint.

```yml
resource "azurerm_api_management_api" "apim_app_a_api" {
  name                = "app-a-webapi"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "2"
  display_name        = "app-a-webapi"
  path                = "app-a"
  protocols           = ["https"]
  service_url         = "https://${azurerm_container_app.aca_app_a.ingress[0].fqdn}"
  subscription_required = false

  import {
    content_format = "openapi+json-link"
    content_value  = "https://${azurerm_container_app.aca_app_a.ingress[0].fqdn}/swagger/v1/swagger.json"
  }
}
```

## **2. Using the Terraform Http Data Resource**

The Terraform ``http`` data resource makes an HTTP GET request to a given URL and exports information about the response.

In the next code snippet, we utilize the Terraform data ``http`` resource to retrieve the content from the remote API OpenApi definition file.    
Subsequently, we use the output to configure the Azure API Management API.

```yml
data "http" "apim_app_b_openapi" {
  url = "https://${azurerm_container_app.aca_app_b.ingress[0].fqdn}/swagger/v1/swagger.json"
  request_headers = {
    Accept = "application/json"
  }
}

resource "azurerm_api_management_api" "apim_app_b_api" {
  name                = "app-b-webapi"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "1"
  display_name        = "app-b-webapi"
  path                = "app-b"
  protocols           = ["https"]
  service_url         = "https://${azurerm_container_app.aca_app_b.ingress[0].fqdn}"
  subscription_required = false

  import {
    content_format = "openapi+json"
    content_value  = data.http.apim_app_b_openapi.response_body
  }
}
```

# **Which option is better to use?**

> TL;DR: Go with the Terraform ``http`` data resource.

The primary advantage of utilizing the Terraform ``http`` resource is that there is no need to manually modify the ``revision`` property in our Terraform files when someone makes a change to the remote API that modifies the Swagger content.

In the first scenario, we had to manually change the ``revision`` property to trigger the recreation of the API Management API resource.

By employing the Terraform data ``http`` resource, no modifications are required in our Terraform files. The ``http`` resource **ALWAYS** fetchs the content from the remote OpenApi definition whenever a ``terraform plan`` command is executed.

The following image illustrates this concept. We've introduced a modification to the downstream API, and when the ``terraform plan`` command is executed, Terraform seamlessly detects the change and updates the API Management API, thanks to the data ``http`` resource.

![data-http-resource-tf-plan-output](/img/api-mgmt-api-sync-openapi-drift.png)

Also, as I have told you in the previous section, the ``revision`` property is not exactly built for what we're doing in here.     

The revision property is used to make non-breaking API changes so you can model and test changes safely. When ready, you can make a revision current and replace your current API.

The correct usage for the ``revision`` property is to have multiple ``azurerm_api_management_api`` resources at the same time, each one containing a different version of the revision.

The following code snippet demonstrates how to have two revisions simultaneously in your Terraform files.

```yml
resource "azurerm_api_management_api" "apim_app_b_api" {
  name                = "app-b-webapi"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "1"
  display_name        = "app-b-webapi"
  path                = "app-b"
  protocols           = ["https"]
  service_url         = "https://${azurerm_container_app.aca_app_b.ingress[0].fqdn}"
  subscription_required = false

  import {
    content_format = "openapi+json"
    content_value  = data.http.apim_app_b_openapi.response_body
  }
}

resource "azurerm_api_management_api" "apim_app_b_api_version_2" {
  name                = "app-b-webapi"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "2"
  display_name        = "app-b-webapi"
  path                = "app-b"
  protocols           = ["https"]
  service_url         = "https://${azurerm_container_app.aca_app_b.ingress[0].fqdn}"
  subscription_required = false
  source_api_id       = "${azurerm_api_management_api.apim_app_b_api.id};rev=1"
}
```

You might be wondering if you could use that approach in here. When the downstream API changes, you create a new ``azurerm_api_management_api`` resource with a new ``revision`` version, and this new ``revision`` will contain the changes made to the downstream OpenAPI definition file.

No, it doesn't work. 

When you create a new ``revision`` as shown in the code snippet above, the value of the ``content_value`` property doesn't get refreshed. Instead, it uses the ``content_value`` of the previous revision, meaning that the changes made to the downstream API are not reflected in this new ``revision``.


  
