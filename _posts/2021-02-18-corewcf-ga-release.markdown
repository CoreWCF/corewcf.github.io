---
# Layout
layout: post
title:  "CoreWCF 0.1.0 GA Release"
date:   2021-02-18 17:00:00 -0800
categories: release
# Author
author: Matt Connew
---
After 21 months of public development, CoreWCF has finally reached its first GA release. It's been a long path to get here, and thee is still have a long way to go. CoreWCF started off as a prototype internal to Microsoft which was started after a meeting with Scott Hunter. During that meeting Scott said something which has shaped the entire development of CoreWCF. He said that we only want one service host (the concept, not the WCF ServiceHost class) in .NET Core. ASP.NET Core had already been released, which meant there wasn't a place for ServiceHost (the WCF class). This made a lot of sense, so I set about building a WCF prototype on top of ASP.NET Core. Eventually a decision was made to release the prototype as a community project under the name CoreWCF. The prototype focused on the developer experience so a lot of the implementation needed some love and attention, and 21 months later, we're releasing our first GA release. Along the way, the CoreWCF project caught the attention of the Amazon AWS team and they have been contributing significantly to help make this project a success. It is thanks to the contributions from Amazon AWS that the initial release supports WS-* protocols (WS-Security, WS-SecureConversation), and some token authentication credential types with the TransportWithMessageCredentials security mode. This is a significant step towards supporting WS-Federation to enable moving enterprise WCF services to CoreWCF hosted on a cloud platform.  


So what changes happened that took 21 months you might ask? Some fundamental architecture changes to make this project sustainable and cross platform. There have been 2 major themes to the changes. Removing Asynchronous Programming Model (APM) api’s and code, and removing direct native system calls and IO code. 


The APM programming pattern is incredibly fast and squeezes out every last bit of performance, but at the cost of code maintainability and debuggability. WCF uses APM to its limit and the codebase can be hard to work with. Porting the code over as-is would require a large commitment from anyone wanting to contribute anything non-trivial. This also would make it hard for the community to be self-supporting. CoreWCF now uses async/await Task based asynchronous programming throughout. Computers are so much faster today than 10 years ago. For a community owned and supported project maintainability is a high priority than speed, within reason. CoreWCF also switched to a request push pipeline model adopting the ASP.NET Core middleware pattern. This made the code simpler, but required a lot of refactoring and rewriting of pieces of WCF.


The second big change is removing platform specific code and removing IO code. CoreWCF doesn't even know what a Socket is and yet supports NetTcp. ASP.NET Core handles all of that for CoreWCF. It just reads and writes to pipes or streams. This means CoreWCF developer time isn't spent having to handle writing code for some obscure scenario on a specific platform, it can (mostly) let the ASP.NET Core team worry about that.


All this is to say that supportability and maintainability is a high priority for CoreWCF.


A lot has changed, but a lot is also the same. Where changes have been needed, a concerted effort has been made to make things familiar. To give an example, let's look at some code. When creating a self hosted service (console or WPF/WinForms as opposed to IIS hosted) in WCF, you would create a ServiceHost, add a ServiceEndpoint, and open the service:

{% highlight csharp %}
ServiceHost host = new ServiceHost(typeof(MyService), new Uri(http://localhost/myservice));
host.AddServiceEndpoint(typeof(IMyService), new BasicHttpBinding(), "/basic");
host.Open();
{% endhighlight %}


With CoreWCF, there's a little more code required due to the configuration model of ASP.NET Core, but the CoreWCF pieces are minimal. Those familiar with ASP.NET Core know that when configuring your host you need to specify a startup class which contains two methods, ConfigureServices and Configure. The ConfigureServices method is where anything needed in the dependency injection (DI) system is added, and any extension methods are called to add support for various frameworks such as MVC. At it's simplest, a single method is called to add WCF support.

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddServiceModelServices();
}
{% endhighlight %}
                

In the Configure method, things will look a little more familiar. The extension method UseServiceModel(this IApplicationBuilder app) is called with a delegate which configures CoreWCF.

{% highlight csharp %}
public void Configure(IApplicationBuilder app)
{
    app.UseServiceModel(builder =>
    {
        builder.AddService<MyService>();
        builder.AddServiceEndpoint<MyService, IMyService>(new BasicHttpBinding(), "/basic");
    });
}
{% endhighlight %}


The AddService<TService>() method registers that a service for the type TService will be used. The service then needs at least one endpoint added to the builder for that service type. When CoreWCF needs an instance of TService, it will first look in DI to see if there's an implementation available, and if not found, will fall back to creating an instance using reflection the same way that WCF does. If either InstanceContextMode.PerSession or InstanceContextMode.PerCall is used, CoreWCF will request an instance from DI if available each time an instance is needed. It will fall back to the same behavior as WCF if the type is not registered in DI. This allows injecting other dependencies such as loggers into the service instances without having to implement IInstanceContextProvider to do so.


CoreWCF still supports the System.ServiceModel namespaced contract attributes such as ServiceContract and OperationContract if the WCF Core client nuget package System.ServiceModel.Primitives is referenced. This allows sharing contract assemblies between the client and the server. There is also CoreWCF namespaced versions of these attributes too so a dependency on the WCF Core client nuget package isn’t needed.


There are multiple resources to find out more and to ask any questions about CoreWCF:
* Sample usages of CoreWCF alongside their WCF conterparts are available [here](https://github.com/CoreWCF/CoreWCF/tree/main/src/Samples). 
* Discussions are enabled on the CoreWCF repo [here](https://github.com/CoreWCF/CoreWCF/discussions).
* Release roadmap can be found [here](https://github.com/CoreWCF/CoreWCF/blob/main/Documentation/roadmap.md)
* Feature prioritizing issue where you can vote for what you need is [here]( https://github.com/CoreWCF/CoreWCF/issues/234 )
* Monthly community sync-up meeting details can be found [here](https://github.com/CoreWCF/CoreWCF/wiki/Community-Sync-up-meetings)
* CoreWCF gitter channel can be found [here](https://gitter.im/CoreWCF/CoreWCF)