---
# Layout
layout: post
title:  "CoreWCF 0.3.0 Release"
date:   2021-09-13 13:00:00 -0800
categories: release
# Author
author: Matt Connew
---
Today we're releasing our third release of CoreWCF, version 0.3.0. We have a few new features in this release, the majority of which were from community contributors. I'd like to publicly express appreciation for work the community has put into CoreWCF with this release. Without them, we couldn't release as many additional features so soon. Here's a brief summary of the new features and how to use them.
		
#### Partial support for WCF app.config configuration

Taylor Southwick is a developer at Microsoft and [created a library](https://github.com/twsouthwick/ServiceModel.Configuration) to support using your .NET Framework app.config for the WCF client packages on .NET Core. Dmitry Maslov (@Ximik87) volunteered to take this code and develop it further for use with CoreWCF. The outcome from that is the package CoreWCF.Configuration. Currently not all options are supported, but this should help when migrating WCF applications. Here are the basic steps to use the new library.  

Copy the system.serviceModel section into a separate file, for example wcf.config, which should exist in the application folder. Here is an example file:
{% highlight xml %}
 <system.serviceModel>
    <bindings>
      <netTcpBinding>
        <binding name="netTcpBindingConfig" receiveTimeout="00:10:00" />       
      </netTcpBinding>
    </bindings>
    <services>
      <service name="Services.ISomeContact" >
        <endpoint address="net.tcp://localhost:8750/Service" binding="netTcpBinding" bindingConfiguration="netTcpBindingConfig" contract="ISomeContact"  />       
      </service>
    </services>
  </system.serviceModel>
{% endhighlight %}

Next, create an ASP.NET Core application and add the CoreWCF.Primitives, CoreWCF.NetTcp, and CoreWCF.ConfigurationManager nuget packages. Configure your services and applications in your Startup class:

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{           
     services.AddServiceModelServices();
     services.AddServiceModelConfigurationManagerFile("wcf.config");
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
      app.UseServiceModel();
}
{% endhighlight %}

Note that if NetTcpBinding is used, then the ports of endpoints using this binding must be additionally specified when configuring WebHostBuilder:

{% highlight csharp %}
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
      WebHost.CreateDefaultBuilder(args)
  	  .UseNetTcp(8750)
          .UseStartup<Startup>();
{% endhighlight %}

That's all there should be too it! If there are any configuration problems in the wcf.config file, this will be visible when the application starts.

The set of supported configuration elemetns are:  
BindingSection:  
BasicHttpBinding, NetHttpBinding, NetTcpBinding, WSHttpBinding  

For all supported bindings - SecurityMode, MaxReceivedMessageSize, ReceiveTimeout, CloseTimeout, OpenTimeout, SendTimeout, and XmlReaderQuotas. Additionally we support the following specific settings for these bindings:  
BasicHttpBinding - MaxBufferSize,TransferMode,TextEncoding
NetHttpBinding - MaxBufferSize,TransferMode,TextEncoding,MessageEncoding
NetTcpBinding - MaxBufferSize,MaxBufferPoolSize,MaxConnections, TransferMode,HostNameComparisonMode
WSHttpBinding  - MaxBufferPoolSize

ServiceSection:  
Endpoint - Address,Binding,BindingConfiguration,Contract

We will add more configuration support over time.

#### Added non-generic overload of ServiceBuilderExtensions.ConfigureServiceHostBase()

Sometimes you need to do additional configuration to the ServiceHostBase instance that is created internally. With CoreWCF, you don't create this instance yourself, it's created as part of the serivce startup. To allow further customization before the ServiceHostBase is opened, we have an extension method to allow providing a configuration delegate to modify the instance.

{% highlight csharp %}
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseServiceModel(builder =>
    {
        builder.AddService<EchoService>();
        builder.AddServiceEndpoint<EchoService, IEchoService>(binding, "basichttp.svc");
        builder.ConfigureServiceHostBase<EchoService>((serviceHostBase) => { /* Modify serviceHostBase */ });
    });
}
{% endhighlight %} 

Sometimes you don't have the type available as a generic parameter and only captured in a Type variable. In this release, Guillaume Delahaye (@g7ed6e) added a non-generic overload of this method which takes the type as a parameter.

{% highlight csharp %}
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    Type serviceType = GetServiceType();
	Type contractType = GetImplementedContractType();
    app.UseServiceModel(builder =>
    {
        builder.AddService(serviceType);
        builder.AddServiceEndpoint(serviceType, contractType, binding, "basichttp.svc", null);
        builder.ConfigureServiceHostBase(serviceType, (serviceHostBase) => { /* Modify serviceHostBase */ });
    });
}
{% endhighlight %} 

#### Made HttpsTransportBindingElement public to enable custom bindings using HTTPS

As of the 0.2.0 release, CoreWCF differentiated HTTP and HTTPS addresses. If you are using a CustomBinding, you will need to use HttpsTransportBindingElement if your service is using HTTPS. This has now been made public in 0.3.0 thanks to Biroj Nayak (@birojnayak).

#### Added compat support for System.ServiceModel.XmlSerializerFormatAttribute

If you have an existing WCF service contract which you wish to share between CoreWCF and either .NET Framework or .NET Core WCF client, double attributing your contract with both CoreWCF and System.ServiceModel namespaced versions of attributes is messy and error prone. It also requires you to add a reference to CoreWCF in places where it's not needed. To make this scenario easier, CoreWCF already has compat support for many System.ServiceModel attributes. This means CoreWCF understands the System.ServiceModel namespaced versions of the attributes. One ommission from this compat feature was support for System.ServiceModel.XmlSerializerFormatAttribute. Guillaume Delahaye (@g7ed6e) added compatibility support for this attribute.

#### Duplex contract support

The changes required to support duplex contracts was contributed by Daniel Kuschny (@Danielku15) which enables a lot of additional scenarios.

#### FaultContractInfo available for use

Many API's in CoreWCF are implemented but are still internal. This is because we have a policy that unless there are tests for an API, we consider it as not working. In many cases the API is already fully functional and just needs to be made public, but we're lacking tests. Guillaume Delahaye (@g7ed6e) had a need for FaultContractInfo so contributed tests to validate the API is working as expected and changed the API from internal to public. Taking this approach also ensures the specific scenario that is needed by a contributer is tested.
