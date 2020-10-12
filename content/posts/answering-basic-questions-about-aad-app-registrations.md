---
title: "Answering some basic questions about Azure Active Directory App Registrations"
date: 2020-10-10T17:07:50+02:00
draft: true
---

In these past years I worked with quite a few companies that uses Azure Active Directory to perform authentication and authorization on their line-of-business (LOB) applications.   
Each application you want the Azure AD to perform identity and access management for needs to be registered on AAD. Whether it's a client application like a web or mobile app, or it's an API that backs a client app, registering it establishes a trust relationship between your application and the identity provider.   

From time to time I get questions about how to register apps on AAD, those qüestions are always pretty similar and most of the time the problem lies in some misconceptions.      
Most of the questions I'm going to answer are really basic and you can find the answer in  but nonetheless I thought it might be a good idea to try to answer some of them and write them down. Besides next time that someone ask me some of those questions I can point them here.

Maybe in the future I update this post with more questions or maybe I write another post with another batch of qüestions. 

Without further ado, I'm going to start with a first batch of 10 questions. 


# Q1: What's the difference between App Registration and Enterprise Applications?


>I'll try to explain using my own words, but I think that the best explanation possible to this qüestion lies right here:  https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals

If you have developed a new application and you want to integrate it with AAD you need to register your application in **App Registrations**.    
In **App Registrations** you will configure how your application is going to behave, you will configure your reply URL, logout URL amongst others attributes. Also when you register your application AAD assigns a unique Application ID to it and allows you to add certain capabilities such as credentials, permissions or certificates.   
The **Enterprise Application** section refers mainly to applications published by other companies and that can be used within your organization. For example, if you want to federate AWS and manage SSO within your organization, you can integrate it from the **Enterprise Applications** section.

I understand that you might get confused between the **Enterprise Applications** section and the **App Registrations** section and that's because when you create an app using **app registration** there is also an **enterprise application** created in your AAD. But what really gets created in the Enterprise Application section is an instantiation of your applications in the tenant.   
You can think of the **App Registration** step like creating the definition of how you want your application to behave in AAD and in **Enterprise Application** you're seeing the instantiation of that definition in that concrete tenant.
For example if you want to assign users to an app you have registered previously on AAD you need to do it from the **Enterprise Application** blade instead of the **App Registration** section, that's because you're adding users from your tenant into that concrete instatiation of the application.   


# Q2: What's the difference between AppRoles and Scopes? And how can I add a new Scope / AppRole in my application?

When talking about scopes on AAD we're refering to the collection of permissions that an API (resource) exposes to client apps.   
These permission scopes need to be granted to client apps via consent grant.   

> If you take a look at the application manifest the scopes are being called called "oauth2permissions", 

<FOTO>

You can add new scopes to an app from the "Expose API" section

<FOTO>

If your app needs to consume another app registered on AAD it can request which scopes it needs from the "API permissions" section

<FOTO>

Role-based access control (RBAC) is a popular mechanism to enforce authorization in applications. When using RBAC, an administrator grants permissions to roles, and not to individual users or groups. The administrator can then assign roles to different users and groups to control who has access to what content and functionality.    
AppRoles are the collection of roles that an app may declare. These roles can be assigned to users, groups, or service principals.   

If you need to add a need AppRole onto your app you need to add it directly into the app manifest, there is not a Ui for it.

<FOTO>

If you want to assign users or groups onto an AppRole you need to do it from the "Enterprise Application" blade. You can't do it from the "App Registration" blade.

<FOTO>

# Q3: What's the meaning of the .default scope

The .default scope is a built-in scope for every application and it refers to the static list of permissions configured on the application registration.
It means that if we request an access token using the .default scope our users will be asked to consent to all of the configured permissions present on our Azure AD App Registration for that specific resource and in return they will get an access token containing all the permissions present.

Let me give you a couple of quick examples:

- ## Example 1
  
I have register this app with the following permissions:

<FOTO default-scope-permission-1>

And if I request a token using the default scope

```javascript
https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/authorize
?client_id=5833d6c7-1ba5-4184-9b14-050a6b5ed212
&redirect_uri=https://oidcdebugger.com/debug
&scope=https://graph.microsoft.com/.default
&response_type=token
&response_mode=form_post
&nonce=yj8o2c250eq
```

I get an access token containing all the scopes that where present on the permissions section

```javascript
{
   "aud": "https://graph.microsoft.com",
   "iss": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
   "iat": 1602519346,
   "nbf": 1602519346,
   "exp": 1602523246,
   "acct": 0,
   "acr": "1",
   "aio": "E2RgYOivOSScyJiyfV6xSPLf3RachzpdQn+WSxV/2VlcWHHi22UA",
   "amr": [
      "pwd"
   ],
   "app_displayname": "payments_api",
   "appid": "5833d6c7-1ba5-4184-9b14-050a6b5ed212",
   "appidacr": "0",
   "given_name": "doe",
   "idtyp": "user",
   "ipaddr": "10.10.10.10",
   "name": "john",
   "oid": "5dcd9121-762e-4e07-a53b-42f70a807b4c",
   "platf": "3",
   "puid": "10032000AC146311",
   "rh": "0.AR8A4nEGijA6ME2cua1wm5x0SsfWM1ilG4RBmxQFCmte0hIfALY.",
   "scp": "AccessReview.Read.All Agreement.Read.All Calendars.Read Device.Read.All profile openid email",
   "sub": "6osZh2-eBOq3NxEm8cGuMv2wSyk-6l9WoU8AIVaT_o0",
   "tenant_region_scope": "EU",
   "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
   "unique_name": "john@carlosponsnoutlook.onmicrosoft.com",
   "upn": "john@carlosponsnoutlook.onmicrosoft.com",
   "uti": "MGkcKc9B6katlAkqt_hGAA",
   "ver": "1.0",
   "wids": [
      "b79fbf4d-3ef9-4689-8143-76b194e85509"
   ],
   "xms_st": {
      "sub": "Hs7qlePul6fglrUu42vjE-e_SvJ6feFPpNi_2ltIR60"
   },
   "xms_tcdt": 1496497960
}

```

- ## Example 2:

My payment app expose these 3 scopes:

<FOTO default-scope-permission-2>


My booking app has the following permissions:

<FOTO default-scope-permission-3>


If I request a token using the default scope

```javascript
https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/authorize
?client_id=c3b8b95f-922b-4384-8ab2-a1215a9fc295
&redirect_uri=https://oidcdebugger.com/debug
&scope=api://payment/.default
&response_type=token
&response_mode=form_post
&nonce=wq2vzi24b5
```

I get an access token containing all the scopes for the payment api that where present on the permissions section

```javascript
{
   "aud": "api://payment",
   "iss": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
   "iat": 1602520037,
   "nbf": 1602520037,
   "exp": 1602523937,
   "acr": "1",
   "aio": "ASQA2/8RAAAA25pG8E+oypMn9d/2eraVByzB6g1lXvLgaYkgE53PMRQ=",
   "amr": [
      "pwd"
   ],
   "appid": "c3b8b95f-922b-4384-8ab2-a1215a9fc295",
   "appidacr": "0",
   "given_name": "doe",
   "ipaddr": "10.10.10.10",
   "name": "john",
   "oid": "5dcd9121-762e-4e07-a53b-42f70a807b4c",
   "rh": "0.AR8A4nEGijA6ME2cua1wm5x0Sl-5uMMrkoRDirKhIVqfwpUfALY.",
   "scp": "payment.read payment.write",
   "sub": "Hs7qlePul6fglrUu42vjE-e_SvJ6feFPpNi_2ltIR60",
   "tid": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
   "unique_name": "john@carlosponsnoutlook.onmicrosoft.com",
   "upn": "john@carlosponsnoutlook.onmicrosoft.com",
   "uti": "nmu3g8WcWUmK0_QXh1RNAA",
   "ver": "1.0"
}
```

> You can find a more flesh out explanation to this question right here: https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent



# Q4: What's the difference between delegated permissions and application permissions?

**Delegated permissions** are used when you want to authenticate to a WebAPI or other services with the currently logged-on user.   
This involves a physical user and a user interface.

**Application permission** are used when there is no user present. Mostly used for API to API calls. This is also used for background and daemon services.   
Unlike delegated permissions, application permissions, however, uses the app id and secret to login and always has the given permissions of the application.


# Q5: Can I add a delegated permission and also an application permission in my application?

Absolutely. Application Permissions and Delegated Permissions are completely independent of one another.    
Maybe your application has a part that requires user interaction and needs that a user is logged-in and another part that needs to be accesed by another application.


Here are a couple of tidbits to be when working with delegated and application permissions:
- In application permissions the "scp" claim is not present in the access token. You need to use the "roles" attribute.
Meanwhile for delegated permissions if the user logged has a role assigned the token will contains both claims: "scp" and "roles".
- The permissions could vary for the Graph API depending on the resource you're consuming. Take a quick look into the Microsoft documentation before assuming that the same permissions will work in both scenarios.   
For example: https://docs.microsoft.com/en-us/graph/api/drive-get?view=graph-rest-beta&tabs=http


# Q6: I have a SPA (Single Page Application) that makes a call to a protected WebAPI,  which in turn needs to call another protected WebAPI, what should I do?   

In this case you need to use the The OAuth 2.0 On-Behalf-Of flow (OBO Flow).
The idea is to propagate the delegated user identity and permissions through the request chain. For the middle-tier service to make authenticated requests to the downstream service.


# Q7: My application needs permission to call Microsoft Graph API, what I need to do?

That's probably one of the simplest things to do.   
Just register your application and in the "API permissions" section, select "Add permission" > "Microsoft Graph" > select if the permissions are going to be delegated  or application permissions > and select the permissions that you need.

# Q8: I have built a WebApi using csharp and I want to add authentication and authorization using AAD, what should I use? ADAL.NET, MSAL.NET or Microsoft.Identity.Web? 

The question between ADAL.NET and MSAL.NET is a pretty common question and to make things worse now we have another one...  

The main difference is that ADAL.NET integrates with the Azure AD for developers (v1.0) endpoint, where MSAL.NET integrates with the Microsoft identity platform (v2.0) endpoint. 

<FOTO>

If you have to choose between ADAL.net and MSAL.NET just use MSAL.NET, which is the latest generation of Microsoft authentication libraries. It will allow you to acquire tokens for users signing-in to your application with Azure AD (work and school accounts), Microsoft (personal) accounts (MSA) or Azure AD B2C.

If you have an old app that is using the AAD v1.0 endpoint probably is using the ADAL.NET library, but if you're creating a new application right now it doesn't make sense to use it.

But what about Microsoft.Identity.Web? 

> Microsoft Identity Web provides the glue between the ASP.NET Core middleware and MSAL .NET to bring a clearer, more robust developer experience, which also leverages the power of the Microsoft identity platform (formerly Azure AD v2.0 endpoint), and leverages OpenId Connect middleware, which means developers can develop applications which allow several identity providers, including integration with Azure AD B2C.

That's how Microsoft describes this new library.

It's a really recent library (version 1.0.0 was released 15 days ago) and I'm still testing it out, but from my standpoint it seems that with this library Microsoft is trying to simplify the multiple steps you needed to do in your application when using MSAL.NET.

Nonetheless if you want to use the library there is catch, and it is that **only works with .NET Core 3.1 and .NET 5**

If you want to know more about this library go here: https://github.com/AzureAD/microsoft-identity-web/wiki


Let me make a quick recap:
- ADAL.NET is obsolete. 
- If you're working with .NET Core 3.1 and you don't mind using a very recent library use Microsoft.Web.Identity. But be wary that you might need to update the nuget version to a newer version fairly quick.
- For everybody else just use MSAL.NET


# Q9: I have registered 2 apps on my AAD and I'm having troubling validating the issuer attribute from the access tokens. The issuer attribute in an access token for the app1 is "login.microsoftonline.com" and the issuer attribute in an access token for the app2 is "sts.windows.net". What's happening?

That's a weird one.

# Q10: Can I define a custom scopes and have them returned when using the client credential flow in Azure AD?

No, you can't.
What you can do is define appRoles and have them returned via the client credential flow. Also you need to specify the .default scope in a client credentials flow you cannot specify a concrete scope.

Like this:

```javascript
curl -X POST https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/oauth2/v2.0/token 
-H 'Cache-Control: no-cache' 
-H 'Content-Type: application/x-www-form-urlencoded' 
-d 'grant_type=client_credentials&client_id=c3b8b9 5f-922b-4384-8ab2-a1215a9fc295&client_secret=e-M2va9la3O1pl~x1HPQNVXKSrK-a9_iRK&scope=api://payment/.default'
```


