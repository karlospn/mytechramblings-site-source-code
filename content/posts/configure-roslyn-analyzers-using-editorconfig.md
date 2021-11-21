---
title: "How to configure your custom roslyn analyzer using an .editorconfig file"
date: 2021-01-14T12:59:43+01:00
tags: ["csharp", "dotnet", "roslyn", "editorconfig"]
description: "Roslyn Analyzers are extensions that analyze source code and report violations. Some analyzers are built-into VS and some are third party ones which can be installed (like StyleCopyAnalyzers, FxCopAnalyzers, etc.). On today's post I will show you how you can configure your custom roslyn analyzers with an .editorconfig file."
draft: false
---

Roslyn Analyzers are extensions that analyze source code and report violations. Some analyzers are built-into VS (like the IDE analyzers that report style issues) and some are third party ones which can be installed (like StyleCopyAnalyzers, FxCopAnalyzers, etc.).   

Analyzers can be configurated using file(s) to allow end users to control the behavior of each analyzer and severity of reported diagnostics. This may be via an editorconfig or a ruleset file or even completely custom format specific to the third party analyzer package.   

IDE analyzers support configuration via .editorconfig for available options and semantics.
Meanwhile styleCop analyzers do not support EditorConfig based configuration, but use their own stylecop.json with their own custom format, that end users can provide in their project for configuration of stylecop analyzers.

If you have been working with .NET long enough you have probably used FxCop XML Ruleset files to configure the code analysis on a particular project, rulesets and editorconfig files can coexist and both can be used to configure analyzers. Both editorconfig files and rulesets let you enable and disable rules and set their severity.   
However, editorconfig files offer additional ways to configure rules too:
- For the .NET code-quality analyzers, editorconfig files let you define which types of code to analyze.   
- For the .NET code-style analyzers, editorconfig files let you define the preferred code styles for a codebase.   

> In this post I don't pretend to write a deep dive about how a roslyn analyzers works or what's an editorconfig file. I was just writing some lines about how these two work together, because that's going to be the main focus of this post.

**In this post I want to show you how you can configure your custom roslyn analyzer using an editorconfig file.**   
**You can implement different code styles on you analyzer and allow the end-user to decide via editorconfig which code style they want to enforce.**

I'm aware that maybe it's not entirely clear what I'm trying to achieve in this post so let me show you a very quick example.

## Quick example

The Roslyn rule **IDE0065** is an analyzer built into Visual Studio, specifically it analyzes if a "using" statement should be inside or outside a namespace based on a configuration attribute.   

This rule can be configure via editorconfig so you can define your preferred coding style. 

> More info here: https://docs.microsoft.com/es-es/dotnet/fundamentals/code-analysis/style-rules/ide0065#csharp_using_directive_placement

As I said previously with rule **IDE0065** you can enforce if a "using" statement should be inside or outside a namespace with the following editorconfig attribute:

```yaml
csharp_using_directive_placement = inside_namespace:error
csharp_using_directive_placement = outside_namespace:error
```
So if you want to enforce that all your "usings" are always placed inside a namespace you can use the "inside_namespace" value and viceversa.   
Let me show a practical test.   

- **Example 1**: If we configure the analyzer **IDE0065** via editorconfig like this:

```yaml
root = true
[*.cs]
csharp_using_directive_placement = inside_namespace:error
```

This block of code will report an error:

```csharp
// csharp_using_directive_placement = outside_namespace
using System;

namespace Convention2
{
    ...
}
```

But this one won't report any error

```csharp
// csharp_using_directive_placement = inside_namespace
namespace Convention1
{
    using System;
    ...
}
```

- **Example 2**: If we configure the analyzer **IDE0065** via editorconfig like this:

```yaml
root = true
[*.cs]
csharp_using_directive_placement = outside_namespace:error
```

This block of code will report an error:

```csharp
// csharp_using_directive_placement = inside_namespace
namespace Convention1
{
    using System;
    ...
}
```

But this one won't report any error

```csharp
// csharp_using_directive_placement = outside_namespace
using System;

namespace Convention2
{
    ...
}
```

**In this example we're using the "csharp_using_directive_placement" attribute to enforce our own code-style.**    

And that's **EXACTLY** what I want to talk in this post.    

**Given a roslyn analyzer how we can add support for different code styles using the editorconfig file.**    
That's a cool feature to add on your own Roslyn analyzers and it's super easy to do it.    
To do this we have the **AnalyzerConfigOptionsProvider Roslyn API**


# AnalyzerConfigOptionsProvider Roslyn API

The API **context.Options.AnalyzerConfigOptionsProvider.GetOptions**, it's a new Roslyn API that allows us to get the configuration from the editorconfig file.

More info. here: https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.diagnostics.analyzerconfigoptionsprovider?view=roslyn-dotnet

Be aware that this feature is only available on the newer versions of Microsoft.CodeAnalysis nuget package. **If you're using a nuget version below 3.1.0 that feature is not available**.

Let me show you how you can use it.  


# Building a Roslyn Analyzer that uses the AnalyzerConfigOptionsProvider API

First of all I have built a roslyn analyzer that reports an error if a csharp file contains more than one namespace. Here's the code:

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;
using System;
using System.Collections.Immutable;

namespace MyOwn.Analyzer
{
    [DiagnosticAnalyzer(LanguageNames.CSharp)]
    public class MyOwnAnalyzerAnalyzer : DiagnosticAnalyzer
    {
        public const string DiagnosticId = "MyRules0001";

        private static readonly LocalizableString Title = new LocalizableResourceString(nameof(Resources.AnalyzerTitle), Resources.ResourceManager, typeof(Resources));
        private static readonly LocalizableString MessageFormat = new LocalizableResourceString(nameof(Resources.AnalyzerMessageFormat), Resources.ResourceManager, typeof(Resources));
        private static readonly LocalizableString Description = new LocalizableResourceString(nameof(Resources.AnalyzerDescription), Resources.ResourceManager, typeof(Resources));
        private const string Category = "Naming";

        private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(DiagnosticId, 
            Title, 
            MessageFormat, 
            Category, 
            DiagnosticSeverity.Error, 
            true, 
            Description);

        public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => ImmutableArray.Create(Rule);

        public override void Initialize(AnalysisContext context)
        {
            context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
            context.EnableConcurrentExecution();
            context.RegisterSyntaxTreeAction(AnalyzeAction);
        }

        private static void AnalyzeAction(SyntaxTreeAnalysisContext context)
        {
           
            var syntaxRoot = context.Tree.GetRoot(context.CancellationToken);

            var descentNodes = syntaxRoot.
                DescendantNodes(node => node != null && !node.IsKind(SyntaxKind.ClassDeclaration));

            var foundNode = false;
            foreach (var node in descentNodes)
            {
                if (node.IsKind(SyntaxKind.NamespaceDeclaration))
                {
                    if (foundNode)
                    {
                        context.ReportDiagnostic(Diagnostic.Create(Rule, Location.None));
                    }
                    else
                    {
                        foundNode = true;
                    }
                }
            }
        }
    }
}
```
So if you have more than one namespace in a single csharp file that analyzer is going to report an error.   
But it would be kind of cool if **the end-user was capable of choosing if a csharp file can contain more than one namespace or not**.

And that's exactly what are we going to build right now, we are going to use a new editorconfig attribute that allows us to choose which behaviour we want to enforce.

First we need to create a new attribute on the editorconfig.

> Beware that you **NEED** to follow this pattern: dotnet_diagnostic.<RULE_ID>.<ATTRIBUTE_NAME>

I decided that the attribute will be:

```yaml
dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_single_file = false
```

And now I need to add the code on my analyzer that reads this new attribute and acts accordingly. Here's the result: 

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;
using System;
using System.Collections.Immutable;

namespace MyOwn.Analyzer
{
    [DiagnosticAnalyzer(LanguageNames.CSharp)]
    public class MyOwnAnalyzerAnalyzer : DiagnosticAnalyzer
    {
        public const string DiagnosticId = "MyRules0001";

        private static readonly LocalizableString Title = new LocalizableResourceString(nameof(Resources.AnalyzerTitle), Resources.ResourceManager, typeof(Resources));
        private static readonly LocalizableString MessageFormat = new LocalizableResourceString(nameof(Resources.AnalyzerMessageFormat), Resources.ResourceManager, typeof(Resources));
        private static readonly LocalizableString Description = new LocalizableResourceString(nameof(Resources.AnalyzerDescription), Resources.ResourceManager, typeof(Resources));
        private const string Category = "Naming";

        private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(DiagnosticId, 
            Title, 
            MessageFormat, 
            Category, 
            DiagnosticSeverity.Error, 
            true, 
            Description);

        public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => ImmutableArray.Create(Rule);

        public override void Initialize(AnalysisContext context)
        {
            context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
            context.EnableConcurrentExecution();
            context.RegisterSyntaxTreeAction(AnalyzeAction);
        }

        private static void AnalyzeAction(SyntaxTreeAnalysisContext context)
        {
            var config = context.Options.AnalyzerConfigOptionsProvider.GetOptions(context.Tree);
            config.TryGetValue("dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file", out var configValue);
           
            if (string.IsNullOrEmpty(configValue))
            {
                return;
            }

            var syntaxRoot = context.Tree.GetRoot(context.CancellationToken);

            var descentNodes = syntaxRoot.
                DescendantNodes(node => node != null && !node.IsKind(SyntaxKind.ClassDeclaration));
            var foundNode = false;

            foreach (var node in descentNodes)
            {
                if (node.IsKind(SyntaxKind.NamespaceDeclaration))
                {
                    if (foundNode)
                    {
                        if (configValue.Equals("false", StringComparison.InvariantCultureIgnoreCase))                       
                        {
                            context.ReportDiagnostic(Diagnostic.Create(Rule, Location.None));
                        }
                    }
                    else
                    {
                        foundNode = true;
                    }
                }
            }
        }
    }
}
```
- As you can see it's really easy to implement, but let me break down the key aspects:

I begin by reading the value from the editorconfig file using the AnalyzerConfigOptionsProvider API.

```csharp
var config = context.Options.AnalyzerConfigOptionsProvider.GetOptions(context.Tree);
config.TryGetValue("dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file", out var configValue);
 ```

 In my case I have implemented a default behaviour, so if the end-user do not set the value on the editorconfig I assume that we can have more than one namespace in the same file.   
 That's the reason why I'm breaking the code execution in case that the value is null or empty.

```csharp
if (string.IsNullOrEmpty(configValue))
{
    return;
}
 ```

Only report a diagnostic if we have found more than one namespace on the same file and the "use_multiple_namespaces_in_a_file" value is set to false

```csharp
if (foundNode)
{
    if (configValue.Equals("false", StringComparison.InvariantCultureIgnoreCase))
    {
        context.ReportDiagnostic(Diagnostic.Create(Rule, Location.None));
    }
}
else
{
    foundNode = true;
}
```


# Testing our analyzer

As a last step let's test the analyzer.

- **Test 1**

I created the following editorconfig file:

```yaml
root = true

# C# files
[*.cs]

dotnet_diagnostic.MyRules0001.severity = error
dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file = false
```

Also I have created a new console program that contains my analyzer.    
The _Program.cs_ file looks like this:

```csharp
//program.cs

using System;

namespace ConsoleApp3
{
    class Program
    {
        public static void Main(string[] args)
        {
            var stud = new StudentNs.Student();
            stud.PrintStudentName("Carlos");        
        }
    }
}


namespace StudentNs
{
    public class Student
    {
       public void PrintStudentName(string name)
       {
            Console.WriteLine($"Your name is: {name}");
       }
    }
}
```

As you can see VS shows an error:

![ns-error](/img/roslyn-analyzer-error-ns.png)


- **Test 2**

Now if I modify the editorconfig file to look like this:

```yaml
root = true

# C# files
[*.cs]

dotnet_diagnostic.MyRules0001.severity = error
dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file = true
```
And I'm using the same program.cs as the previous test.

Now Visual Studio doesn't show any error, because now we are allowing to use multiple namespaces in a single file.


# Useful Resources
- https://github.com/dotnet/roslyn-analyzers/blob/master/docs/Analyzer%20Configuration.md
- https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/code-style-rule-options?view=vs-2017
- https://docs.microsoft.com/en-us/visualstudio/code-quality/analyzers-faq?view=vs-2019
- https://github.com/dotnet/roslyn-analyzers
