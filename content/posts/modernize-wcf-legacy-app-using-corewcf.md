---
title: "How to modernize a legacy .NET Framework WCF app using CoreWCF and .NET 7"
date: 2023-01-17T18:43:16+01:00
draft: true
description: "CoreWCF is a port of the Windows Communication Foundation (WCF) framework to the latest .NET version. The main goal of the CoreWCF project is to enable existing WCF services to be moved to .NET 7. In this post I'll show you step by step how easy (or hard) is the process of migrating an existing .NET 4.7 WCF app to .NET 7 using the CoreWCF project."
tags: ["dotnet", "wcf", "soap"]
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/modernize-wcf-app-using-corewcf).

If you are not old enough, probably you're asking yourself: "What's a WCF?"   
WCF stands for "Windows Communication Foundation", and if you want to know more about you can read the official Microsoft docs: [here](https://learn.microsoft.com/en-us/dotnet/framework/wcf/whats-wcf)

Nowadays no one is going to create a WCF from zero, it makes no sense, it's a deprecated technology.   
But if you're working for a company that's been around long enough and uses .NET as its main tech stack, it's pretty likely for them to have some legacy applications laying around, and I'm quite confident that some of them are going to be WCF applications.

The WCF framework is not compatible with .NET Core or the latest .NET version (right now, its .NET 7), it only works with .NET Full Framework. 

Microsoft recommendation if you want to modernize a WCF app is to move to gRPC, but migrating to gRPC it's not always viable: 
- Maybe your app must be consumed using SOAP ( e.g. some government services)
- Maybe gRPC doesn't support some features that WCF does (e.g. Windows integrated authentication)
- Maybe your WCF app is been consumed by some clients and you can't force them to move to another paradigm like REST or gRPC.   

So, what are our options to modernize a WCF app if for some reason we can't move away from the WCF framework? 
- The response to this question is: **CoreWCF**.   

CoreWCF is a port of the Windows Communication Foundation (WCF) framework to the latest .NET version. The main goal of the CoreWCF project is to enable existing WCF services to be moved to .NET 7.   
And in this post **I want to find out how easy (or hard) is the process of modernizing a .NET 4.7 WCF app using the CoreWCF project.**

To test this modernization process I decided not to use a "Hello World" app, because there is no real value on that.   
Instead, I'll try to migrate some old WCF app that I have found on my computer laying around, it is a more realistic app that uses an N-Layer and a Domain-Driven Design (DDD) architecture.


# **CoreWCF**

CoreWCF is a community-driven .NET Foundation project that makes WCF surface area available on modern versions of .NET. It is not a Microsoft-owned project, but Microsoft provides support for CoreWCF. 

CoreWCF isn't a straight port of the WCF framework - there are a few changes that were needed as part of the port:

- Using ASP.NET Core as the service host, push pipeline, and the middleware pattern for extensibility.
- Removing the obsolete Asynchronous Programming Model (APM) programming pattern as it made the codebase incredibly hard to work with, which isn't desirable for a community project wanting to encourage external contributions.
- Removing platform-specific and IO code. Refactoring apps into microservices and Linux-based containers is a common requirement, and so CoreWCF needs to be able to run anywhere that .NET core can be run.

CoreWCF supports many common WCF scenarios, but it does not support all WCF functionality.   
Be aware that your modernization experience may vary depending on the overlap between your WCF usage and the functionality included in CoreWCF.

# **Application**

The app we're going to modernize has the following features:
- Built entirely using .NET 4.7.2
- Uses an N-Layer and a Domain-Driven Design architecture.
- It is comprised by the following projects: 
    - BookingMgmt.WCF.WebService
    - BookingMgmt.Contracts
    - BookingMgmt.Application
    - BookingMgmt.Domain
    - BookingMgmt.Database
    - BookingMgmt.SharedKernel
- It also has a couple of unit tests projects and an integration test project:
    - BookingMgmt.Application.UnitTest
    - BookingMgmt.Domain.UnitTest
    - BookingMgmt.WCF.WebService.IntegrationTest

Here's a look at how it looks in Visual Studio.

![corewcf-project-distribution](/img/corewcf-project-distribution.png)

> If you want to take a look at the source code, click [here](https://github.com/karlospn/modernize-wcf-app-using-corewcf/tree/main/before).

But, what this app does? It manages flight bookings. It allows us to do the following actions:
- Create a new booking.
- Cancel a booking.
- List all active bookings.
- List all canceled bookings.

# **Start modernizing the app**

Let's begin by defining the end goal for this app. The desired output is the following one:
- Move from the WCF framework to the CoreWCF one.
- Upgrade the WCF project TFM from .NET 4.7.2 to .NET 7.
- Upgrade the rest of the projects that comprise this application (BookingMgmt.Contracts, BookingMgmt.Application, BookingMgmt.Domain and BookingMgmt.SharedKernel) from .NET 4.7.2 to .NET Standard.
- Upgrade the test projects TFM from .NET 4.7.2 to .NET 7.
- Have the application running on a Linux container.

In the following sections we will be doing the tasks listed above.

# **1. Moving the BookingMgmt.WCF.WebService project to CoreWCF**

The first step is to move away from the WCF framework and begin using CoreWCF.

## **1.1. Trying to use the .NET Upgrade Assistant tool to migrate the WCF project**

.[.NET Upgrade Assistant tool](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant) is a command-line (CLI) tool that helps users interactively upgrade projects from .NET Framework to .NET Standard and .NET 7. 
This tool is able to assists you in the upgrade of a series of projects types. Currently, the tool supports the following project types:
- ASP.NET MVC
- Windows Forms
- Windows Presentation Foundation (WPF)
- Console
- Libraries
- UWP to Windows App SDK (WinUI)
- Xamarin.Forms to .NET MAUI

It can also help you upgrade a WCF project to CoreWCF. You can read more about how to do it in the next link:
- https://devblogs.microsoft.com/dotnet/migration-wcf-to-corewcf-upgrade-assistant/

Let's give it a shot.

```bash
$ upgrade-assistant upgrade BookingMgmt.WCF.WebService.csproj
-----------------------------------------------------------------------------------------------------------------
Microsoft .NET Upgrade Assistant v0.4.355802+b2aeae2c0e41fbfed35df6ab2e88b82a0c11be2b

We are interested in your feedback! Please use the following link to open a survey: https://aka.ms/DotNetUASurvey
-----------------------------------------------------------------------------------------------------------------

[17:25:38 ERR] Support for WCF Server-side Services is limited to .NET Full Framework. Consider updating to use CoreWCF (https://aka.ms/CoreWCF/migration) in later provided steps or rewriting to use gRPC (https://aka.ms/migrate-wcf-to-grpc).
[17:25:38 ERR] Project E:\Coding\Dotnet\modernize-wcf-app-using-corewcf\before\BookingMgmt.WCF.WebService\BookingMgmt.WCF.WebService.csproj uses feature(s) that are not supported. If you would like upgrade-assistant to continue anyways please use the "--ignore-unsupported-features" option.
[17:25:38 ERR] Unable to upgrade project E:\Coding\Dotnet\modernize-wcf-app-using-corewcf\before\BookingMgmt.WCF.WebService\BookingMgmt.WCF.WebService.csproj
```

As you can see, the .NET Upgrade Assistant doesn't work from the get-go, the reason is because this app is a web-hosted one that uses a .svc file, and that kind of WCF is not supported.    
.NET Upgrade Assistant does not support the following WCF scenarios:

❌ WCF server that are Web-hosted and use a .svc file.    
❌ Behavior configuration via xml config files except serviceDebug, serviceMetadata, serviceCredentials (clientCertificate, serviceCertificate, userNameAuthentication, and windowsAuthentication).   
❌ Endpoints using bindings other than NetTcpBinding and HTTP-based bindings.   


## **1.2. Creating a new CoreWCF project from scratch**

Our approach of using the .NET Upgrade tool didn't pan out, so instead of that I'm going to **create a new CoreWCF project from scratch and move the source code from the WCF project to this new one.**

The best way to create a new CoreWCF project is using the CoreWCF project template, which can be installed using the dotnet tool.

```bash
 dotnet new --install CoreWCF.Templates
```

After installing the template we can create a new CoreWCF project like this:

```bash
dotnet new corewcf -n BookingMgmt.CoreWCF.WebService
```

A new CoreWCF project has been created and here's how it looks:

![corewcf-new-corewcf-project](/img/corewcf-new-corewcf-project.png)

The first thing we have to do is to delete the demo service:  ``IService.cs``, also do not delete the ``Program.cs``.    
Now we're ready to move the source code from the WCF project to the newly created CoreWCF one.   
There is no need to move every file.
- The ``.svc`` file is not needed, we only need the ``svc.cs`` file, whicht contains the generated code.
- The ``web.config`` and the ``global.asax`` are useless, with CoreWCF the application setup is done on the ``Program.cs``.
- The ``packages.config`` is also useless, this project uses the old csproj-SDK format and we are moving away from it. 
 
Here's how the project looks like after moving everything we needed into the CoreWCF project:

![corewcf-wcf-project-files](/img/corewcf-wcf-project-files.png)


## **1.3. Setting up the Program.cs**

The WCF app uses a ``global.asax``, which allows us to write code that runs in response to "system level" events, such as the application starting, a session ending, an application error occuring, without having to try and shoe-horn that code into each and every page of your site.

CoreWCF does not uses a ``global.asax``, instead of that uses a ``Program.cs`` to set up the host and expose the services bindings.    
In fact, if you're familiar with .NET Core or .NET 7, this will look somewhat familiar because CoreWCF uses the same host and builder model, but with some extra middleware that implements WCF services and provides WSDL generation through metadata.

The next code snippet shows how the ``Program.cs`` looks like after configuring it properly.

```csharp
var builder = WebApplication.CreateBuilder();

builder.Services.AddServiceModelServices();
builder.Services.AddServiceModelMetadata();
builder.Services.AddSingleton<IServiceBehavior, UseRequestHeadersForMetadataAddressBehavior>();

builder.Services.AddSingleton<BookingCreatorService>();
builder.Services.AddTransient<IBookingCreatorApplicationServices, BookingCreatorApplicationServices>();
builder.Services.AddTransient<IBookingCreatorDomainServices, BookingCreatorDomainServices>();
builder.Services.AddTransient<IBookingCreatorService, BookingCreatorService> ();
builder.Services.AddTransient<IUnitOfWorkBookingCreator, UnitOfWorkBookingCreator>();
builder.Services.AddTransient<IBookingCreatorValidations, BookingCreatorValidations>();
builder.Services.AddTransient<IBookingFeaturesDomainServices, BookingFeaturesDomainServices>();
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

var app = builder.Build();

app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<BookingCreatorService>();
    serviceBuilder.AddServiceEndpoint<BookingCreatorService, IBookingCreatorService>(new BasicHttpBinding(BasicHttpSecurityMode.None), "/BookingCreatorService.svc");
    var serviceMetadataBehavior = app.Services.GetRequiredService<ServiceMetadataBehavior>();
    serviceMetadataBehavior.HttpsGetEnabled = false;
    serviceMetadataBehavior.HttpGetEnabled = true;
});

app.Run();
```
Let's explain a little bit what we're doing in the above code snippet.

- The ``AddServiceModelServices``, ``AddServiceModelMetadata`` method are used to setup the WCF support.

```csharp
builder.Services.AddServiceModelServices();
builder.Services.AddServiceModelMetadata();
```

- The ``UseRequestHeadersForMetadataAddressBehavior`` is used to modify the generated help page and WSDL document to reflect the hostname and port number that the request came from.     
This is useful when the machine hostname is different than the hostname that a client uses to connect to the service. For example, if the service is running in a docker container or behind a load balancer.

```csharp
builder.Services.AddSingleton<IServiceBehavior, UseRequestHeadersForMetadataAddressBehavior>();
```

- The original WCF app uses ``Autofac`` as an IoC container for dependendy injection, we're using CoreWCF alongside with .NET 7, so there is no need for an external IoC container, we can use the native container.

```csharp
builder.Services.AddSingleton<BookingCreatorService>();
builder.Services.AddTransient<IBookingCreatorApplicationServices, BookingCreatorApplicationServices>();
builder.Services.AddTransient<IBookingCreatorDomainServices, BookingCreatorDomainServices>();
builder.Services.AddTransient<IBookingCreatorService, BookingCreatorService> ();
builder.Services.AddTransient<IUnitOfWorkBookingCreator, UnitOfWorkBookingCreator>();
builder.Services.AddTransient<IBookingCreatorValidations, BookingCreatorValidations>();
builder.Services.AddTransient<IBookingFeaturesDomainServices, BookingFeaturesDomainServices>();
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```
- The ``UseServieModel`` method is used to configure which endpoints we want to expose in the host. 
- With the ``serviceBuilder.AddServiceEndpoint<BookingCreatorService, IBookingCreatorService>(new BasicHttpBinding(BasicHttpSecurityMode.None), "/BookingCreatorService.svc")`` method we're exposing an endpoint named ``BookingCreatorService.svc``, which has a ``Basic Http Binding`` and the message is NOT secured during transfer. 
- The ``HttpGetEnabled`` and ``HttpsGetEnabled`` are used to set how we're going to fetch the metadata/WSDL file. In this case it will be using Http.

```csharp
app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<BookingCreatorService>();
    serviceBuilder.AddServiceEndpoint<BookingCreatorService, IBookingCreatorService>(new BasicHttpBinding(BasicHttpSecurityMode.None), "/BookingCreatorService.svc");
    var serviceMetadataBehavior = app.Services.GetRequiredService<ServiceMetadataBehavior>();
    serviceMetadataBehavior.HttpsGetEnabled = false;
    serviceMetadataBehavior.HttpGetEnabled = true;
});
```

# **2. Using .NET Upgrade Assistant tool to migrate the rest of the projects**

In the previous section we have migrated the WCF project, but this application is using an N-Layer architecture, which means that there are still a few projects that need to be migrated from .NET 4.7.2.

In this section we're going to migrate the following projects:
- BookingMgmt.Contracts
- BookingMgmt.Application
- BookingMgmt.Domain
- BookingMgmt.Database
- BookingMgmt.SharedKernel

Those 5 projects need to be migrated from .NET 4.7.2 to .NET Standard. And also from the old csproj SDK-format (with a packages.config) to the new one.

Doin the migration manually is going to be quite tedious, but thankfully ``.NET Upgrade Assistant tool`` can do it for us, it's really easy, just execute the ``upgrade-assistant upgrade BookingMgmt.WebService.sln`` command and follow the on-screen instructions.

![corewcf-net-upgrade-tool-dependencies](/img/corewcf-net-upgrade-tool-dependencies.png)

After running the ``.NET Upgrade Assistant tool``, here's how the ``BookingMgmt.Domain.csproj`` looks like:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <OutputType>Library</OutputType>
    <SccProjectName>SAK</SccProjectName>
    <SccLocalPath>SAK</SccLocalPath>
    <SccAuxPath>SAK</SccAuxPath>
    <SccProvider>SAK</SccProvider>
    <PublishUrl>publish\</PublishUrl>
    <Install>true</Install>
    <InstallFrom>Disk</InstallFrom>
    <UpdateEnabled>false</UpdateEnabled>
    <UpdateMode>Foreground</UpdateMode>
    <UpdateInterval>7</UpdateInterval>
    <UpdateIntervalUnits>Days</UpdateIntervalUnits>
    <UpdatePeriodically>false</UpdatePeriodically>
    <UpdateRequired>false</UpdateRequired>
    <MapFileExtensions>true</MapFileExtensions>
    <ApplicationRevision>0</ApplicationRevision>
    <ApplicationVersion>1.0.0.%2a</ApplicationVersion>
    <IsWebBootstrapper>false</IsWebBootstrapper>
    <UseApplicationTrust>false</UseApplicationTrust>
    <BootstrapperEnabled>true</BootstrapperEnabled>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
    <RunCodeAnalysis>false</RunCodeAnalysis>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">
    <RunCodeAnalysis>true</RunCodeAnalysis>
  </PropertyGroup>
  <ItemGroup>
    <BootstrapperPackage Include="Microsoft.Net.Client.3.5">
      <Visible>False</Visible>
      <ProductName>.NET Framework 3.5 SP1 Client Profile</ProductName>
      <Install>false</Install>
    </BootstrapperPackage>
    <BootstrapperPackage Include="Microsoft.Net.Framework.3.5.SP1">
      <Visible>False</Visible>
      <ProductName>.NET Framework 3.5 SP1</ProductName>
      <Install>true</Install>
    </BootstrapperPackage>
    <BootstrapperPackage Include="Microsoft.Windows.Installer.3.1">
      <Visible>False</Visible>
      <ProductName>Windows Installer 3.1</ProductName>
      <Install>true</Install>
    </BootstrapperPackage>
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="System.Data.DataSetExtensions" Version="4.5.0" />
    <PackageReference Include="Microsoft.DotNet.UpgradeAssistant.Extensions.Default.Analyzers" Version="0.4.355802">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="fasterflect" Version="2.1.3" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\BookingMgmt.SharedKernel\BookingMgmt.SharedKernel.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="log4net" Version="1.2.10" />
    <PackageReference Include="Newtonsoft.Json" Version="12.0.2" />
  </ItemGroup>
</Project>
```
As you can see, the project has been converted to the new SDK-format and also the TFM has been upgraded from .NET 4.7.2 to .NET Standard 2.0. 

Every project attribute present has been ported as well, but if you take a look at them, it's pretty clear that most of them are useless right now. So, a good tip after doing a project migration is to take a good look at the resulting ``.csproj`` file because usually it can be cleaned up of attributes quite a bit.

# **3.Mending incompatible dependencies**

The first step has been to upgrade the WCF project (manually) to .NET 7 and then with the help of the ``.NET Upgrade Assistant tool`` we have upgraded the rest of the application projects to .NET Standard 2.0, but some packages incompatibilities still remain.

The ``BookingMgmt.Infrastructure`` project and the ``BookingMgmt.SharedKernel`` contains a reference to the version 6.1.1 of ``EntityFramwork 6``, and this version is NOT compatible with .NET Standard.    

A possible solution for this incompatibility is to upgrade the ``EntityFramework 6`` package to the latest version (right now, 6.4.4).   
The problem with this upgrade is that the 6.4.4 version is only compatible with .NET Standard 2.1 and our projects are targeting .NET Standard 2.0, so **we're going to manually upgrade every project that targets .NET Standard 2.0 to .NET Standard 2.1**.

The other troublesome dependencies are: ``fasterflect`` and ``log4net``, both packages are not compatible with .NET Standard 2.1.   
The good news here is that after moving to .NET 7 I have no real use for those packages, so I can remove them.    
The ``log4net`` package can be replaced by the native .NET ``ILogger`` API and the ``fasterflect`` dependency can be completely removed from the application.

# **4. Migrating application settings from the web.config to IConfiguration**

The application settings on the original WCF app are defined on the ``web.config``, more specifically on the ``AppSettings`` section.   
The only settings that this app requires is the database connection string.

```xml
  <appSettings>
    <add key="DatabaseConnectionString" value="Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=Flights;Integrated Security=True;Connect Timeout=60;Encrypt=False;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False" />
  </appSettings>
```
The ``DatabaseConnectionString`` setting is used via the ``ConfigurationManager`` API. This setting is needed for setting up the EntityFramework DbContext.   
The next code snippet shows how we're using the ``DatabaseConnectionString`` app setting.

```csharp
    public class BookingCreatorContext : DbContextBase
    {
        static BookingCreatorContext()
        {
            Database.SetInitializer<BookingCreatorContext>(null);

        }

        public BookingCreatorContext()
            : base(ConfigurationManager.AppSettings["DatabaseConnectionString"])
        {
            Configuration.LazyLoadingEnabled = false;
        }

        public virtual DbSet<Booking> Bookings { get; set; }
        public virtual DbSet<Journey> Journeys { get; set; }
    }
```

Our new CoreWCF app doesn't require a ``web.config``, so we must move those application settings somewhere else, with .NET 7 we can use the ``IConfiguration`` native implementation.     

We can simply move the ``DatabaseConnectionString`` setting to the ``appsettings.json`` and replace the ``ConfigurationManager.AppSettings`` method for ``IConfiguration``.

# **5. Running the app locally**

After fixing the NuGet incompatibilities and moving the application settings into the ``appsettings.json``, we're ready to run our new CoreWCF app.

If we open a browser on the following address: ``http://localhost:5186/BookingCreatorService.svc``, we'll be presented with the svc html page.

![corewcf-svc-html](/img/corewcf-svc-html.png)

If we send a few requests to our app, it responds successfully.

![corewcf-wcftestclient-request](/img/corewcf-wcftestclient-request.png)

We can say that we have successfully migrated the BookingMgmt app to CoreWCF and .NET 7.    
But there are still a few things left to do.

# **6. Manually migrating test projects**

The unit and integration tests are still on .NET 4.7.2 and need to be moved to .NET 7. 

The ``.NET Upgrade Assistant tool`` is unable to migrate MSTest project types, which means that the only possible way to migrate those projects is manually.    
For every test project I'm going to create a new MSTest project that targets .NET 7 and move over the original source code.

![corewcf-mstest-project-type](/img/corewcf-mstest-project-type.png)

> These test projects are using MSTest, not xUnit. Now would be a good time to also move them from MSTest to xUnit, but then a code refactor is needed, so I'm not going to do it.

The test projects are using the ``NMock3`` package to mock dependencies, this package is not compatible with .NET 7, which means that we need to use an alternative.    

I decided to switch over to ``Moq``, but that means that a code refactor needs to be put in place to adjust the mock dependencies.

The next code snippet shows a pair of Unit Tests where the mock dependencies package have been switched from ``NMock3`` to ``Moq``.

```csharp
namespace BookingMgmt.Domain.UnitTest.GivenBooking
{
    [TestClass]
    public class WhenCancelBooking
    {
        private static IBookingCreatorDomainServices _sut;
        private static Mock<IUnitOfWorkBookingCreator> _unitOfWorkBookingCreator;
        private static Mock<IRepository<Booking>> _bookingRepository;
        private static Mock<IRepository<Journey>> _journeyRepository;
        private static Mock<IBookingFeaturesDomainServices> _bookingFeaturesDomainServices;

        [ClassInitialize]
        public static void ClassInitialize(TestContext context)
        {
            _unitOfWorkBookingCreator = new Mock<IUnitOfWorkBookingCreator>();
            _bookingRepository = new Mock<IRepository<Booking>>();
            _journeyRepository = new Mock<IRepository<Journey>>();
            _unitOfWorkBookingCreator.Setup(x => x.GetRepository<Booking>()).Returns(_bookingRepository.Object);
            _unitOfWorkBookingCreator.Setup(x => x.GetRepository<Journey>()).Returns(_journeyRepository.Object);
            _bookingFeaturesDomainServices = new Mock<IBookingFeaturesDomainServices>();
    
            var  iBookingCancelerValidations = new BookingCreatorValidations(_bookingFeaturesDomainServices.Object);   
            _sut = new BookingCreatorDomainServices(_unitOfWorkBookingCreator.Object, iBookingCancelerValidations);
        }

        [TestMethod]
        public void Then_BookingIsCanceled()
        {

            var salesAgent = "web";
            var amountOfBooking = 1;

            var bookingsToCancel = BookingMockedProvider.GetBookings(salesAgent, amountOfBooking);
            var firstModificationDate = bookingsToCancel.SingleOrDefault().Modified;
            var journeysAmountToCancel = bookingsToCancel.SingleOrDefault().Journeys.Count;

            _bookingRepository.Setup(x => x.Get(
                It.IsAny<Expression<Func<Booking, bool>>>(), 
                It.IsAny<Func<IQueryable<Booking>, IOrderedQueryable<Booking>>>(), 
                It.IsAny<List<Expression<Func<Booking, object>>>>(), 
                It.IsAny<int?>(), 
                It.IsAny<int?>(), 
                It.IsAny<bool>())).Returns(bookingsToCancel);
            _journeyRepository.Setup(x => x.Delete(It.IsAny<Journey>()));
            _bookingRepository.Setup(x => x.UpdateRootEntity(It.IsAny<Booking>()));
            _unitOfWorkBookingCreator.Setup(x => x.Save()).Returns(journeysAmountToCancel);
            _bookingFeaturesDomainServices.Setup(x => x.IsAgency(It.IsAny<Booking>())).Returns(false);
            
            _sut.CancelBooking(bookingsToCancel.First().Id);

            Assert.IsTrue(bookingsToCancel.SingleOrDefault()?.Modified > firstModificationDate, "Property Modified in Booking was not modified.");
        }


        [TestMethod]
        [ExpectedException(typeof(InvalidBookingOperationException))]
        public void Then_BookingIsNotCanceledWhenAgency()
        {

            var salesAgent = "agency";
            var amountOfBooking = 1;

            var bookingsToCancel = BookingMockedProvider.GetBookings(salesAgent, amountOfBooking);
            var journeysAmountToCancel = bookingsToCancel.SingleOrDefault().Journeys.Count;

            _bookingRepository.Setup(x => x.Get(
                It.IsAny<Expression<Func<Booking, bool>>>(),
                It.IsAny<Func<IQueryable<Booking>, IOrderedQueryable<Booking>>>(),
                It.IsAny<List<Expression<Func<Booking, object>>>>(),
                It.IsAny<int?>(),
                It.IsAny<int?>(),
                It.IsAny<bool>())).Returns(bookingsToCancel);
            _journeyRepository.Setup(x => x.Delete(It.IsAny<Journey>()));
            _bookingRepository.Setup(x => x.UpdateRootEntity(It.IsAny<Booking>()));
            _unitOfWorkBookingCreator.Setup(x => x.Save()).Returns(journeysAmountToCancel);
            _bookingFeaturesDomainServices.Setup(x => x.IsAgency(It.IsAny<Booking>())).Returns(true);
            
            _sut.CancelBooking(bookingsToCancel.First().Id);
        }
    }
}
```


# **7. Running the app on a Linux container**

The last step is to run this new version of the app on a Linux container.     

Why the purpose of doing this? To test no dependency with the WCF framework remains (which only runs on Windows OS).

The next code snippet shows how the application ``Dockerfile`` looks like.
```yaml
FROM mcr.microsoft.com/dotnet/sdk:7.0-bullseye-slim AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.sln ./
COPY /BookingMgmt.CoreWCF.WebService/*.csproj ./BookingMgmt.CoreWCF.WebService/
COPY /BookingMgmt.Application/*.csproj ./BookingMgmt.Application/
COPY /BookingMgmt.Contracts/*.csproj ./BookingMgmt.Contracts/
COPY /BookingMgmt.Domain/*.csproj ./BookingMgmt.Domain/
COPY /BookingMgmt.Infrastructure/*.csproj ./BookingMgmt.Infrastructure/
COPY /BookingMgmt.SharedKernel/*.csproj ./BookingMgmt.SharedKernel/
RUN dotnet restore "./BookingMgmt.CoreWCF.WebService/BookingMgmt.CoreWCF.WebService.csproj" -s "https://api.nuget.org/v3/index.json"

# Copy everything else and build
COPY . ./
RUN dotnet publish BookingMgmt.CoreWCF.WebService/*.csproj -c Release -o /app/publish

# Build runtime imagedock
FROM mcr.microsoft.com/dotnet/aspnet:7.0-bullseye-slim
WORKDIR /app
COPY --from=build-env /app/publish .
ENTRYPOINT ["dotnet", "BookingMgmt.CoreWCF.WebService.dll"]
```

To run the app and its dependencies (MS SQL) I'm going to use a ``docker-compose`` file. The next code snippet shows how it looks like.
```yaml
version: '3.8'

networks:
  wcf:
    name: flights-network
    
services:
  mssql:
    build: 
      context: ./scripts/sql
    ports:
      - "1433:1433"  
    environment:
      SA_PASSWORD: "P@ssw0rd?"
      ACCEPT_EULA: "Y"
    networks:
      - wcf

  app:
    build:
      context: ./
      dockerfile: ./Dockerfile
    depends_on:
      - mssql
    ports:
      - 5001:80
    environment:
      DatabaseConnectionString: Server=mssql;Database=Flights;User Id=SA;Password=P@ssw0rd?
    networks:
      - wcf
```

And when we execute the ``docker compose up`` command, the app run successfully!

# **Wrap-up**

> If you want to take a look at the source code of the migrated WCF app, you can go: [here](https://github.com/karlospn/modernize-wcf-app-using-corewcf/tree/main/after).

Moving a WCF from the WCF framework to the CoreWCF implementation is not very complicated, unless your WCF uses some of the more obscure configurations.    
Probably the hardest thing when trying to modernize this type of apps is to move away from .NET Framework, that's because there is a big probability that your app is using some dependencies that are not compatible with .NET 7 or .NET Standard 2.x.

If your app has some dependencies that can't be used with .NET 7 or .NET Standard 2.x, you'll have a few options available:
- The best option avaiable is that there is a newer version available of the same package that is compatible, this is what happened on the above example and ``Entity Framework 6``.
- Find an package that does a similar functionality.
- Build yourself a new version of the package.
- Remove the package completelly and build the business logic by yourself.   

Except the first one, the rest of the options are going to be a time sunk and require a code refactor.

The WCF Framework only runs in Windows platform, which means that maybe you could be using a native Windows API, and then the migration process becomes even worse.

The modernization process of the BookingMgmt WCF app has been relatively quick and painless. I have spent between 4 and 5 hours to successfully migrate to a .NET 7 CoreWCF app that is hosted on a linux containers.   
This app had very few incompatible dependencies, only ``EF6``, ```log4net`` and ``fasterflect``, and the fix was really easy and quick.

Also, I did the minimum of work required to make the app run in a linux containers, the app could have been further improved, e.g. we could have move from EF 6 to EFCore 7, which probably would have a give us a performance boost.
We could also move the test project from MSTest to a newer project type like xUnit.