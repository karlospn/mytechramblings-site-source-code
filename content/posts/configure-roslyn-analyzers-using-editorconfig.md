---
title: "How to configure your custom roslyn analyzer using an .editorconfig file"
date: 2021-01-13T12:59:43+01:00
tags: ["c#", "csharp", "dotnet", "roslyn", "editorconfig"]
draft: true
---

# Introduction

Roslyn Analyzers are extensions that analyze source code and report violations. Some analyzers are built-into VS (like the IDE analyzers that report style issues) and some are third party ones which can be installed (like StyleCopyAnalyzers, FxCopAnalyzers, etc.)

Analyzers can be configurated using file(s) to allow end users to control the behavior of each analyzer and severity of reported diagnostics. This may be via an editorconfig or a ruleset file or even completely custom format specific to the third party analyzer package.   

IDE analyzers support configuration via .editorconfig for available options and semantics.
Meanwhile styleCop analyzers do not support editorconfig based configuration, but use their own stylecop.json with their own custom format, that end users can provide in their project for configuration of stylecop analyzers.

If you have being working with .NET long enough you have probably used FxCop XML Ruleset files to configure the code analysis on a particular project. RuleSets and EditorConfig files can coexist and can both be used to configure analyzers. Both editorConfig files and rule sets let you enable and disable rules and set their severity.   
However, EditorConfig files offer additional ways to configure rules too:
- For the .NET code-quality analyzers, EditorConfig files let you define which types of code to analyze.   
- For the .NET code-style analyzers that are built into Visual Studio, EditorConfig files let you define the preferred code styles for a codebase.   

> I DON'T pretend to write a deep dive about how a roslyn analyzers works or what's an editorconfig file.   

**In this post I want to show you how you can add whatever your want on an Editorconfig file and how a roslyn analyzer can read those values and act accordingly.**

**That's a really nice cool because you can implement different code styles on you analyzer and allow the end user to decide via editorconfig which code style they want to enforce.**

I'm aware that probably it's not entirely clear what I'm trying to achieve in this post so let me show you a very quick example.

## Quick example

The Roslyn rule **IDE0065** is an analyzer built into Visual Studio, specifically it analyzes if a using statement should be inside or outside a namespace based on a configuration attribute.   

This rule can be configure via EditorConfig and you can define your preferred code style. More info here: https://docs.microsoft.com/es-es/dotnet/fundamentals/code-analysis/style-rules/ide0065#csharp_using_directive_placement

With rule **IDE0065** you can define if a using statement should be inside or outside a namespace
with the following editorconfig attribute:

```yaml
csharp_using_directive_placement = inside_namespace:error
csharp_using_directive_placement = outside_namespace:error
```
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

**In concrete example we're using the "csharp_using_directive_placement" editorconfig attribute to enforce to different code-styles.**  

And that's **EXACTLY** what I want to talk in this post.   
Given a roslyn analyzer how we can add support to different code styles using an attribute on the editorconfig file.   
That's a cool feature to add on your own Roslyn analyzers and it's super easy to do it.    
To do this we have the **AnalyzerConfigOptionsProvider Roslyn API**


# AnalyzerConfigOptionsProvider Roslyn API

The API **context.Options.AnalyzerConfigOptionsProvider.GetOptions**, it's a new Roslyn API that allows us to get the configuration from the editorconfig file.

More info. here: https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.diagnostics.analyzerconfigoptionsprovider?view=roslyn-dotnet

Be aware that this feature is only available on the newer versions of Microsoft.CodeAnalysis nuget package. **If you're using a nuget version below 3.1.0 that feature is not available**.

Let me show you how you can use it.   
I have built a roslyn analyzer that reports an error if a csharp file contains more than one namespace. Here's the code:

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
                        var location = GetNameOrIdentifierLocation(node);
                        if (location != null)
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

         internal static Location GetNameOrIdentifierLocation(SyntaxNode member)
        {
            Location location = null;
            location = location ?? (member as PropertyDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as FieldDeclarationSyntax)?.Declaration?.Variables.FirstOrDefault()?.Identifier.GetLocation();
            location = location ?? (member as MethodDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as ConstructorDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as DestructorDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as BaseTypeDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as NamespaceDeclarationSyntax)?.Name.GetLocation();
            location = location ?? (member as UsingDirectiveSyntax)?.Name.GetLocation();
            location = location ?? (member as ExternAliasDirectiveSyntax)?.Identifier.GetLocation();
            location = location ?? (member as AccessorDeclarationSyntax)?.Keyword.GetLocation();
            location = location ?? (member as DelegateDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as EventDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as IndexerDeclarationSyntax)?.ThisKeyword.GetLocation();
            location = location ?? member.GetLocation();
            return location;
        }
    }
}
```

Now I want to add editorconfig support. More specifically **I want the end user to be capable of choosing if a csharp file can contains more than one namespace or not.**

On the editorconfig file I wanted to add a line that will look something like this:

```yaml
dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_single_file = false
```
Beware that you **NEED** to follow this pattern: dotnet_diagnostic.<RULE_ID>.<ATTRIBUTE>

Here's the code after adding support for multiple namespaces:

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
                            var location = GetNameOrIdentifierLocation(node);
                            if (location != null)
                            {
                                context.ReportDiagnostic(Diagnostic.Create(Rule, Location.None));
                            }
                        }
                    }
                    else
                    {
                        foundNode = true;
                    }
                }
            }
        }

         internal static Location GetNameOrIdentifierLocation(SyntaxNode member)
        {
            Location location = null;
            location = location ?? (member as PropertyDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as FieldDeclarationSyntax)?.Declaration?.Variables.FirstOrDefault()?.Identifier.GetLocation();
            location = location ?? (member as MethodDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as ConstructorDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as DestructorDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as BaseTypeDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as NamespaceDeclarationSyntax)?.Name.GetLocation();
            location = location ?? (member as UsingDirectiveSyntax)?.Name.GetLocation();
            location = location ?? (member as ExternAliasDirectiveSyntax)?.Identifier.GetLocation();
            location = location ?? (member as AccessorDeclarationSyntax)?.Keyword.GetLocation();
            location = location ?? (member as DelegateDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as EventDeclarationSyntax)?.Identifier.GetLocation();
            location = location ?? (member as IndexerDeclarationSyntax)?.ThisKeyword.GetLocation();
            location = location ?? member.GetLocation();
            return location;
        }
    }
}
```
- As you can see it's really easy to implement but let me break down the key aspects:

I start by reading the value from the editorconfig using the AnalyzerConfigOptionsProvider API.

```csharp
var config = context.Options.AnalyzerConfigOptionsProvider.GetOptions(context.Tree);
config.TryGetValue("dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file", out var configValue);
 ```

Also I  implemented a default behaviour. So if the end user do not want to set the value on the editorconfig I assume that we allow to have multiple namespaces in the same csharp file and simply return and don't validate anything.

```csharp
if (string.IsNullOrEmpty(configValue))
{
    return;
}
 ```

And only report a diagnostic if we have found a previous namespace on the same file and the "use_multiple_namespaces_in_a_file" is set to false

```csharp

if (foundNode)
{
    if (configValue.Equals("false", StringComparison.InvariantCultureIgnoreCase))
    
    {
        var location = GetNameOrIdentifierLocation(node);
        if (location != null)
        {
            context.ReportDiagnostic(Diagnostic.Create(Rule, Location.None));
        }
    }
}
else
{
    foundNode = true;
}
```

- As you can see it's really easy to add Editorconfig support for you Roslyn Analyzers and it's a really nice bonus you can use to support multiple code styles without the need to write a lot of boilerplatecode. 


# Useful Resources
- https://github.com/dotnet/roslyn-analyzers/blob/master/docs/Analyzer%20Configuration.md
- https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/code-style-rule-options?view=vs-2017
- https://docs.microsoft.com/en-us/visualstudio/code-quality/analyzers-faq?view=vs-2019
- https://github.com/dotnet/roslyn-analyzers
