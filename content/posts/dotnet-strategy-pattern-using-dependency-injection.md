---
title: "Back to .NET basics: How to easily build a Strategy pattern using dependency injection"
date: 2023-09-13T15:03:54+02:00
tags: ["patterns", "dotnet", "basics"]
description: "In this very brief post, we will see how easy is to build a Strategy pattern in .NET when using dependency injection."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/dotnet-strategy-pattern-using-dependency-injection).

**If you are a .NET veteran, this post is probably not intended for you**.

The Strategy pattern is probably one of the most well-known software design patterns nowadays. It is a behavioral pattern that defines a set of behaviors or algorithms, encapsulates each one of them, and makes them interchangeable.

This pattern allows the app to choose an algorithm from a family of algorithms at runtime, without altering the code that uses the algorithm.

Here's a diagram _(which you'll find in dozens of other places)_ showing the components involved when trying to build this pattern:

![strategy-patter-diagram](/img/strategy-pattern-diagram.png)

The key components participating in this pattern are:

- **Strategy**: This is the interface that defines the methods that all Concrete Strategies must implement.

- **Concrete Strategies**: These are the implementations of the strategy interface. Each Concrete Strategy provides a specific implementation of the algorithm.

- **Context**: This is the class that holds a reference to the Concrete Strategies and executes one or another based on the application inputs.


What some people may not realize is that the .NET dependency injection system does most of the heavy lifting for you when you're trying to implement a Strategy pattern. **Let me show it to you.**


# **Defining the Strategy interface**

The first step is to create an interface that all the Concrete Strategies must implement.

Let's keep it really basic here, with an ``Execute`` method to execute the algorithm and a property called ``Name`` that will store the name of the strategy.

```csharp
public interface IStrategy
{
    string Name { get; }
    string Execute(string message);
}
```

# **Implementing the Concrete Strategies**

Now, let's implement the Concrete Strategies.    

We're going to do something simple: given a string, we'll have three strategies. One will convert the entire string to uppercase, another to lowercase, and the last one will reverse it.

Obviously every strategy needs to implement the ``IStrategy`` interface.


```csharp
public class ToUpperStrategy : IStrategy
{
    public string Name => nameof(ToUpperStrategy);

    public string Execute(string message)
    {
        return message.ToUpper();
    }
}

public class ToLowerStrategy : IStrategy
{
    public string Name => nameof(ToLowerStrategy);

    public string Execute(string message)
    {
        return message.ToLower();
    }
}

public class ReverseStrategy : IStrategy
{
    public string Name => nameof(ReverseStrategy);

    public string Execute(string message)
    {
        var charArray = message.ToCharArray();
        Array.Reverse(charArray);
        return new string(charArray);
    }
}
```

# **Registering the Concrete Strategies into the DI container**

We have multiple classes implementing the same interface, so the registration into the DI is exactly as it sounds.

```csharp
builder.Services.AddTransient<IStrategy, ToUpperStrategy>();
builder.Services.AddTransient<IStrategy, ToLowerStrategy>();
builder.Services.AddTransient<IStrategy, ReverseStrategy>();
```

The drawback of registering every Concrete Strategy as shown in the code above is that every time we want to add a new Concrete Strategy to our Strategy pattern, we must remember to register it.

A better option to register the Concrete Strategies is to use a little bit of Reflection. This way, we can create as many behaviors as we want, and they will always be automatically registered in the DI container.

```csharp
var strategies = AppDomain.CurrentDomain.GetAssemblies()
                .SelectMany(assembly => assembly.GetTypes())
                .Where(type => typeof(IStrategy).IsAssignableFrom(type) && !type.IsInterface && !type.IsAbstract);

foreach (var strategy in strategies)
{
    builder.Services.AddTransient(
        typeof(IStrategy), 
        strategy);
}
```

# **Building the Context class**

The multiple ``IStrategy`` implementations that we have registered into the DI container are resolved into a collection of ``IEnumerable<IStrategy>``.    
With that in mind, the context class becomes very straightforward to implement. It only needs to filter which Concrete Strategy wants to use from the ``IEnumerable<IStrategy>`` and run it.

```csharp
public class StrategyContext : IStrategyContext
{
    private readonly IEnumerable<IStrategy> _strategies;

    public StrategyContext(IEnumerable<IStrategy> strategies)
    {
        _strategies = strategies;
    }

    public string ExecuteStrategy(
        string strategyName, 
        string message)
    {
        var instance = _strategies.FirstOrDefault(x =>
            x.Name.Equals(strategyName, StringComparison.InvariantCultureIgnoreCase));

        return instance is not null ?
            instance.Execute(message) :
            string.Empty;
    }
}
```


# **Testing it out**

To test the Strategy Pattern, I have built an ``APIController`` that receives the name of the strategy to execute and the message that needs to be converted. It then calls the ``ExecuteStrategy`` method from the ``StrategyContext`` class.

> This is just one way to test it in a straightforward manner. Obviously, in a real-world scenario, there will be some business logic involved that determines which behavior of the Strategy pattern needs to be executed.

```csharp
[ApiController]
[Route("[controller]")]
public class StrategiesController : ControllerBase
{
    private readonly IStrategyContext _strategyContext;
    
    public StrategiesController(IStrategyContext strategyContext)
    {
        _strategyContext = strategyContext;
    }

    [HttpGet()]
    public ActionResult Get(string strategyName, string message)
    {
        var result = _strategyContext.ExecuteStrategy(strategyName, message);
        
        if (string.IsNullOrEmpty(result))
        {
            return BadRequest();
        }

        return Ok(result);
    }
}

```

If we try to call the Api like this:

```bash
curl -X 'GET' \
  'https://localhost:7285/Strategies?strategyName=ToUpperStrategy&message=hello' \
  -H 'accept: */*'
```
It responds:

```bash
HELLO
```