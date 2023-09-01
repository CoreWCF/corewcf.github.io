---
# Layout
layout: post
title:  "Introducing Unix Domain Socket binding support in CoreWCF and WCF Client"
date:   2023-09-01 13:00:00 -0800
categories: release
# Author
author: Biroj Nayak (https://github.com/birojnayak)
---
#### Introduction
With the new V1.5.0 CoreWCF release, CoreWCF will have an additional binding called Unix Domain Socket Binding both in CoreWCF and WCF Client. The reason for enabling UDS(Unix Domain Socket) binding inside CoreWCF and WCF Client is to provide an alternative for NetNamedPipe in Linux. NetNamedPipe is only meant for windows, while UDS is allowed in Linux and Windows.

### How this works
We expose and extension method called `UseUnixDomainSocket`. As part of the extension method, we inject `UnixDomainSocketHostedService` and `UnixDomainSocketFramingOptionsSetup` into the DI . `UnixDomainSocketHostedService` is of type `IHostedService`, and `UnixDomainSocketFramingOptionsSetup` is of type `IConfigureOptions<KestrelServerOption>`.  So when initilization happens via `Ihost.StartAsyc`, the `StartAsync` method of `UnixDomainSocketHostedService` is called via asp.netcore pipeline, and that calls the `Configure` method of `UnixDomainSocketFramingOptionsSetup`, which calls the `ListenUnixSocket` method of kestrel by providing the path. Before passing the filepath, we check if path exist; if it is, we delete the path.

### Authentication Model
As of the time of release it only supports two security moodes. SecurityMode None and Transport. Under the SeucityMode Transport, we support four modes for the client and server to validate each other. Those 4 modes are `Certificate`, `Default`, `IdentityOnly` (Linux only), `Windows`. `IdentityOnly` mode was specifically introduced for Linux OS, keeping unix posix identity in mind. When `ClientCredentialType` is set to `IdentityOnly`, the server figures out the client user information who is connecting to the socket by calling few system native APIs and add that as ClaimsIdentity for future use. When ClientCredential type is set to `Default`, for Windows OS, it defaults to mode 'Window` and for Linux it sets to `IdentityOnly`.

#### Getting Started

The way to initialize service in your statuup class is

```csharp
var hostBuilder = Host.CreateDefaultBuilder(Array.Empty<string>());
hostBuilder.UseUnixDomainSocket(options =>

{
    options.Listen(new Uri("net.uds://" + "yoursocketfilepath"));
});
hostBuilder.ConfigureServices(services => services.AddServiceModelServices());
IHost host = hostBuilder.Build();
CoreWCF.UnixDomainSocketBinding serverBinding = new CoreWCF.UnixDomainSocketBinding(SecurityMode.);
host.UseServiceModel(builder =>

{
    builder.AddService<Services.TestService>();
    builder.AddServiceEndpoint<Services.TestService, ServiceContract.ITestService>(serverBinding, "net.uds://" + "yoursocketfilepath");

});
await host.StartAsync();
```
Once service started the client can be used like below,
```csharp
System.ServiceModel.UnixDomainSocketBinding binding = new UnixDomainSocketBinding(System.ServiceModel.SecurityMode.Transport);
binding.Security.Transport.ClientCredentialType = System.ServiceModel.UnixDomainSocketClientCredentialType.IdentityOnly;
var factory = new System.ServiceModel.ChannelFactory<ClientContract.ITestService>(binding, new System.ServiceModel.EndpointAddress(new Uri("net.uds://" + yoursocketfilepath)));
channel = factory.CreateChannel();
((IChannel)channel).Open();
string result = channel.EchoString(testString);
```
For using the WCF client for UDS binding please use `  <add key="ClientPackages" value="https://pkgs.dev.azure.com/dotnet/CoreWCF/_packaging/ClientPackages/nuget/v3/index.json" />` as your package source until WCF client team makes the full release.

#### Thanks

Thanks to AWS(https://aws.amazon.com/blogs/opensource/category/programing-language/dot-net/) for supporting this project since last 3 years.
