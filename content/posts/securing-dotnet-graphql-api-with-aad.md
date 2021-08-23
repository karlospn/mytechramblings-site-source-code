---
title: "Securing a graphQL API with Azure Active Directory"
date: 2021-08-22T15:51:33+02:00
tags: ["dotnet", "csharp", "aad", "graphql", "azure"]
draft: true
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/securing-graphql-api-with-aad).   

In today's post I'll to tackle how you can secure a .NET graphQL API using Azure Active Directory (AAD).    
If you want to build a graphQL API in .NET right now, there are a couple of options: you can use the graphQL.NET implementation (https://graphql-dotnet.github.io/) or you can the HotChocolate  (https://chillicream.com/).    
After tinkering quite a bit with both of them I decided to use the HotChocolate implementation. HotChocolate has more features, some off them are really useful like schema stitching. It is also faster and has a smaller memory footprint. And to top it off it has an steady update frequency.    
But in this post I won't be talking about how you can build a graphQL API with HotChocolate, mainly because there a lot of great examples onlines and writing another one seems meaningless. Instead of that I'll be focusing on how you can secure it a graphQL API.   

The main talking points in this post are going to be the following ones:

- How to secure a graphQL api using the Microsoft.Web.Identity nuget package.
- What are the introspection queries and how to secure them.
- How to call a protected graphQL api using the Microsoft.Web.Client nuget package.


# 1. How to secure a graphQL api using the Microsoft.Web.Identity nuget package

First thing you need to do is to register a couple of apps on AAD.    
In my case I have registered an app called  ``GraphQL.WebApi`` and another one called ``GraphQL.Client``. Also the ``GraphQL.WebApi`` app exposes a scope and that scope is consumed by the ``GraphQL.Client`` app.

Now it's time to install and set up the ``Microsoft.Identity.Web`` nuget package. To install it run the command: ``dotnet add package Microsoft.Identity.Web --version 1.16.0``

After the library is installed there are a few thing you need to do:

1. On the ``appsettings.json``  configure how the library is going to work:

```javascropt
"AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
    "Domain": "carlosponsnoutlook.onmicrosoft.com",
    "ClientId": "2afa13f1-6873-4629-a072-c6a5792e55c3",
    "TokenValidationParameters": {
      "ValidateIssuer": true,
      "ValidIssuer": "https://sts.windows.net/8a0671e2-3a30-4d30-9cb9-ad709b9c744a/",
      "ValidateAudience": true,
      "ValidAudiences": [ "api://2afa13f1-6873-4629-a072-c6a5792e55c3" ]
    }
```
 ``2afa13f1-6873-4629-a072-c6a5792e55c3`` is the ``GraphQL.WebApi`` clientID.    
 Also it is a good practice to always validate the ``iss`` and the ``aud`` attributes from the JWT Token, that way you can ascertain that the token has been issued by your own Idp and the target for the token is your API.

1. On the ``Startup.cs`` register the library dependencies using the ``AddMicrosoftIdentityWebApi`` extension method.

```csharp
    public void ConfigureServices(IServiceCollection services)
    {
        ...
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddMicrosoftIdentityWebApi(Configuration);
        ...
    }
```
With the ``Microsoft.Web.Identity`` package put in place the API is capable to authenticate and validate your calls using the AAD.

Now you need to install the ``HotChocolate.AspNetCore.Authorization`` nuget package. To install it run the command: ``dotnet add package HotChocolate.AspNetCore.Authorization --version 11.3.5``

This library will allow us to add authorization for our graphQL types, querys and mutations. 

The nice thing about this library is that is built on top of the .NET Core authentication mechanisms so it will work hand in hand with the ``Microsoft.Identity.Web`` library without any extra code.   
Once the ``HotChocolate.AspNetCore.Authorization`` is installed you need to register the library dependencies using the ``AddAuthorization()``  extension method:

```csharp
 services.AddGraphQLServer()
         .AddAuthorization();
```
After that the only thing left is to choose what part of your graphQL Api you want to secure. There are 4 options available here:

**Option 1.** You can protect the entire set of queries or mutations. To do it you simply need to add the ``Authorize`` attribute at class level.

> Be careful! The ``Authorize`` attribute that you need is the one from the ``HotChocolate.AspNetCore.Authorization`` assembly. Do not get confused with the other one that comes from the ``Microsoft.AspNetCore.Authorization`` assembly.

```csharp


```

**Option 2.** You can protect concrete queries or mutations. To do it you need to add the ``Authorize`` attribute at method level.

```csharp


```

**Option 3.** You can protect an object type. In that case you'll need to be authorized only if you want to execute a query that returns this particular object type.   
To do it you need to add the ``Authorize`` attribute in the object type if you're using an annotation based approach.

In the example below I'm protecting the ``Book`` object type. If I try to run a query that returns a list of books without the authorization header it will return an error.

```csharp


```

If you're using a code-first based approach instead of the annotation based approach, you'll need to create a new class inheriting from ``ObjectType<T>`` and define it there.

```csharp


```

**Option 4.** You can protect a concrete object attribute. In that case you'll need to be authorized only if you want to execute a query that returns this particular attribute.   
To do it you need to add the ``Authorize`` attribute at attribute level if you're using an annotation based approach.

In the example below I'm protecting the ``Title`` attribute of the ``Book`` object type. If I try to run a query that returns a list of books without the authorization header and I ask to get back the title field it will return an error.    
If I run the same query but I don't ask for the title field it would work.

```csharp


```

If you're using a code-first based approach instead of the annotation based approach, you'll need to create a new class inheriting from ``ObjectType<T>`` and define it there.

```csharp


```

# 2. What are the introspection queries and how to secure them.


The introspection query enables anyone to query a GraphQL server for information about the  schema.   
With this query you can gain information about the directives, available types, and available operation types.

The introspection query is used primary as a discovery mechanism.   
Probably you're not going to ran the query by yourself, but nonetheless it’s important for tooling and GraphQL IDEs like GraphiQL or Banana Cake Pop. Behind the scenes, GraphQL IDEs use introspection queries to offer a rich user experience and for diagnosing your graph during development.

When you move your API to production it doesn't seem a good idea to leave the introspection query unprotected, it might reveal some data or give extra information to a malicious user.

I'm going to show a couple of options available if you want to protect the introspection query in your graphQL API.

**Option 1.**


# 3. How to call a protected graphQL api using the Microsoft.Web.Client nuget package.