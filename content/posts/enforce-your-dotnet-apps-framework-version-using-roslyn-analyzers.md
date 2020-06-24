---
title: "Enforce the use of a specific .NET Core version using Roslyn"
date: 2020-06-24T15:25:42+02:00
tags: ["dotnet", ".NET", "roslyn", "analyzers", "c#"]
draft: true
---

In these past few years Microsoft has kept a steady flow of new .NET Core versions: .NETCore 1.0, 1.1, 2.0, 2.1, 2.2 and so on and so forth.   

If you have hundreds of applications in your company it's almost impossible to keep them updated to the most recent version of the framework, so most probably you're going to end  having multiple versions running at the same time.   

Another factor to consider if that only a few of those versions are long-time support (LTS). But what does it means exactly?  It means that only a few versions are supported for three years after the initial release.

> If you want to know more about the .NET Core support policy click here: [.NET Core official support](https://dotnet.microsoft.com/platform/support/policy/dotnet-core)

Here is a table showing which .NET Core versions are LTS and when the support ends.

| Version        | Latest Update           | Support Ends  |
| ------------- |:-------------:| -----:|
| .NET Core 1.0     | 1.0.16 | June 27,2019 |
| .NET Core 1.1     | 1.1.13 | June 27,2019 |
| .NET Core 2.0     | 2.0.9 | October 1, 2018 |
| .NET Core 2.1     | 2.1.19 (LTS) | August 1, 2021 |
| .NET Core 2.2     | 2.2.8 | December 23, 2019 |
| .NET Core 3.0     | 3.0.3 | March 3, 2020 |
| .NET Core 3.1     | 3.1.5(LTS) | December 3, 2022 |


So if you are working in a company and you have to create a new .NET Core application and your company is working only with NET Core 2 you should ask yourself, what version do I use? .NET Core 2.1 is an LTS release, should I use that version? Or should a newer version like .NET Core 2.2? I'm going to migrate my app to .NET Core 3 in the near future?

The same question can be arised when moving to .NET Core 3, should I use .NET Core 3.1 or .Net Core 3.0? That one is easier to respond because in that case the LTS version is the newer one and every developer loves to use the latest tech possible.

To avoid people having these kind of questions we could ask ourselves, how can I reassure that everyone on my team is going to use the correct framework version when they have the need to create a new application?  
An easy response to that qüestion could be: Using our own **Roslyn Analyzer**. 

>  For those unaware:
>- Roslyn is the compiler platform for .NET. It consists of the compiler itself and a powerful set of APIs to interact with the compiler.  
> - Roslyn code analyzers can inspect your C# code an enforce your own style, quality and maintainability rules. 
> - Roslyn code analyzers can be installed per-project via a NuGet package.  

One thing I always advise people is that if they have some time to spare they should try to build their own roslyn analyzers containing their own code rules, pack them in a nuget and enforce that every application they create have the nuget installed.  
That's an easy way to reassure that every application enforces your company coding rules.

Well, anyways, I could keep rambling about Roslyn for a long time, but let's focus on building a roslyn analyzer that validates the framework version of our application.

The C# code would be something like this:
  

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Diagnostics;
using System.Collections.Immutable;
using System.Linq;

namespace ATC.Analyzers.Rules
{
    [DiagnosticAnalyzer(LanguageNames.CSharp)]
    public class CheckNetCoreFrameworkVersionRule : DiagnosticAnalyzer
    {
        private readonly string[] _allowedFrameworkVersions = { "2.1", "3.1" };

        private const string CoreFrameworkString = ".NETCoreApp,Version=v";
        private const string DiagnosticId = "MyRule0001";
        private const string Title = "Application framework not allowed" ;
        private const string MessageFormat = "The application '{0}' has the attribute: '{1}', that does not match with the supported framework versions: '{2}'";
        private const string Description = "Right now we are only supporting applications targeting a LTS .NETCore version.";
        private const string HelpLink = "https://dotnet.microsoft.com/platform/support/policy/dotnet-core";
        private const string Category = "CustomRules.Maintenability";

        private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(
            DiagnosticId,
            Title,
            MessageFormat,
            Category,
            DiagnosticSeverity.Error, 
            true,
            description: Description,
            helpLinkUri: HelpLink);

        public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => 
            ImmutableArray.Create(Rule);

        public override void Initialize(AnalysisContext context)
        {
            context.RegisterCompilationAction(AnalyzeMethod);
        }

        private void AnalyzeMethod(CompilationAnalysisContext context)
        {
            var attrs = context.Compilation.Assembly.GetAttributes()
                .FirstOrDefault(attr => attr.ToString().Contains("TargetFrameworkAttribute"));

            if (attrs != null)
            {
                var attrString = attrs.ToString();
                
                var found = _allowedFrameworkVersions
                    .Select(version => $"{CoreFrameworkString}{version}")
                    .Any(pattern => attrString.Contains(pattern));

                if (!found)
                {
                    var diagnostic = Diagnostic.Create(
                        Rule,
                        Location.None,
                        context.Compilation.AssemblyName,
                        attrString,
                        _allowedFrameworkVersions.Aggregate((result, next) => $"{next}, {result}"));

                    context.ReportDiagnostic(diagnostic);
                }
            }
        }
    }
}
```

Let's describe what the analyzer does:

- Obtains the attribute "TargetFrameworkAttribute" from the Compilation object.
   
> The TargetFramework attribute identifies the version of the .NET Framework that a particular assembly was compiled against.  
The TargetFrameworkAttribute attribute can specify a FrameworkDisplayName property to provide a more descriptive .NET Framework version string that is suitable for displaying to clients of the assembly.   
Source: https://msdn.microsoft.com/en-us/library/system.runtime.versioning.targetframeworkattribute(v=vs.110).aspx
  
The attribute has the following format ".NETCoreApp,Version=vX.Y", where X.Y is the framework application version.  
For example in a .NET Core 3.1 console application the value is : 
_System.Runtime.Versioning.TargetFrameworkAttribute(".NETCoreApp,Version=v3.1", FrameworkDisplayName = "")_
  
- For every version we want to enforce we build the pattern .NETCoreApp,Version=vX.Y. In our case we are build 2 patterns: .NETCoreApp,Version=v2.1 and .NETCoreApp,Version=v3.1 patterns.
- Check if the attribute value matches any of our patterns and if it doesn't, raise an error.
  
Let's test the analyzer. When we install the analyzer in a .NET Core 2.2 application we get the following error:

![Framework error](/img/roslyn-framework-error.png)

In our Roslyn Analyzer we are looking for the **TargetFrameworkAttribute** attribute and only raising an error if the value **.NETCoreApp,Version=vX.Y** does not match, but we could apply the same principle with a .NET Framework.  
Instead of looking for the value **.NETCoreApp,Version=vX.Y** we could search for the value **.NETFramework,Version=vX.Y**


