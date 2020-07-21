---
title: "Trying to automate AAD App Registration using Terraform"
date: 2020-07-21T21:22:44+02:00
tags: ["azure", "terraform", "AAD"]
draft: true
---

Today I want to try to use Terraform to automate the _App Registration_ step in Azure Active Directory.

If you want to secure an application Azure Active Directory is a pretty good option, but I don't want to configure my application on AAD manually, what I really want is to add a step in my CI / CD pipeline that does that for me, and for that purpose Terraform might be a good option.   

Terraform already has an official Azure Active Directory provider written by Microsoft (https://www.terraform.io/docs/providers/azuread/index.html) so in today's post I'm going to focus on trying it out.


I'm going to build the following scenario:


- **Payment API**: That's going to be our resource server.
  - It exposes 2 scopes : payment.write and payment.read
  - It has 2 application roles: Reader and Writer
    - I have  2 users in my AAD: John and Mary
      - John has a Reader role in the Payment API app
      - Mary has a Writer role in the Payment API app

- **Booking API**: 
  - Consumes the Payment API using a Client Credentials flow
    - Obtains an access_token from AAD and uses it to attain access to the Payment API
  - The booking API app has permission to ask for the payment.write and payment.read scope.

- **FrontEnd SPA**:
  - It's a _"Single Page Application"_ written in Angular..
    - Uses an implicit flow to obtain an access_token and a id_token and uses the access_token to attain access to the Payment API
  - The FrontEnd SPA app has permission only to ask for the payment.read scope

Let's begin building it. I'm going to need to register at least 3 apps. But first of all I need to configure the azuread provider.   

  

## Step 1: Create a master app
---

I'm going to use the client id and a client secret authentication method (https://www.terraform.io/docs/providers/azuread/guides/service_principal_client_secret.html) 

So the first step is going to be register a master app with permissions to create another apps There is documentation about how to it: https://www.terraform.io/docs/providers/azuread/guides/service_principal_configuration.html  

The first weird thing that you're going to find is that the provider uses the **old Azure Active Directory Graph API** instead of the Graph API.   
That's a bad sign to begin with, it means that all the most recent features probably are not doable with the provider.

But let's going forward, that's the final look after registering in AAD the app and giving it the proper permissions:

<FOTO- 1>

Now we can configure the Terraform provider using the app client_id and client_secret

```yaml

provider "azuread" {

  client_id         = "ba4d0620-0522-4ada-b0b6-0cdd8cfaeae7"
  client_secret     = "my_secret_goes_here"
  tenant_id         = "my_tenant_goes_here"
  subscription_id   = "my_subscription_id_goes_here"
}

```

## Step 2: Register the payment API
---

Next step is to create the payment API using Terraform.
The payment API has the following configuration:

  - It exposes 2 scopes : payment.write and payment.read
  - It has 2 application roles: Reader and Writer
    - I have  2 users in my AAD: John and Mary
      - John has a Reader role in the Payment API app
      - Mary has a Writer role in the Payment API app

```yaml

resource "azuread_application" "payments_api" {
    name                       = "payments_api"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = false
    type                       = "webapp/api"
    identifier_uris            = ["api://payment"]

  
    oauth2_permissions {
        admin_consent_description  = "Allow the application to access the commit payment methods"
        admin_consent_display_name = "payment.write"
        is_enabled                 = true
        type                       = "Admin"
        value                      = "payment.write"
    }

    oauth2_permissions {
        admin_consent_description  = "Allow the application to access the read payment methods"
        admin_consent_display_name = "payment.read"
        is_enabled                 = true
        type                       = "User"
        value                      = "payment.read"
        user_consent_description  = "Allow the application to access the read payment methods"
        user_consent_display_name = "payment.read"    
    }

    app_role {
        allowed_member_types = [
        "User",
        "Application",
        ]
        description  = "Can read and make payments"
        display_name = "Writer"
        is_enabled   = true
        value        = "Writer"
    }

    app_role {
        allowed_member_types = [
        "User",
        "Application",
        ]

        description  = "Can only read payments"
        display_name = "Reader"
        is_enabled   = true
        value        = "Reader"
    } 
}

resource "azuread_service_principal" "payment_sp" {
  application_id               = azuread_application.payments_api.application_id
  app_role_assignment_required = false
  tags = [ "payment"]
}
```

It's really easy to do it but there are some issues.   

The first problem is:   
I can create the application roles but **there is NO way to assign users or groups to those roles**.   
I can go into the  _"Enterprise applications"_ blade in the Azure Active Directory Portal and  assign users and groups manually to the Reader and Write role but that's a bummer...

Poking around Github (https://github.com/terraform-providers/terraform-provider-azuread) I found that it's an already know issue  ( https://github.com/terraform-providers/terraform-provider-azuread/issues/230) and the issue is because the provider is using the old AAD api...

The other problem I found is that not all the Manifest attributes are present in the provider.   
For example, I usually change the _"accessTokenAcceptedVersion"_ attribute so the token endpoint only generates tokens in V2 format _(I'm going to talk about that weird behaviour in some other rambling...)_ but I cannot do it with the provider, I have to change it again manually...


## Step 3: Register the Booking API
---

The Booking API has the following configuration:
 - Consumes the Payment API using a Client Credentials flow
    - Obtains an access_token from AAD and uses it to attain access to the Payment API
  - The booking API app has permission to ask for the payment.write and payment.read scope.

```yaml

resource "azuread_application" "booking_api" {
    name                       = "booking_api"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = false
    type                       = "webapp/api"
    identifier_uris            = ["api://booking"]

    required_resource_access {
        resource_app_id = azuread_application.payments_api.application_id
        resource_access {
            id   = tolist(azuread_application.payments_api.oauth2_permissions)[0].id
            type = "Scope"
        }

        resource_access {
            id   = tolist(azuread_application.payments_api.oauth2_permissions)[1].id
            type = "Scope"
        }
    }
}


resource "azuread_service_principal" "booking_sp" {
  application_id               = azuread_application.booking_api.application_id
  app_role_assignment_required = false
  tags = [ "payment"]
}

resource "azuread_application_password" "booking_api_pwd" {
  application_object_id = azuread_application.booking_api.id
  description           = "My managed password"
  value                 = "VT=uSgbTanZhyz@%nL9Hpd+Tfay_MRV#"
  end_date              = "2099-01-01T01:02:03Z"
}

```

I registered the app and also created a secret to test the client credentials flow, but I have encountered some issues in the process as well.   


The first issue is : 

Whereas in an implicit flow, you can let the user approve or deny the scopes on demand, in a client credentials flow you need to grant the client access in advance because the flow does not involve the user’s interaction. This is important to remember, since if you forget this step, you’ll get an error when making a request for an access token.

That step cannot be done using the provider either, another step that needs to be done manually...


The second issue is:

I cannot grant consent to use the resource owner scopes in an automatically way, so I need to do it manually again...


## Step 4: Register the FrontEnd SPA
---

The FrontEnd SPA has the following configuration:
  - It's a public _"Single Page Application"_ written in Angular. Uses an implicit flow to obtain an access_token and a id_token and uses the access_token to attain access to the Payment API
  - The FrontEnd SPA app has permission only to ask for the payment.read 
  scope

```yaml

resource "azuread_application" "frontend_spa" {
    name                       = "frontend_spa"
    available_to_other_tenants = false
    oauth2_allow_implicit_flow = true
    type                       = "webapp/api"
    reply_urls                 = ["http://localhost:4200"]
    logout_url                 = "http://localhost:4200/logout"
    public_client              = true
    required_resource_access {
        resource_app_id = azuread_application.payments_api.application_id
        resource_access {
            id   = tolist(azuread_application.payments_api.oauth2_permissions)[1].id
            type = "Scope"
        }
    }
}


resource "azuread_service_principal" "frontend_spa" {
  application_id               = azuread_application.frontend_spa.application_id
  app_role_assignment_required = false
  tags = [ "payment"]
}

```

I have found the same problems that I mention in step 3, but there is another one.

If you use the portal when you register an SPA you can specify that the application type is "SPA" and use the auth code flow with PKCE flow. But that option is also missing here.



## Conclusion
---
