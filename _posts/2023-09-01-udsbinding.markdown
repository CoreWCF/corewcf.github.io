---
# Layout
layout: post
title:  "Introducing Unix Domain Socket binding support in CoreWCF and WCF Client"
date:   2023-09-01 13:00:00 -0800
categories: release
# Author
author: Biroj Nayak (https://github.com/birojnayak)
---
| * Update * |
| -------- |
| This Blog post has been updated with the changes from the GA release. The main difference is the introduction of the enum type `UnixDomainSocketSecurityMode`, and its value `TransportCredentialOnly`, and renaming the credential enum value `IdentityOnly` to `PosixIdentity`. This change was made to reflect that `PosixIdentity` doesn't provide any transport encryption or integrity. |

#### Introduction
With the new v1.5.0-preview1 release, CoreWCF will have an additional binding using Unix Domain Sockets (UDS). We're providing a UDS based transport for CoreWCF and the WCF Client to provide an alternative to NetNamedPipe which works on Linux. NetNamedPipe only works in Windows, while UDS is cross platform and is supported on Linux and Windows.

### How this works
We have added a new extension method for `IHostBuilder` called `UseUnixDomainSocket`. This extension method adds all the types to DI that are necessary for CoreWCF to configure ASP.NET Core for the UDS transport. We implement the UDS transport using a hosted service implementing the `IHostedService` interface. This hosted service creates its own instance of `KestrelServer` and configures it to use Unix domain sockets. We implemented it this way as a `KestrelServer` instance can only use a single transport type. This enables using Kestrel for TCP based communication for handling HTTP requests via the regular ASP.NET Core configuration mechanisms without conflicting with the need for CoreWCF to use Unix domain sockets.

### Authentication Model
In the initial release, UDS supports two security modes, `None` and `Transport`. This matches the capabilities of the NetNamedPipe binding as they are intended for use in the same scenarios. When using Transport security, we support four client credential types. These are:  
* Default
* Certificate
* Windows
* PosixIdentity - Only supported on Linux 

The client credential type of `PosixIdentity` was specifically introduced for the Linux OS. As the name suggests, it provides the service with the Posix identity of the calling client, but it does not encrypt or sign the data that flows back and forth between the client and the service. As UDS is used for communication between processes on the same host, the communication can't be observed by a 3rd party host. If you wish to keep the communication private from other processes on the same machine, we recommend you use the Certificate client credential type which will secure the communication using TLS.  

The server gets the user information for the process that owns the client end of the socket and populates the Claims information. This allows authorizing clients with a custom authorization manager without the need to manage any additional infrastructure such as certificate distribution. When a client makes a service call, the username is populated in a `GenericIdentity` wrapped in a `ClaimsIdentity` and is available from `OperationContext.ServiceSecurityContext.PrimaryIdentity`. The `ClaimsIdentity` instance will have the following claims added.  

| Claim Type | Purpose/Value |
| -------- | -------- |
| http://schemas.xmlsoap.org/ws/2005/05/identity/claims/posixgroupid | The id of the group the client process belongs to |
| http://schemas.xmlsoap.org/ws/2005/05/identity/claims/posixgroupname | The nane of the group the client process belongs to |
| http://schemas.xmlsoap.org/ws/2005/05/identity/claims/processid | The process id of the client process |
 
The client credential type `Default` uses a different credential type based on the OS. When running on Windows, it's equivalent to `Windows` and will authenticate the client the same way as NetNamedPipe authenticates. When running on Linux, it's equivalent to `PosixIdentity`.

#### Getting Started

Here is how you initialize a service in your start up class:

```csharp
var hostBuilder = Host.CreateDefaultBuilder(Array.Empty<string>());
hostBuilder.UseUnixDomainSocket(options =>
{
    options.Listen(new Uri("net.uds://" + "yoursocketfilepath"));
});
hostBuilder.ConfigureServices(services => services.AddServiceModelServices());
IHost host = hostBuilder.Build();
CoreWCF.UnixDomainSocketBinding serverBinding = new CoreWCF.UnixDomainSocketBinding(UnixDomainSocketSecurityMode.TransportCredentialOnly);
host.UseServiceModel(builder =>
{
    builder.AddService<Services.TestService>();
    builder.AddServiceEndpoint<Services.TestService, ServiceContract.ITestService>(serverBinding, "net.uds://" + "yoursocketfilepath");
});
await host.RunAsync();
```
Once the service has been started, the client can be used like below:
```csharp
var binding = new System.ServiceModel.UnixDomainSocketBinding(System.ServiceModel.UnixDomainSocketSecurityMode.TransportCredentialOnly);
binding.Security.Transport.ClientCredentialType = System.ServiceModel.UnixDomainSocketClientCredentialType.PosixIdentity;
var factory = new System.ServiceModel.ChannelFactory<ClientContract.ITestService>(binding, new System.ServiceModel.EndpointAddress(new Uri("net.uds://" + yoursocketfilepath)));
channel = factory.CreateChannel();
((IChannel)channel).Open();
string result = channel.EchoString(testString);
```
The UDS client packages have been released as part of the WCF Client project at https://github.com/dotnet/wcf as a part of the 6.2 release. The client NuGet package is called [System.ServiceModel.UnixDomainSocket](https://www.nuget.org/packages/System.ServiceModel.UnixDomainSocket).

#### Thanks

Thanks to AWS(https://aws.amazon.com/blogs/opensource/category/programing-language/dot-net/) for supporting this project for the last 3 years.
