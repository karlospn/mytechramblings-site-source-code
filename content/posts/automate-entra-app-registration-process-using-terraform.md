---
title: "Trying to automate Microsoft Entra ID App Registration process using Terraform"
date: 2023-11-20T10:00:48+01:00
description: "The purpose of this post is to test the App Registration process using the latest version of the Terraform provider for Microsoft Entra ID. To achieve this, we're going to register multiple apps on Microsoft Entra ID and make them interact with each other using a couple of different OAuth2 authorization flows."
tags: ["azure", "terraform", "auth", "entra", "iac"]
draft: false
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/testing-entra-app-registration-process-using-terraform).

If you're looking to secure your applications, Microsoft Entra ID (aka. Azure Active Directory) is a great option.    
However, registering apps manually can pose several challenges. As your organization expands and the number of applications grows, managing them manually can become time-consuming and increase the risk of inconsistencies between them.

A more efficient alternative is to use Infrastructure as Code (IaC) tools, such as Terraform. Terraform allows you to manage your Entra apps in a more predictable way. Moreover, you can seamlessly integrate Terraform with a CI/CD process, further automating the registration process.

Terraform already has an official Entra provider written by Microsoft itself.
 - https://registry.terraform.io/providers/hashicorp/azuread/latest    

In this post, I want to explore its capabilities, and I'll attempt to construct a fairly common authorization scenario. To achieve this, I'm going to register multiple apps on Microsoft Entra ID and have them interact with each other using a couple of different OAuth2 authorization flows.

> _This post has been written using the version **2.45.0** of the Terraform provider for Microsoft Entra ID.     
> Keep in mind that when you read this post, there might be a more recent version available, and certain resources may have undergone changes or experienced some kind of breaking change._

# **Scenario Details**

The scenario we're going to build in this post is the following one:

- **Payments API**
  - It exposes 2 scopes:
    - A ``payment.write`` scope. It requires an admin consent to be used.
    - A ``payment.read`` scope. 
  - It has 2 app roles:
    - A ``Reader`` role. It can be assigned to both users and applications.
    - An ``Admin`` role. It can be assigned to both users and applications.

- **Bookings API**
  - Consumes the Payments API through a **Client Credentials** flow.
    - Acquires an access token from the Entra `/token` endpoint and uses it to gain access to the Payments API.
  - It has the Payments API ``Reader`` Role assigned.

- **FrontEnd SPA**
  - A Single Page Application that utilizes the **Authorization Code flow with PKCE** to acquire an access token and make calls to the Payments API.
  - The FrontEnd SPA app is granted permission only to request the `payment.read` scope from the Payments API.

- **User Permissions**
  - Two users are registered on my Microsoft Entra ID tenant:
    - John and Jane.
  - Jane holds a `Reader` role in the Payments API app.
  - John possesses an `Admin` role in the Payments API app.

# **OAuth2 Authorization diagrams**

The following diagrams shows the authorization flows we're going to implement in this post.

- **The Bookings API uses a Client Credentials Flow to acquire an access token from Microsoft Entra ID, utilizing it to invoke various methods within the Payments API.**

![testing-entra-with-terraform-client-credentials](/img/testing-entra-with-terraform-client-credentials.png)

- **The FrontEnd SPA uses an Authorization Code flow with PKCE to acquire a token from Microsoft Entra ID, utilizing it to invoke various methods within the Payments API.**

![testing-entra-with-terraform-auth-code-flow](/img/testing-entra-with-terraform-auth-code-flow.png)

# **1. Configure the Microsoft Entra ID Terraform provider**

> **_As of today (11/18/2023), the Terraform provider for Microsoft Entra ID still has its former name: ``azuread``._**

The first step is to configure the Terraform Provider for Microsoft Entra ID.   

There are multiple ways to setup the ``azuread`` Terraform provider. It supports a number of different methods for authenticating to Entra:

- Using the Azure CLI.
- Using a Managed Identity.
- Using a Service Principal and a Client Certificate.
- Using a Service Principal and a Client Secret.
- Using a Service Principal and OpenID Connect.

In this post, I'll be using the **Service Principal + Client Secret** method. This approach is well-suited for a potential CI/CD implementation in the future, and it's quite easy to set up. 

Essentially, we're registering a "master app" on Entra. This "master app" is nothing more than an application with permissions to create other apps.

To create this "master app", we'll manually register an app in Entra with the following permissions for `Microsoft.Graph`:
- ``Application.ReadWrite.All``
- ``AppRoleAssignment.ReadWrite.All``
- ``User.Read.All``

![testing-entra-with-terraform-master-app-permissions](/img/testing-entra-with-terraform-master-app-permissions.png)

Additionally, you need to create a new client secret for this app.

![testing-entra-with-terraform-master-app-secret](/img/testing-entra-with-terraform-master-app-secret.png)

Once we have completed these steps, we are ready to configure the Terraform Provider using the "master app" client_id, client_secret, and our Microsoft Entra Tenant ID.   

There are a a couple of ways to configure the Provider:

- Using environment variables (the recommended way).

```shell
$ export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
$ export ARM_CLIENT_SECRET="MyCl1eNtSeCr3t"
$ export ARM_TENANT_ID="10000000-2000-3000-4000-500000000000"
```

```yaml
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45.0"
    }
  }
}

provider "azuread" {}
```

- Configuring the provider directly using variables.

```yaml
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45.0"
    }
  }
}

provider "azuread" {
  client_id     = var.master_application_client_id
  client_secret = var.master_application_client_secret
  tenant_id     = var.tenant_id
}
```

At this point running either ``terraform plan`` or ``terraform apply`` should allow Terraform to authenticate to your Entra Tenant using the provided Client ID and Client Secret.

# **2. Create the Payments API application**

Next step is to create the Payments API using Terraform. This API has the following configuration:
- It exposes 2 scopes : ``payment.write`` and ``payment.read``.
  - The ``payment.write`` scope requires an admin consent to be used, while the ``payment.read`` scope does not.
- It has 2 application roles: ``Reader`` and ``Admin``.
  - Both app roles can be assigned to Entra users and applications.


```yaml
resource "random_uuid" "payments_write_scope_id" {}
resource "random_uuid" "payments_read_scope_id" {}
resource "random_uuid" "payments_admin_app_role_id" {}
resource "random_uuid" "payments_reader_app_role_id" {}

resource "azuread_application" "payments_api_application" {
    
    display_name     = "payments-api"
    identifier_uris  = ["api://payments"]
    owners           = [data.azuread_client_config.current.object_id]

    api {
        requested_access_token_version = 2

        oauth2_permission_scope {
            admin_consent_description  = "Allow the application to access the commit payment methods"
            admin_consent_display_name = "payment.write"
            enabled                    = true
            id                         = random_uuid.payments_write_scope_id.result
            type                       = "Admin"
            user_consent_description  = "Allow the application to access the commit payment methods"
            user_consent_display_name  = "payment.write"
            value                      = "payment.write"
        }

        oauth2_permission_scope {
            admin_consent_description  = "Allow the application to access the read payment methods"
            admin_consent_display_name = "payment.read"
            enabled                    = true
            id                         = random_uuid.payments_read_scope_id.result
            type                       = "User"
            user_consent_description   = "Allow the application to access the read payment methods"
            user_consent_display_name  = "payment.read"
            value                      = "payment.read"
        }
    }

    app_role {
        allowed_member_types = ["User", "Application"]
        description          = "Can read and make payments"
        display_name         = "Admin"
        enabled              = true
        id                   = random_uuid.payments_admin_app_role_id.result
        value                = "Admin"
    }

    app_role {
        allowed_member_types = ["User", "Application"]
        description          = "Can only read payments"
        display_name         = "Reader"
        enabled              = true
        id                   = random_uuid.payments_reader_app_role_id.result
        value                = "Reader"
    }
}

resource "azuread_service_principal" "payments_sp" {
  client_id                    = azuread_application.payments_api_application.client_id
  app_role_assignment_required = false
  owners                       = [data.azuread_client_config.current.object_id]
  tags                         = ["payments", "api"]
}
```

To be honest, if you take a look at what we were trying to build and at the code above, it is pretty self-explanatory what we're doing.    

The only thing worth mentioning here is the fact that, apart from creating an Entra app, we're also creating a Service Principal for the Payments API app. If you want to know why, then go read the next section because we'll put it to work.

# **3. Add user permissions on the Payments API**

Now that we've created the Payments API application on Entra, it's time to assign permissions to some of my Entra users. Specifically:

- Two users are registered on my Entra:
    - John.
    - Jane.
- Jane holds a ``Reader`` role in the Payments API app.
- John possesses an ``Admin`` role in the Payments API app.

```yaml
data "azuread_user" "jane_user" {
  user_principal_name = "jane@carlosponsnoutlook.onmicrosoft.com"
}

data "azuread_user" "john_user" {
  user_principal_name = "john@carlosponsnoutlook.onmicrosoft.com"
}

resource "azuread_app_role_assignment" "jane_payments_api_role_assignment" {
  app_role_id         = azuread_application.payments_api_application.app_role_ids["Reader"]
  principal_object_id = data.azuread_user.jane_user.object_id
  resource_object_id  = azuread_service_principal.payments_sp.object_id
}

resource "azuread_app_role_assignment" "john_payments_api_role_assignment" {
  app_role_id         = azuread_application.payments_api_application.app_role_ids["Admin"]
  principal_object_id = data.azuread_user.john_user.object_id
  resource_object_id  = azuread_service_principal.payments_sp.object_id
}
```

Not much to say here. I'm simply utilizing the ``azuread_user`` resource to retrieve the users within my Entra Tenant and the ``azuread_app_role_assignment`` resource to assign them one of the Payments API app roles.

# **4. Create the Bookings API application**

The Bookings API has the following configuration:

- Consumes the Payments API through a **Client Credentials** flow.
- Acquires an access token from the Entra `/token` endpoint and uses it to make calls to the Payments API.
- It has the Payments API ``Reader`` Role assigned.


```yaml
resource "azuread_application" "bookings_api_application" {
    
    display_name     = "bookings-api"
    identifier_uris  = ["api://bookings"]
    owners           = [data.azuread_client_config.current.object_id]

    api {
        requested_access_token_version = 2
    }

    required_resource_access {
        resource_app_id = azuread_application.payments_api_application.client_id

        resource_access {
            id   = azuread_application.payments_api_application.app_role_ids["Reader"]
            type = "Role"
        }
    }
}

resource "azuread_application_password" "bookings_api_pwd" {
  application_id        = azuread_application.bookings_api_application.id
  display_name          = "Terraform Managed Password"
  end_date              = "2099-01-01T01:02:03Z"
}

resource "azuread_service_principal" "bookings_sp" {
  client_id                    = azuread_application.bookings_api_application.client_id
  app_role_assignment_required = false
  owners                       = [data.azuread_client_config.current.object_id]
  tags                         = ["bookings", "api"]
}
```

In addition to creating the app, we're using the ``azuread_application_password`` resource to create a client secret to test the Client Credentials flow.

Keep in mind, that Entra apps are authorized to call APIs when they are granted permissions by user/admins as part of the consent process. Therefore, it is necessary to manually grant admin consent for the Bookings API to call the Payments API.

![testing-entra-with-terraform-admin-grant-consent](/img/testing-entra-with-terraform-admin-grant-consent.png)

There is no native functionality in the ``azuread`` Terraform Provider to grant admin consent to an API permission. 

A workaround can be built using the Terraform ``null_resource`` resource and the AZ CLI. We can use both to invoke the ``az ad app permission admin-consent`` command. However, to successfully run this command, the Service Principal needs to have a ``Global Administrator`` role assigned.    

This role is one of the most powerful ones, it is capable of managing all aspects of Microsoft Entra. I am not very comfortable assigning such a powerful role to a Service Principal, I prefer that an Entra Admin manually grants admin consent.

Anyways, after granting consent to the Bookings API to call the Payments API, let’s test and see if the Client Credentials flow works properly.    
With the following command, I'm going to request an access token using the Bookings API client ID and client secret.

```text
curl -k -X POST \
  https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/token \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=e896bbda-6a3b-4309-a140-41c816049f09&client_secret=uPd8Q~D.Ntc1D2MbzTuJJwdIwxuFKVARezmURduC&scope=api://payments/.default'
```

> **What’s of the .default scope?**  
> The .default scope is a built-in scope for every Entra application and it refers to the static list of permissions configured on the application registration.     
> You must specify the .default scope in a Client Credentials flow, you cannot ask for a concrete one.

And it returns an access token with the following attributes:

```json
{
  "aud": "2034f5fd-8e85-48f0-8282-983ee03319b5",
  "iss": "https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/v2.0",
  "iat": 1699874572,
  "nbf": 1699874572,
  "exp": 1699878472,
  "aio": "E2VgYLjBYnn7vPbZdxopTy/kGH5xqHmfEixWUtEp6W7OHiVbmQoA",
  "azp": "e896bbda-6a3b-4309-a140-41c816049f09",
  "azpacr": "1",
  "oid": "013f9e69-d867-42bd-9ea1-6b3dda3dd2df",
  "rh": "0.AR8A4nEGijA6ME2cua1wm5x0Sv31NCCFjvBIgoKYPuAzGbUfAAA.",
  "roles": [
    "Reader"
  ],
  "sub": "013f9e69-d867-42bd-9ea1-6b3dda3dd2df",
  "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
  "uti": "ZQMAaXC8q028-QpMeA5UAA",
  "ver": "2.0"
}
```

If we examine the received access token, we can verify that it includes the following attributes:

- The ``iss`` (issuer) should match your Entra ID tenant, formatted as: https://login.microsoftonline.com/{tenant-id}/v2.0"
- The ``aud`` (audience) should match the client ID of the Payments API.
- It must possess a ``Reader`` role.

Now with this access token we should be able to successfully call the Payments API.

# **5. Create  the FrontEnd SPA application**

The FrontEnd SPA has the following configuration:

- It's a public Single Page Application.
- Utilizes an **Authorization Code flow with PKCE** to obtain an access token and uses it to make calls to the Payments API.
- The FrontEnd SPA app has permission only to ask for the `payment.read` scope.

```yaml
resource "azuread_application" "frontend_spa_application" {   
    display_name     = "frontend-spa"
    owners           = [data.azuread_client_config.current.object_id]

    single_page_application {
        redirect_uris = ["https://oidcdebugger.com/debug"]
    }

    required_resource_access {
        resource_app_id = azuread_application.payments_api_application.client_id

        resource_access {
            id   = azuread_application.payments_api_application.oauth2_permission_scope_ids["payment.read"]
            type = "Scope"
        }
    }
}

resource "azuread_service_principal" "frontend_spa_sp" {
  client_id                    = azuread_application.frontend_spa_application.client_id
  app_role_assignment_required = false
  owners                       = [data.azuread_client_config.current.object_id]
  tags                         = ["frontend", "spa"]
}


resource "azuread_application_pre_authorized" "frontend_spa_preauthorized" {
  application_id       = azuread_application.payments_api_application.id
  authorized_client_id = azuread_application.frontend_spa_application.client_id

  permission_ids = [
    random_uuid.payments_read_scope_id.result
  ]
}
```
As you can see, the above code is self-explanatory, but there are a couple of things worth mentioning:

- The ``redirect_uris`` value is set to ``https://oidcdebugger.com/debug``, that's because this site will allow me to easily test an Authorization Code Flow with PKCE. It is obvious that in a real application, you need to set it up with a proper URI.
- We're also using the ``azuread_application_pre_authorized`` resource. This resource allows us to pre-authorize the usage of the ``payment.read`` scope from the Payments API by the FrontEnd SPA so that it won't require a user consent the first time a user logs in.   

If you don't pre-authorize the FrontEnd SPA to use the ``payment.read`` scope from the Payments API, then when the user logs in, it will be asked for its consent. The next screenshot shows what the user will see the first time they log in to the FrontEnd SPA.

![testing-entra-with-terraform-user-grant-permissions](/img/testing-entra-with-terraform-user-grant-permissions.png)

This time it is not necessary to manually grant admin consent for the FrontEnd SPA to call the Payments API, because the ``payment.read`` scope doesn't require an admin consent.

![testing-entra-with-terraform-user-grant-consent](/img/testing-entra-with-terraform-user-grant-consent.png)


To easily test an Authorization Code flow with PKCE we're going to use this website:

- https://oidcdebugger.com/

Simply fill out the form with the appropriate fields and select code flow with PKCE. 

![testing-entra-with-terraform-oidcdebugger](/img/testing-entra-with-terraform-oidcdebugger.png)

Once you click submit, it will redirect you to Entra, where you'll need to log in. In our case, for example, if we log in as Jane, we will obtain the following access token.

```json
{
  "aud": "c90ce23f-68ad-4e29-a134-13e40676b5ea",
  "iss": "https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/v2.0",
  "iat": 1700053398,
  "nbf": 1700053398,
  "exp": 1700058919,
  "aio": "AUQAu/8VAAAAM9n6UAtue+BBe0M/eeOhHxJJIJK6nvxZuuxtTDKYKMLIad0Oa8v2m7hSUlHeBE2WOpjHACzbvkEnb7V+OJpzcw==",
  "azp": "e13242fb-4b3a-4337-a92f-d9f435dcc43b",
  "azpacr": "0",
  "name": "jane",
  "oid": "0a92ffc5-9551-47a0-baf0-7f3352eac015",
  "preferred_username": "jane@carlosponsnoutlook.onmicrosoft.com",
  "rh": "0.AR8A4nEGijA6ME2cua1wm5x0Sj_iDMmtaClOoTQT5AZ2teofAAc.",
  "roles": [
    "Reader"
  ],
  "scp": "payment.read",
  "sub": "9-U8mZ7iqRPd31tYsO0d4sGj4sYwd4-RjVfP361kvTg",
  "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
  "uti": "bYwFTTO1xkqWHz79FUyVAA",
  "ver": "2.0"
}
```
If we inspect the received access token from Entra, you can verify that the token includes the following attributes:

- The ``iss`` (issuer) should match your Entra ID tenant, formatted as: https://login.microsoftonline.com/{tenant-id}/v2.0"
- The ``aud`` (audience) should match the client ID of the Payments API.
- It must have a "Reader" role, as Jane is assigned the "Reader" role in the Payments API. If you log in as John, you will possess an "Admin" role instead of a "Reader" role.

Here's an access token acquired by John. As you can see, it possesses the "Admin" role.
```json
{
  "aud": "c90ce23f-68ad-4e29-a134-13e40676b5ea",
  "iss": "https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/v2.0",
  "iat": 1700052943,
  "nbf": 1700052943,
  "exp": 1700058281,
  "aio": "AUQAu/8VAAAAtkQjFl+KJ7+RU1dnPX+9dpbUUOBl/m/o28BJ+2HRlKHt0RJL9ISFTJkcgI23anYCYFLQ2Ppii6I+F4C3ISaKeA==",
  "azp": "e13242fb-4b3a-4337-a92f-d9f435dcc43b",
  "azpacr": "0",
  "name": "john",
  "oid": "5dcd9121-762e-4e07-a53b-42f70a807b4c",
  "preferred_username": "john@carlosponsnoutlook.onmicrosoft.com",
  "rh": "0.AR8A4nEGijA6ME2cua1wm5x0Sj_iDMmtaClOoTQT5AZ2teofALY.",
  "roles": [
    "Admin"
  ],
  "scp": "payment.read",
  "sub": "IuPfLR7qcegr18ZiggU9j1Wq77ph3rpDdmiww5GT_iw",
  "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
  "uti": "YJ615gnt8kiZhi9ew4RoAA",
  "ver": "2.0"
}
```

# **Closing thoughts**

As you have seen in this post, we have successfully implemented both authorization scenarios using Microsoft Entra ID and Terraform. We haven't built anything overly complex; in fact, both scenarios were quite simple. However, many times, we don't need to create elaborate authorization setups; using the basics properly is often more than enough.

For those of you who have been following my blog for some time, you may know that I conducted a similar test around three years ago when the Terraform provider for AAD was in version 1.1.1. The results were quite disappointing because setting up a simple scenario like the one we have seen in this post was practically impossible at that time. Therefore, I can conclude by saying that this provider has evolved positively. 

Today, if I had to use Microsoft Entra ID as my IDP, I would undoubtedly use this Terraform Provider and deploy changes through a CI/CD flow.
