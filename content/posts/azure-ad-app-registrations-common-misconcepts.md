---
title: "Azure Active Directory App Registrations basic qüestions"
date: 2020-10-10T17:07:50+02:00
draft: true
---

In these past years I worked with quite a few companies that uses Azure Active Directory to perform authentication and authorization on their line-of-business (LOB) applications.
Each application you want the Azure AD to perform identity and access management for needs to be registered. Whether it's a client application like a web or mobile app, or it's an API that backs a client app, registering it establishes a trust relationship between your application and the identity provider, the Microsoft identity platform.   

From time to time I get qüestions about how to register apps on AAD and those qüestions are almost always the same and most of the time the problem lies in some misconception.   
Most of the qüestion are pretty basic but nonetheless I thought it might be a good idea to  try to answer some of them in this post, so next time someone ask me some qüestion I can point them to this post.

Maybe in the future I update this post with more qüestions or maybe I write another post with another batch of qüestions.  

Let's start with the first batch of qüestions:

- What's the difference between App Registration and Enterprise Application?
- What's the difference between AppRoles and Scopes? 
- How I define a new AppRole for my application?
- What's the difference between delegated permissions and application permissions?
- Can I add a delegated permission and also an application permission in my application?
- I have a SPA (Single Page Application) that makes a call to a protected WebAPI and also this WebApi needs to make a call to another protected WebApi, what should I do?   
- My application needs permission to call Microsoft Graph API, what I need to do?
- I have built a WebApi using csharp and I want to add authentication and authorization using AAD, what should I use? ADAL.NET, MSAL.NET or Microsoft.Identity.Web? 
- I have registered 2 apps on my AAD and I'm having troubling validating the issuer attribute from the access tokens. The issuer attribute in an access token for the app1 is "login.microsoftonline.com" and the issuer attribute in an access token for the app2 is "sts.windows.net". What's happening?
- Can I define a custom scopes and have them returned when using the client credential flow in Azure AD?


