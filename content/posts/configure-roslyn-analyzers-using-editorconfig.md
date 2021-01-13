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

> I don't pretend to write a deep dive about how a roslyn analyzers works or what's an editorconfig file.   
>

**In this post I want to show you how you can add your own attributes on an editorconfig file and how a roslyn analyzer can use those attributes to do one thing or another.**   

## Quick example

Probably it's not entirely clear what I'm trying to achieve in this post so let me show you a very quick example.

The Roslyn rule **IDE0065** is an analyzer built into Visual Studio and the EditorConfig file is used to define the preferred code style, specifically it analyzes if a using statement should be inside or outside a namespace based on a configuration attribute. (https://docs.microsoft.com/es-es/dotnet/fundamentals/code-analysis/style-rules/ide0065#csharp_using_directive_placement )

The analyzer can be configured with the value "inside_namespace" or "outside_namespace" in the editorconfig file.

- If we setup the editorconfig like this

```yaml
root = true
[*.cs]
csharp_using_directive_placement = inside_namespace:error
```
The following block of code will not report an error

```csharp
// csharp_using_directive_placement = inside_namespace
namespace Convention1
{
    using System;
    ...
}
```

But this block of code will report an error:

```csharp
// csharp_using_directive_placement = outside_namespace
using System;

namespace Convention2
{
    ...
}
```

- And if we setup the editorconfig like this:

```yaml
root = true
[*.cs]
csharp_using_directive_placement = outside_namespace:error
```
Only the first block of code (Convention1 namespace) will report an error


**And that's exactly what we want to achieve in our custom roslyn analyzer, we want to setup the analyzer behaviour based on a editorconfig attribute**

And to do this we have the **AnalyzerConfigOptionsProvider Roslyn API**


# AnalyzerConfigOptionsProvider Roslyn API

The new Roslyn API context.Options.AnalyzerConfigOptionsProvider.GetOptions allows to get the configuration from the editorconfig file.


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

```yaml
root = true

# C# files
[*.cs]

dotnet_diagnostic.MyRules0001.severity = warning
dotnet_diagnostic.MyRules0001.use_multiple_namespaces_in_a_file = false
```


# Useful Resources
- https://github.com/dotnet/roslyn-analyzers/blob/master/docs/Analyzer%20Configuration.md
- https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/code-style-rule-options?view=vs-2017
- https://docs.microsoft.com/en-us/visualstudio/code-quality/analyzers-faq?view=vs-2019
- https://github.com/dotnet/roslyn-analyzers
