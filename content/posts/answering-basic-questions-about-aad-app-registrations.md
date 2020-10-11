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

Without further ado, let's start answering the first batch of qüestions.

## Q1: What's the difference between App Registration and Enterprise Applications?


>I'll try to explain using my own words, but I think that the best explanation possible to this qüestion lies right here:  https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals

If you have developed a new application and you want to integrate it with AAD you need to register your application in **App Registrations**.    
In **App Registrations** you will configure how your application is going to behave, you will configure your reply URL, logout URL amongst others attributes. Also when you register your application AAD assigns a unique Application ID to it and allows you to add certain capabilities such as credentials, permissions or certificates.   
The **Enterprise Application** section refers mainly to applications published by other companies and that can be used within your organization. For example, if you want to federate AWS and manage SSO within your organization, you can integrate it from the **Enterprise Applications** section.

I understand that you might get confused between the **Enterprise Applications** section and the **App Registrations** section and that's because when you create an app using **app registration** there is also an **enterprise application** created in your AAD. But what really gets created in the Enterprise Application section is an instantiation of your applications in the tenant.   
You can think of the **App Registration** step like creating the definition of how you want your application to behave in AAD and in **Enterprise Application** you're seeing the instantiation of that definition in that concrete tenant.
For example if you want to assign users to an app you have registered previously on AAD you need to do it from the **Enterprise Application** blade instead of the **App Registration** section, that's because you're adding users from your tenant into that concrete instatiation of the application.   

## Q2: What's the difference between AppRoles and Scopes? 
## Q3: How I define a new AppRole for my application?
## Q4: What's the difference between delegated permissions and application permissions?

**Delegated permissions** are used when you want to authenticate to a WebAPI or other services with the currently logged-on user. This typically involves a physical user and a user interface.

**Application permission** are used when there is no user present. Mostly used for API to API calls. This is also used for background and daemon services.   
Unlike delegated permissions, application permissions, however, uses the app id and secret to login and always has the given permissions of the application.

- Can I add a delegated permission and also an application permission in my application?
- I have a SPA (Single Page Application) that makes a call to a protected WebAPI and also this WebApi needs to make a call to another protected WebApi, what should I do?   
- My application needs permission to call Microsoft Graph API, what I need to do?
- I have built a WebApi using csharp and I want to add authentication and authorization using AAD, what should I use? ADAL.NET, MSAL.NET or Microsoft.Identity.Web? 
- I have registered 2 apps on my AAD and I'm having troubling validating the issuer attribute from the access tokens. The issuer attribute in an access token for the app1 is "login.microsoftonline.com" and the issuer attribute in an access token for the app2 is "sts.windows.net". What's happening?
- Can I define a custom scopes and have them returned when using the client credential flow in Azure AD?


