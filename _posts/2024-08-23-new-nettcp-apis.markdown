---
# Layout
layout: post
title:  "New configuration API for NetTcp"
date:   2024-08-23 12:00:00 -0800
categories: release
# Author
author: Matt Connew(https://github.com/mconnew)
---
# Introduction
The CoreWCF project released a security fix in March of 2024 releated to connection initialization timeouts. The problem was caused by the ChannelInitializationTimeout not being used consistently. This setting can be found on the connection pool settings of the relevant transport binding element (e.g. TcpTransportBindingElement). This timeout is used to limit how long a client is allowed to take to complete the initial connection handshake. This includes the client informing the server which endpoint it's connecting to, and completing any security upgrade of the connection, e.g. wrapping the communication with `SslStream` if using certificate authentication. The vulnerability was that early in the connection handshake, a read was made from the incoming connection without applying this timeout. This allowed a client to connect to a service and cause a socket to be indefinitely allocated, never needung to send any data.  

The security fix that was released was the minimum change required to prevent a service from being vulnerable, but in creating the minimal fix it became clear that there was a mismatch between the api to configure this timeout and the implementation. If you have multiple endpoints listening on the same port with different timeouts configured, CoreWCF doesn't know which endpoint a client is connecting to until part way through the handshake. Each endpoint could have a different timeout configured. The security patch released uses the first NetTcp binding configured in the service for the timeout. This isn't too dissimilar to how WCF on .NET Framework configures the timeout. The difference with WCF is that after the first endpoint is configured to listen, subsequent endpoints using the same port are compared to see if their binding is compatible, and if it isn't, will throw an exception and fault the entire ServiceHost, whereas CoreWCF will keep using the configuration from the first endpoint.  

The hosting model for CoreWCF is significantly different due to being built on top of ASP.NET Core. Setting up the listening port is done at the start of the host startup before CoreWCF begins initialization and is configured independently from the service endpoint which contains the binding and listen Uri. This enables a cleaner solution as there's a single place to configure any common properties such as the connection initialization timeout.  

NetNamedPipe had already introduced a newer configuration api shape which is closer in design to the ASP.NET Core Kestrel configuration api's. As NetNamedPipe, NetTcp, and UnixDomainSocket all share common code in NetFramingBase, having a shared base implementation will also result in simpler code.  

The result of this is a new configuration api for using NetTcp. The existing api's will continue to be available as they've been reimplemented using the new api. All your existing code will continue to so there's no code change needed unless you wish to modify any of the connection properties exposed by the newer api.

# How this works

### Configuring a CoreWCF Service
Let's start with an existing CoreWCF which is using NetTcp. Using ASP.NET Core's Minimal API, the bare bones setup code would look like this:

```csharp
using System.Net;

var builder = WebApplication.CreateBuilder();
builder.WebHost.UseNetTcp(IPAddress.IPv6Any, 8808);
builder.Services.AddServiceModelServices();
var app = builder.Build();

app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<Service>();
    serviceBuilder.AddServiceEndpoint<Service, IService>(
        new NetTcpBinding(), "/Service.svc");
});

app.Run();
```
You can keep using this code if you wish, or you can replace the call to `UseNetTcp` to use the new api overload which would look like this:

```csharp
builder.WebHost.UseNetTcp((NetTcpOptions options) =>
{
    options.Listen("net.tcp://localhost:8808");
});
```
Providing a uri instead of an IP address and port number is how a base address is passed to ServiceHost in WCF. The IP address used to listen for connections is now also the same behavior as WCF. If the hostname portion of the uri is an IP address, it listens on that IP. If you specify a dns or machine name, then if the OS supports IPv6, it listens on `IPAddress.IPv6Any`, which listens on all IPv6 and IPv4 addresses. If the OS doesn't support IPv6, then it will listen on `IPAddress.Any`, which listens on all IPv4 addresses. The hostname "localhost" has no special treatment and will result in listening on all addresses. This again matches the behavior of WCF and hopefully minimizes any confusion when porting services from WCF to CoreWCF. If you wish to only accept connections on the loopback address, specify the loopback IP address as the hostname, ie `options.Listen("net.tcp://127.0.0.1:8808")`.  

If you wish to modify one of the properties of the listening port, then you can provide a delegate to the `NetTcpOptions.Listen` method to modify those properties. Here's an example:

```csharp
builder.WebHost.UseNetTcp((NetTcpOptions options) =>
{
    options.Listen("net.tcp://localhost:8808", (TcpListenOptions listenOptions) =>
    {
        listenOptions.ConnectionPoolSettings.ChannelInitializationTimeout = 
            TimeSpan.FromSeconds(5);
    });
});
```

Both the `NetTcpOptions` and `TcpListenOptions` classes have the property `IServiceProvider ApplicationServices { get; }` defined to make it more convenient to retrieve values from DI. This can make scenarios such as using configuration to store your listen address easier to implement, especially if you don't use an anonymous delegate for your configuration when you might not be able to access anything outside of the passed in parameters. There are overloads of `NetTcpOptions.Listen` that take a `Uri` instead of a string.  

If you wish to have a second service listening on a different port, there are 2 changes you need to make. The first is a second call to `NetTcpOptions.Listen` to specify the second port you wish to listen on. The second change would be to use an overload of `AddService<TService>` which takes a delegate to configure `ServiceOptions`. In this configuration delegate and you would configure the specific base address with the port number you are using for that service. The complete solution would look like this:

```csharp
var builder = WebApplication.CreateBuilder();
builder.WebHost.UseNetTcp((NetTcpOptions options) =>
{
    options.Listen("net.tcp://localhost:8808");
    options.Listen("net.tcp://localhost:8809");
});

builder.Services.AddServiceModelServices();

var app = builder.Build();
app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<Service>((ServiceOptions serviceOptions) =>
    {
        serviceOptions.BaseAddresses.Clear();
        serviceOptions.BaseAddresses.Add(new Uri("net.tcp://localhost:8808/"));
    });
    serviceBuilder.AddServiceEndpoint<Service, IService>(new NetTcpBinding(), "/Service.svc");
    serviceBuilder.AddService<Service2>((ServiceOptions serviceOptions) =>
    {
        serviceOptions.BaseAddresses.Clear();
        serviceOptions.BaseAddresses.Add(new Uri("net.tcp://localhost:8809/"));
    });
    serviceBuilder.AddServiceEndpoint<Service2, IService2>(new NetTcpBinding(), "/Service.svc");
});

app.Run();
```

The existing binding api's for settings that now appear on `TcpListenOptions` now have the `ObsoleteAttribute` applied with a message to use the new api.