---
# Layout
layout: post
title:  "Add 'Keyed by Service type' ServiceBehavior support in CoreWCF"
date:   2024-07-05 13:00:00 +0200
categories: release
# Author
author: Guillaume Delahaye (https://github.com/g7ed6e)
---
### Introduction
The CoreWCF 1.6.0 release introduced a new feature that allows to apply a ServiceBehavior registered in DI to only one Service when hosting multiple services in a single host.

### Implementation
This feature uses the `IKeyedServiceProvider` capabilities introduced in .NET8 and requires the registration of the `ServiceBehavior` type to be done using the `IServiceCollection.AddKeyedSingleton<TService, TImplementation>(object? serviceKey)` extension method.

### Example

In the below `Startup` class, the call to `AddKeyedSingleton<IServiceBehavior, MyServiceBehavior>(typeof(ReverseEchoService))` apply `MyServiceBehavior` to `ReverseEchoService` only.
```c#
private class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton<EchoService>();
        services.AddSingleton<ReverseEchoService>();
        services.AddKeyedSingleton<IServiceBehavior, MyServiceBehavior>(typeof(ReverseEchoService));
        services.AddServiceModelServices();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseServiceModel(builder =>
        {
            builder.AddService<EchoService>();
            builder.AddServiceEndpoint<EchoService, IEchoService>(new BasicHttpBinding(), "/EchoService.svc");
            builder.AddService<ReverseEchoService>();
            builder.AddServiceEndpoint<ReverseEchoService, IReverseEchoService>(new BasicHttpBinding(), "/ReverseEchoService.svc");
        });
    }
}
```
