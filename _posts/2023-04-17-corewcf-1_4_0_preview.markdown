---
# Layout
layout: post
title:  "CoreWCF 1.4.0 Preview release"
date:   2023-04-17 13:00:00 -0800
categories: release
# Author
author: Matt Connew
---
### Introduction

We've just released the preview1 release of CoreWCF 1.4.0, and it comes with some new transports. We're adding named pipe support along with multiple queue based transports. We're initially releasing with MSMQ and RabbitMQ support, with Apache Kafka support coming in a future release. There is a lot of new code with this release which is why we're releasing it as a preview. Different components will come out of preview at different times, and I'll talk about the milestones needed for a GA release of each of them.

### NetNamedPipe

The CoreWCF.NetNamedPipe package provides the NetNamedPipe binding that many will be familiar with from WCF. This package will only work on Windows as it uses an indirect connection protocol involving named shared memory that isn't transferable to Linux. The named pipe transport shares most of its code with the NetTcp transport, the only real difference being how to create a connection, and send/receive bytes. The connection open handshake and message framing is identical for these two transports. That common code has been moved from the NetTcp package into the NetFramingBase package and is a dependency for both transports. This makes it a lot easier to add more connection based transports in the future. We plan to add Unix domain socket support to provide an equivalent to named pipe support that can be used on Linux. Recent releases of Windows also support Unix domain sockets and we'll be able to release the Unix domain socket support across platform. 

Adding NetNamedPipe support to an app is similar to how you add NetTcp support. The API pattern has evolved a bit to make dealing with base paths a bit easier.  

```csharp
var builder = WebApplication.CreateBuilder();
builder.WebHost.UseNetNamedPipe(options =>
{
    options.Listen("net.pipe://localhost/MyService.svc");
});

builder.Services.AddServiceModelServices();

var app = builder.Build();
app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<Service>();
    serviceBuilder.AddServiceEndpoint<Service, IService>(new NetNamedPipeBinding(), "/netpipe");
});
app.Run();
```

There are overloads to the `Listen` method which takes another delegate to configure some settings which used to exist on NetNamedPipeBinding or NamedPipeTransportBindingElement but make more sense to be placed as listen options when configuring the WebHost. This API pattern matches the who you configure Kestrel. An application which uses that might look like this.

```csharp
var builder = WebApplication.CreateBuilder();
builder.WebHost.UseNetNamedPipe(options =>
{
    options.Listen(new Uri("net.pipe://localhost/service.svc/"), listenOptions =>
    {
        // Set connection buffer size to 64K
        listenOptions.ConnectionBufferSize = 64 * 1024;
    });
});
```

There are a couple of items needing to be completed before named pipe support will come out of preview. The biggest one is that there's currently no WSDL support. The second one is when stopping the WebApplication, the channel close doesn't happen cleanly. Instead of informing the client that the channel is closing, the underlying named pipe connection is abruptly closed.

### Queue based transports

When discussing bringing MSMQ support in a CoreWCF issue, I mentioned I'd like to make something generic so that CoreWCF could support multiple queue transport protocols. There was a lot of positive response to this idea, so I wrote up a brief [description](https://github.com/CoreWCF/CoreWCF/issues/164#issuecomment-908793395) of how that could look. Biroj Nayak (@birojnayak) from Amazon AWS took this description and after a few discussions wrote a detailed design [document](https://github.com/CoreWCF/CoreWCF/blob/main/Documentation/DesignDocs/corewcfqueue-design.md) . Dmitry Maslov (@Ximik87) submitted a PR with an initial implementation of this base queue support along with an MSMQ implementation built on top of it. Jon Louie (@jonlouie) from Amazon AWS implemented the RabbitMQ transport on top of this. Biroj helped iterate the base queue implementation due to needing some changes to accommodate different API patterns of the RabbitMQ client library. There are two other queue transports coming soon. The first will be for Apache Kafka and is being contributed by Guillaume Delahaye (@g7ed6e). This will be part of and released by the CoreWCF project. The second will be for Azure Queue Storage which is being developed by Microsoft and will be released by them.  

To use any of the queue transports, you need to add the generic queue support to your application services.

```csharp
var builder = WebApplication.CreateBuilder();
builder.Services.AddServiceModelServices()
                .AddQueueTransport();
```

#### MSMQ

The CoreWCF project has a requirement to only depend on 3rd party packages which are backed by an open source foundation with a charter that ensures continued support of libraries if the maintainers step away from the project. A well known example in the .NET ecosystem is the [.NET Foundation](https://dotnetfoundation.org/). This is important to ensure that any potential security issues can be fixed in a timely manner. The current MSMQ implementation depends on a community owned release of a fork of the .NET Framework System.Messaging libraries which doesn't meet this requirement. The CoreWCF.MSMQ package will remain in a pre-release state until we have resolved this. We would encourage you to use the CoreWCF MSMQ transport as a stepping stone to moving to a more modern queue transport.  

To use the MSMQ transport, in addition to adding the queue transport support, you also need to add Msmq support.

```csharp
var builder = WebApplication.CreateBuilder();
builder.Services.AddServiceModelServices()
                .AddQueueTransport()
                .AddServiceModelMsmqSupport();

var app = builder.Build();

app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<Service>();
    serviceBuilder.AddServiceEndpoint<Service, IService>(new NetMsmqBinding(), "net.msmq://localhost/private/myqueue");
});
app.Run();
```

#### RabbitMQ
The RabbitMQ support has separate service and client packages, CoreWCF.RabbitMQ and CoreWCF.RabbitMQ.Client respectively. While the CoreWCF.RabbitMQ service package supports netstandard2.0, the client only CoreWCF.RabbitMQ.Client packge supports .NET 6.0 and later. This is due to requiring the latest (as of writing still in pre-release) WCF client packages which support .NET 6.0 or later. These packages will come out of preview once we've had enough feedback/usage to have confidence that we have the correct APIs for configuring and developers are able to use these packages successfully.  
Here's the code you need to use the service binding.

```csharp
var builder = WebApplication.CreateBuilder();
builder.Services.AddServiceModelServices()
                .AddQueueTransport()

var app = builder.Build();

app.UseServiceModel(serviceBuilder =>
{
    var uri = new Uri("net.amqps://localhost:5672/amq.direct/corewcf-classic-queue#corewcf-classic-key");
    var sslOption = new SslOption
    {
        ServerName = uri.Host,
        Enabled = true
    };
	// Replace with actual credentials for connecting to RabbitMQ host
	var credentials = new NetworkCredential(ConnectionFactory.DefaultUser, ConnectionFactory.DefaultPass);
    serviceBuilder.AddService<Service>();
    services.AddServiceEndpoint<Service, IService>(
        new RabbitMqBinding
        {
            SslOption = sslOption,
            Credentials = credentials,
            QueueConfiguration = new ClassicQueueConfiguration()
        },
        uri);
    });
});
app.Run();
```
For the queue configuration, we have two types, `ClassicQueueConfiguration` and `QuorumQueueConfiguration`. This configures whether the queue is either a classic mirrored queue, or the newer quorum queue. More information can [be found here](https://www.rabbitmq.com/quorum-queues.html). There are properties on these classes to configure various aspects of the queue such as whether it is durable.  
There is a mapping of the Uri passed to the endpoint and the queue options. The format of the Uri is `SCHEME://HOSTNAME:PORT/EXCHANGE/QUEUE_NAME#ROUTING_KEY`. The scheme can be one of two values, `net.amqp` or `net.amqps`, with the latter using a TLS secured connection to the AMQP server. The exchange and routing key are optional. If you aren't using an exchange and only have a queue name, ensure the Uri does NOT end in a slash. For example, the Uri `net.amqp://server/queuename` will connect to the queue called queuename and won't use an exchange. You can provide an optional routing key as a Uri fragment.  

For the code example above, we're connecting to an AMQP server running on port 5672, using TLS. The exchange is `amq.direct`, and the queue name is `corewcf-classic-queue`. It's using the routing key `corewcf-classic-key`.  

This is how you would use the client binding with the WCF Client.  

```csharp
var uri = new Uri("net.amqps://localhost:5672/amq.direct/corewcf-classic-queue#corewcf-classic-key");
var sslOption = new SslOption
{
    ServerName = uri.Host,
    Enabled = true
};
var endpointAddress = new System.ServiceModel.EndpointAddress(uri);
var rabbitMqBinding = new ServiceModel.Channels.RabbitMqBinding
{
    SslOption = sslOption
};
var factory = new ChannelFactory<IService>(rabbitMqBinding, endpointAddress);
factory.Credentials.UserName.UserName = ConnectionFactory.DefaultUser;
factory.Credentials.UserName.Password = ConnectionFactory.DefaultPass;
var channel = factory.CreateChannel();
((System.ServiceModel.Channels.IChannel)channel).Open();
await channel.CallServiceAsync();
```

### One last thing

There's one last minor feature that's shipping in 1.4.0 that many have been asking for. With this release you can now host a NetTcp service in the same WebHost that's serving HTTP requests with an IServer other than Kestrel. This means you can host a NetTcp service in IIS without having to run a second WebHost instance in the same process. This doesn't bring support for NetTcp activation or port sharing yet.

Please give this release a try and give any feedback in the discussion linked in [the release notes here](https://github.com/CoreWCF/CoreWCF/releases/tag/v1.4.0-preview1). 