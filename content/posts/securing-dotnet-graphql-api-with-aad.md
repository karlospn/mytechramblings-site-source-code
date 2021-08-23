---
title: "Securing a graphQL API with Azure Active Directory"
date: 2021-08-22T15:51:33+02:00
tags: ["dotnet", "csharp", "aad", "graphql", "azure"]
draft: true
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/securing-graphql-api-with-aad).   

In today's post I want to talk about how you can secure a .NET graphQL API using Azure Active Directory (AAD).    

If you want to build a graphQL API in .NET right now, there are a couple of options available: the graphQL.NET implementation (https://graphql-dotnet.github.io/) or the HotChocolate one (https://chillicream.com/).    

After tinkering quite a bit with both of them I decided to use the HotChocolate implementation. HotChocolate has more features, some off them are really useful like schema stitching. It is also faster and has a smaller memory footprint. And to top it off it has an steady update frequency.    

In this post I won't be talking about how you can build a graphQL API with HotChocolate, mainly because there are a lot of great examples online and writing another one seems meaningless. Instead of that I'll be focusing on how you can secure it with AAD.   

The main talking points in this post are going to be the following ones:

- How to secure a graphQL api using the Microsoft.Web.Identity nuget package.
- What are the introspection queries and how to secure them.
- How to call a protected graphQL api using the GraphQL.Client and the Microsoft.Web.Client nuget packages.


# 1. How to secure a graphQL api using the Microsoft.Web.Identity nuget package

First thing is to register a couple of apps on AAD. In my case I have registered an app called  ``GraphQL.WebApi`` and another one called ``GraphQL.Client``.    
Also the ``GraphQL.WebApi`` app exposes a scope and that scope is consumed by the ``GraphQL.Client`` app.

Now it's time to install and set up the ``Microsoft.Identity.Web`` nuget package. To install it run the command: ``dotnet add package Microsoft.Identity.Web --version 1.16.0``

After the library is installed there are a few thing you need to do:

1. On the ``appsettings.json``  configure how the library is going to work with my AAD:

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
 Also it is a good practice to always validate the ``iss`` and the ``aud`` attributes from the JWT Token, that way you can ascertain that the token has been issued by your own identity provider and the target for the token is your API.

2. On the ``Startup.cs`` register the library dependencies using the ``AddMicrosoftIdentityWebApi`` extension method.

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

3. Now you need to install the ``HotChocolate.AspNetCore.Authorization`` nuget package. To install it run the command: ``dotnet add package HotChocolate.AspNetCore.Authorization --version 11.3.5``

This library will allow us to add authorization for our graphQL types, querys and mutations. 

The nice thing about this library is that is built on top of the .NET Core authentication mechanisms, so it will work hand in hand with the ``Microsoft.Identity.Web`` library without any extra code.   
Once the ``HotChocolate.AspNetCore.Authorization`` is installed you need to register the library dependencies using the ``AddAuthorization()``  extension method:

```csharp
 services.AddGraphQLServer()
         .AddAuthorization();
```
The only thing left is to choose what part of your graphQL API you want to secure. There are 4 options available here:

**Option 1.** You can protect the entire set of queries or mutations. To do it you simply need to add the ``Authorize`` attribute at class level.

> Be careful! The ``Authorize`` attribute that you need is the one from the ``HotChocolate.AspNetCore.Authorization`` assembly. Do not get confused with the other one that comes from the ``Microsoft.AspNetCore.Authorization`` assembly.

```csharp
[GraphQLDescription("Represents the available Book queries.")]
[ExtendObjectType(typeof(Query))]
[Authorize]
public class BookQuery
{
    public  IEnumerable<Book> GetBooks(
        [Service] IBookRepository repository)
    {
        return repository.GetBooks();
    }

    [Authorize]
    public IEnumerable<Book> GetBooksByAuthor(
        [Service] IBookRepository repository,
        string author)
    {
        return repository.GetBookByAuthor(author);
    }
}
```

**Option 2.** You can protect concrete queries or mutations. To do it you need to add the ``Authorize`` attribute at method level.
In this example only the ``GetBooks`` method is protected.

```csharp
[GraphQLDescription("Represents the available Book queries.")]
[ExtendObjectType(typeof(Query))]
public class BookQuery
{
    [Authorize]
    public  IEnumerable<Book> GetBooks(
        [Service] IBookRepository repository)
    {
        return repository.GetBooks();
    }

    public IEnumerable<Book> GetBooksByAuthor(
        [Service] IBookRepository repository,
        string author)
    {
        return repository.GetBookByAuthor(author);
    }
}
```

**Option 3.** You can protect an object type. In that case you'll need to be authorized only if you want to execute a query that returns this particular object type.   
To do it you need to add the ``Authorize`` attribute in the object type if you're using an annotation based approach.

In the example below I'm protecting the ``Book`` object type. If I try to run a query that returns a list of books without the authorization header it will return an error.

```csharp
[Authorize]
public class Book
{
    public string Author { get; set; }
    public string Title { get; set; }
    public double Price { get; set; }
    public int NumberOfPages { get; set; }
}
```

If you're using a code-first based approach instead of the annotation based approach, you'll need to create a new class inheriting from ``ObjectType<T>`` and define it there.

```csharp
public class BookType : ObjectType<Book>
{
    protected override void Configure(IObjectTypeDescriptor<Book> descriptor)
    {
        descriptor.Authorize();

        descriptor.Description("Represents a book entity.");

        descriptor
            .Field(c => c.Author)
            .Description("The author of the book.");

        descriptor
            .Field(c => c.Title)
            .Description("The title of the book.");

        descriptor
            .Field(c => c.Price)
            .Description("Book price on Euros.");

        descriptor
            .Field(c => c.NumberOfPages)
            .Description("How many pages the book has.");

        base.Configure(descriptor);
    }
}
```

**Option 4.** You can protect a concrete object attribute. In that case you'll need to be authorized only if you want to execute a query that returns this particular attribute.   
To do it you need to add the ``Authorize`` attribute at attribute level if you're using an annotation based approach.

In the example below I'm protecting the ``Title`` attribute of the ``Book`` object type. If I try to run a query that returns a list of books without the authorization header and I ask to get back the title field it will return an error.    
If I run the same query but I don't ask for the title field it would work.

```csharp
public class Book
{
    public string Author { get; set; }
    [Authorize]
    public string Title { get; set; }
    public double Price { get; set; }
    public int NumberOfPages { get; set; }
}

```

If you're using a code-first based approach instead of the annotation based approach, you'll need to create a new class inheriting from ``ObjectType<T>`` and define it there.

```csharp
public class BookType : ObjectType<Book>
{
    protected override void Configure(IObjectTypeDescriptor<Book> descriptor)
    {
        descriptor.Description("Represents a book entity.");

        descriptor
            .Field(c => c.Author)
            .Description("The author of the book.");

        descriptor
            .Field(c => c.Title)
            .Description("The title of the book.")
            .Authorize();

        descriptor
            .Field(c => c.Price)
            .Description("Book price on Euros.");

        descriptor
            .Field(c => c.NumberOfPages)
            .Description("How many pages the book has.");

        base.Configure(descriptor);
    }
}
```

# 2. What are the introspection queries and how to secure them.


The introspection query enables anyone to query a GraphQL server for information about the  schema.   
With this query you can gain information about the directives, available types, and available operation types.

The introspection query is used primary as a discovery mechanism.  

Probably you're not going to run the query by yourself, but nonetheless it’s important for tooling and GraphQL IDEs like GraphiQL or Banana Cake Pop. Behind the scenes, GraphQL IDEs uses the introspection queries to offer a rich user experience and for diagnosing your graph during development.

When you move your API to production it doesn't seem a good idea to leave the introspection query unprotected, it might reveal some data or give extra information to a malicious user.

I'm going to show you a couple of options available if you want to protect the introspection query in your graphQL API.

**Option 1. Protect the introspection query using a custom header.**

With this option we're allowing introspection queries only if the request contains a certain custom header.

You'll need to build a custom interceptor that intercepts any Http request and inspects if the header is present or not. Here's how it looks.

```csharp
public class ConditionalIntrospectionHttpRequestInterceptor : DefaultHttpRequestInterceptor
{
    private readonly IConfiguration _configuration;

    public ConditionalIntrospectionHttpRequestInterceptor(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public override async ValueTask OnCreateAsync(HttpContext context,
        IRequestExecutor requestExecutor,
        IQueryRequestBuilder requestBuilder,
        CancellationToken cancellationToken)
    {
        string header = context.Request.Headers["X-INTROSPECTION-QUERY"];
        var key = _configuration["IntrospectionKey"];

        if (header != null &&
            key != null &&
            header.Equals(key, StringComparison.InvariantCultureIgnoreCase))
            requestBuilder.AllowIntrospection();
        else
            requestBuilder.SetIntrospectionNotAllowedMessage($"The introspection query has been disabled.");

        await base.OnCreateAsync(context, requestExecutor, requestBuilder, cancellationToken);
    }
}
```

Also you need the register the dependencies like these:

```csharp
services.AddGraphQLServer()
            .AddIntrospectionAllowedRule()
            .AddHttpRequestInterceptor(_ => new ConditionalIntrospectionHttpRequestInterceptor(configuration));
```
The ``AddIntrospectionAllowedRule()`` extension method by default is blocking each and everyone of the introspection queries that the server receives. And with the ``ConditionalIntrospectionHttpRequestInterceptor`` only the introspection queries that contains our custom header are getting through the interceptor.

**Option 2. Protect the introspection query with AAD.**

This is more of a side effect when trying to protect the entire query set. In the previous section I have showed you that you could protect the entire query or mutation set by placing the ``Authorize`` attribute at class level. Like this:

```csharp
[GraphQLDescription("Represents the available Book queries.")]
[ExtendObjectType(typeof(Query))]
[Authorize]
public class BookQuery
{
    public  IEnumerable<Book> GetBooks(
        [Service] IBookRepository repository)
    {
        return repository.GetBooks();
    }

    [Authorize]
    public IEnumerable<Book> GetBooksByAuthor(
        [Service] IBookRepository repository,
        string author)
    {
        return repository.GetBookByAuthor(author);
    }
}
```

Placing the ``Authorize`` attribute at class level will protect the entire query set including including the introspection query.


# 3. How to call a protected graphQL api using the Microsoft.Web.Client nuget package.

The HotChocolate suite has a graphQL client named [``Strawberry Shake``](https://chillicream.com/docs/strawberryshake), but it's a very opinionated client.   
To use it you'll need to:
- Create a ``graphqlrc.json`` file that specfies where the schema can be fetched.
- Create a ``.graphql`` file that contains the query you want to execute.
- Execute the ``dotnet build`` command. This will auto-generate a bunch of files and amongst them there will generate a  ``csharp`` client that you can use to call the graphQL API.

I'm not a big fan of ``Strawberry.Shake``, usually I prefer to use the ``GraphQL.Client`` nuget package. It's built on top of the C# ``HttpClient`` implementation and works exactly like an HttpClient.
-  Create a new instance of the graphQL client.
-  Specify the URI and the query.
-  Ran it.

To show you how to call a protected graphQL API I have created a secondary web API and I'll use it to call the protected graphQL server. 

To call the protected graphQ server I will be using an OAth2 Client Credentials flow and for that purpose I need to install the ``Microsoft.Identity.Client`` package. 

After the library is installed there are a few thing you need to do:

1. On the ``appsettings.json``  I need to configure how the library is going to work with my AAD:

```javascript
  "AzureAd": {
    "Authority": "https://login.microsoftonline.com/8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
    "ClientId": "a895c416-cfb9-4df0-9051-190febbbdf64",
    "ClientSecret": "J.6l-VR2lefhipuJSB",
    "Scopes": [ "api://2afa13f1-6873-4629-a072-c6a5792e55c3/.default" ]
  } 
```

As I have stated in the previous section I have registered in my AAD an app called  ``GraphQL.WebApi`` and another one called ``GraphQL.Client``.    
The ``a895c416-cfb9-4df0-9051-190febbbdf64`` is the ``GraphQL.Client`` ClientId and the ``api://2afa13f1-6873-4629-a072-c6a5792e55c3/.default`` is the exposed scope from the ``GraphQL.WebApi``.

2. Register the ``Microsoft.Identity.Client`` configuration and the ``graphQL.Client`` implementation into the DI container:


```csharp
services.AddScoped<IGraphQLClient>(sp =>
    new GraphQLHttpClient(new GraphQLHttpClientOptions
        {
            EndPoint = new Uri(Configuration["GraphQLURI"]),
            HttpMessageHandler = new AuthorizationHandler(       
                sp.GetRequiredService<IOptions<AzureAdConfig>>()),

        }, new SystemTextJsonSerializer())
);

services.AddOptions<AzureAdConfig>()
    .Bind(Configuration.GetSection("AzureAd"));

```

To fetch a JWT token from my AAD I'll be using an ``HttpMessageHandler``. The implementation of the ``DelegatingHandler`` looks like this:

```csharp
public class AuthorizationHandler : DelegatingHandler
{
    private readonly IOptions<AzureAdConfig> _config;

    public AuthorizationHandler(IOptions<AzureAdConfig> config, HttpMessageHandler inner = null) 
        : base(inner ?? new HttpClientHandler())
    {
        _config = config;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, 
        CancellationToken cancellationToken)
    {
        var app = ConfidentialClientApplicationBuilder.Create(_config.Value.ClientId)
            .WithClientSecret(_config.Value.ClientSecret)
            .WithAuthority(new Uri(_config.Value.Authority))
            .Build();

        var token = await app.AcquireTokenForClient(_config.Value.Scopes)
            .ExecuteAsync(cancellationToken);

        request.Headers.Authorization = new AuthenticationHeaderValue("bearer", token.AccessToken);
        return await base.SendAsync(request, cancellationToken);
    }
}
```

3. Create a graphQL query:

```csharp
public static class GetBooksByAuthor
{
    public const string Value =
        @"
        query GetByAuthor ($author: String!) {
            getBooksByAuthor: booksByAuthor(author: $author) {
                author
                title
                numberOfPages
                price
            }
        }";
}
```

4. And use it

```csharp
[ApiController]
[Route("[controller]")]
public class BooksController : ControllerBase
{
    private readonly ILogger<BooksController> _logger;
    private readonly IGraphQLClient _client;

    public BooksController(
        ILogger<BooksController> logger, 
        IGraphQLClient client)
    {
        _logger = logger;
        _client = client;
    }

    [HttpGet("{author}")]
    public async Task<ActionResult> GetBooksByAuthor(
        [FromRoute]string author)
    {
        try
        {
            var query = new GraphQLRequest
            {
                Query = Queries.GetBooksByAuthor.Value,
                Variables = new
                {
                    author = author
                }
            };

            var result = await _client.SendQueryAsync<GetBooksByAuthorData>(query);

            if (result.Errors != null && result.Errors.Any())
            {
                return StatusCode(StatusCodes.Status500InternalServerError);
            }

            return Ok(result.Data.Books);
        }
        catch (Exception e)
        {
            _logger.LogError("Something went wrong", e);
            return StatusCode(StatusCodes.Status500InternalServerError, "Something went wrong");
        }

    }
}
```
