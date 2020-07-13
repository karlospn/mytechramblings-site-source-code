---
title: "How to Validate AAD V1 and V2 Jwt Tokens"
date: 2020-07-13T21:34:20+02:00
draft: true
---

In these post I want to talk about how you can validate a JWT token that's being issued either by Azure Active Directory V1 endpoint or Azure Active Directory V2 endpoint.

> If you want to know what's the difference between the AAD v1 endpoint and the v2 endpoint you can read it from [here](https://devblogs.microsoft.com/premier-developer/azure-ad-endpoint-v1-vs-v2/)


If you go to into your AAD > app registrations section you can see both endpoints:

<FTO>

Probably after reading the title of the post I'm pretty sure that you are thinking something like:

- "Nowadays no one is using the AAD V1 endpoint, why should I care about it?"
- "I don't need to know about these, my new application is already using the AAD V2 endpoint."

Well, yes and no. And here's the tricky part.   
The AAD V2 endpoint is capable of issuing tokens in V1 and V2 format. **And by default is issuing tokens in V1 format.**   
And a common issue I'm seeing in quite a few places is that people thinks that they're validating a token on V2 format but that's not the case, what they're really doing is validating a token **issued** by the AAD V2 endpoint but in the **V1** format 

And that's a problem if you are using a middleware like these one:

```csharp
app.UseWindowsAzureActiveDirectoryBearerAuthentication(
            new WindowsAzureActiveDirectoryBearerAuthenticationOptions
            {
                Audience = ConfigurationManager.AppSettings["ida:Audience"],
                Tenant = ConfigurationManager.AppSettings["ida:Tenant"]
            });

```
That middleware works OK when you want to validate a V1 format token but it's not going to work for a token in V2 format.


# Differences between AAD V1 and V2 JWT tokens

Let's check it out. I did a couple of things:   

- I registered 2 apps in AAD: a client app and a resource app.
- The client is going to obtain a token from the Authorization Server (AAD) and pass it to the resource server.
- The resource server is going to validate that the token is issued by a trusted source and have the correct audience.


When the client app asks for an access_token in the v1 format:

```json

```


When the client app asks for an access_token in the v2 format:

```json

```




