---
# Layout
layout: post
title:  "CoreWCF 0.4.0 Release"
date:   2021-12-14 17:00:00 -0800
categories: release
# Author
author: Matt Connew
---
Today we're releasing our fourth release of CoreWCF, version 0.4.0. As with the previous release, the majority of new features were from community contributors. You can [read here the full release notes](https://github.com/CoreWCF/CoreWCF/releases/tag/v0.4.0) for all the new features. In this blog post I'm going to be talking about three of the new features which need a little bit of explanation, all of which were contributed by Guillaume Delahaye ([@g7ed6e](https://github.com/g7ed6e)).

#### Async support for SecurityTokenAuthenticator

In .NET Framework, the abstract class SecurityTokenAuthenticator looks like this:
```csharp
public abstract class SecurityTokenAuthenticator
{
    protected SecurityTokenAuthenticator() { }
    public bool CanValidateToken(SecurityToken token) { /* ... */ };
    public ReadOnlyCollection<IAuthorizationPolicy> ValidateToken(SecurityToken token) { /* ... */ };
    protected abstract bool CanValidateTokenCore(SecurityToken token);
    protected abstract ReadOnlyCollection<IAuthorizationPolicy> ValidateTokenCore(SecurityToken token);
}
```
When you wand to implement a custom SecurityTokenAuthenticator, you derive from this class and implement `CanValidateTokenCore` and `ValidateTokenCore`. One of the pain points I've seen WCF customers have over the years is that token authentication is synchronous. Sometimes you need to call out to another system, for example validating against a SQL server. Doing this in a synchronous method is less than ideal as you end up blocking a thread which can result in scalability limitations. We've changed this class in CoreWCF to look like this:
```csharp
public abstract class SecurityTokenAuthenticator
{
    protected SecurityTokenAuthenticator() { }
    public bool CanValidateToken(SecurityToken token) { /* ... */ };
    public ReadOnlyCollection<IAuthorizationPolicy> ValidateToken(SecurityToken token) { /* ... */ };
    public ValueTask<ReadOnlyCollection<IAuthorizationPolicy>> ValidateTokenAsync(SecurityToken token) { /* ... */ };
    protected abstract bool CanValidateTokenCore(SecurityToken token);
    [Obsolete("Implementers should override ValidateTokenCoreAsync.")]
    protected virtual ReadOnlyCollection<IAuthorizationPolicy> ValidateTokenCore(SecurityToken token)  { /* ... */ };
    protected virtual ValueTask<ReadOnlyCollection<IAuthorizationPolicy>> ValidateTokenCoreAsync(SecurityToken token) { /* ... */ };
}
```
The result of this change is that you can now implement your authenticator asynchronously, but you can leave an existing implementation as-is if you prefer. The only possible change you might make is to suppress a compiler warning caused by overriding a now obsolete method. This is safe to do as the Obsolete attribute is there to raise awareness. Everything will continue to function the same as before. To make your `ValidateTokenCore` implementation async, change the method signature to match the new `ValidateTokenCoreAsync` method and modify your code to take advantage of `async`/`await`. Everything should behave the same as it did before.

In a future release the `ValidateTokenCore` method may be removed. There are currently no plans to do so, and this wouldn't happen before we get to a 2.0.0 release.

#### Support for scoped IServiceProvider

We've added the capability to have an `IServiceProvider` scope created automatically. This feature gets turned on when you use DI to provide your service instance. This decision was made to avoid adding the additional cost of creating a DI scope unless you are actively using DI for your service implementation.  

The lifetime of the created scope is tied to the `InstanceContext` lifetime and can be controlled by specifying the `InstanceContextMode` of your service via the `ServiceBehaviorAttribue`. If you want a new scope for every call, use `InstanceContextMode.PerCall`, if you want one scope per session, use `InstanceContextMode.PerSession`. For example, when using `PerSession` with NetTcp a new `IServiceProvider` scope will be created for each connected client and will persist the same scope across multiple calls.  

The scoped service provider is exposed via an `InstanceContext` extension. You would access it like this:
```csharp
    var provider = OperationContext.Current.InstanceContext.Extensions.Find<IServiceProvider>();
    var logger = provider.GetRequiredService<ILogger<MyService>>();
```
When the InstanceContext is no longer needed, the underlying `IServiceScope` will be disposed and all disposable instantiated instances in the scope will then be disposed too. If you use a custom `IInstanceProvider` implementation, for example to add logging, I suggest wrapping the CoreWCF provided implementation and using it's implementation of `GetInstance` and `ReleaseInstance` for the service instance instantiation.

#### DI injected operation parameters

This is the new feature that I'm most excited about. This feature is similar to the ASP.NET MVC attribute `FromServicesAttribute`. We were discussing how to implement DI injection into operation methods with various ideas on how to achieve this. Very quickly there was strong pushback against touching the interface contract, which meant the interface methods couldn't know about extra injected parameters. This causes a problem, how do you implement an interface when the implemented methods have extra parameters which aren't in the interface and keep the C# compiler happy? The solution was a source generator which back fills the now missing interface methods. Let's start with an example. Here's my simple IEchoService interface and the service implementation.
```csharp
[ServiceContract]
public interface IEchoService
{
    [OperationContract]
    string Echo(string message);
}

public class EchoService : IEchoService
{
    public string Echo(string message)
    {
        return message;
    }
}
```
My service is working great for telling callers what they said, but now I want to add logging to my method, so I need an ILogger. To enable this feature, there are two prerequisites. First, make sure that EchoService is provided by the DI system. This will cause a service provider scope to be created for us so that an `ILogger` can be fetched from DI. The second thing we need to do is make our `EchoService` class `partial`. This is so that the source generator can backfill an Echo method implementation which matches the interface as our Echo method won't match any more. Once these two things have been done, we can add extra parameters to our method and attribute them with `CoreWCF.InjectedAttribute`. To add an ILogger to our service we change our service implementation to this:
```csharp
public partial class EchoService : IEchoService
{
    public string Echo(string message, [Injected]ILogger<EchoService> logger)
    {
        logger.Log($"Received echo message {message}");
        return message;
    }
}
```
Now we don't have an `Echo` method which implements the interface method. When you next build your project, the source generator will create the following code which implements the `Echo` method to match the `IEchoService` interface. It does the work of fetching the `ILogger` from the service scope and passing it to the new `Echo` method. 
```csharp
public partial class EchoService
{
    public string Echo(string message)
    {
        IServiceProvider provider = OperationContext.Current.InstanceContext.Extensions.Find<IServiceProvider>();
        if (provider == null) throw new InvalidOperationException("Missing IServiceProvider in InstanceContext extensions");
        if (OperationContext.Current.InstanceContext.IsSingleton)
        {
            using (IServiceScope scope = provider.CreateScope())
            {
                ILogger<EchoService> service = scope.ServiceProvider.GetService<ILogger<EchoService>>();
                return this.Echo(message, service);
            }
        }
        else
        {
            ILogger<EchoService> service = provider.GetService<ILogger<EchoService>>();
            return this.Echo(message, service);
        }
    }
}
```
If you are using `InstanceContextMode.Single`, a new service scope is created for the call, otherwise the scope that was already created is used. This is to prevent memory leaks from transient services being created and not being cleaned up. Then your modified method is called with the parameters from the contract and the DI injected parameters.

These are some great new features to move CoreWCF forward and enabling develoeprs to embrace more modern programming models. If there's an area of WCF which you feel could benefit from some modernization and would like to roll up your sleeves and implement it, please [open an issue](https://github.com/CoreWCF/CoreWCF/issues) in our GitHub repo and we can discuss it. As long as it doesn't break existing services, fits modern programming patterns, and will be generally useful, we're open to new ideas. CoreWCF isn't just reimplementing WCF, it's bringing WCF to modern services and applications.