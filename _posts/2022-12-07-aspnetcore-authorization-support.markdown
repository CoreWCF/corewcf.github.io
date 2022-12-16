---
# Layout
layout: post
title:  "Introducing ASP.NET Core Authorization support and modernization of legacy WCF Authentication and Authorization APis"
date:   2022-12-16 07:00:00 +0100
categories: release
# Author
author: Guillaume Delahaye (https://github.com/g7ed6e)
---
### Introduction
The latest release of CoreWCF will bring support of ASP.NET Core Authorization to allow developers to use ASP.NET Core builtin authentication middleware such as the `Microsoft.AspNetCore.Authentication.JwtBearer` and apply appropriate authorization policies.

### Builtin attributes support
When working with ASP.NET Core MVC usually developers use `[Authorize]` and `[AllowAnonymous]` to decorate actions that require specific authorizations.
#### Authorize support
To enable a seamless developer experience we brought the ability to decorate an `OperationContract` implementation with the ASP.NET Core Authorize attribute. However we introduced the below limitations to suggest developers to embrace the flexible [Policy-based](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-6.0) model based on `IAuthorizationRequirement`.
- `AuthenticationSchemes` property is not supported and will trigger a build warning `COREWCF_0201`.
- `Roles` property is not supported and will trigger a build warning `COREWCF_0202`.

#### AllowAnonymous support
We did not bring support of the `[AllowAnonymous]` attribute as we believe that a strong interface segregation between anonymous and secured operations should be set. Moreover supporting this attribute would imply delaying the authentication step in the pipeline leading to potential DoS vulnerabilities. Decorating an `OperationContract` implementation with `[AllowAnonymous]` will have no effect and will trigger a build warning `COREWCF_0200`.
### Configuration
To setup this feature in your CoreWCF application you should follow the below steps. I'm assuming that we want to enforce clients authenticating using a JWT Bearer token issued by an authorization server `https://authorization-server-uri`, the service should be protected by the audience `my-audience` and two policies should be defined, one requiring a scope `read` and another one requiring a scope `write`.
1. Register authentication infrastructure services and configure JWT Bearer authentication middleware as default `AuthenticationScheme`. (Internally CoreWCF is calling `HttpContext.AuthenticateAsync()` with the default registered authentication scheme).
```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
.AddJwtBearer(options => 
{
    options.Authority = "https://authorization-server-uri";
    options.Audience = "my-audience";
});
```
2. Register authorization infrastructure services and policies.
```csharp
services.AddAuthorization(options => 
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder(JwtBearerDefaults.AuthenticationScheme).RequireClaim("scope", "read").Build();
    options.AddPolicy("WritePolicy", new AuthorizationPolicyBuilder(JwtBearerDefaults.AuthenticationScheme).RequireClaim("scope", "write").Build());
})
```
3. Configure your service to use ASP.NET Core Authentication and Authorization middlewares setting the `ClientCredentialType` to `HttpClientCredentialType.InheritedFromHost`.
```csharp
app.UseServiceModel(builder =>
{
    builder.AddService<SecuredService>();
    builder.AddServiceEndpoint<SecuredService, ISecuredService>(new BasicHttpBinding
    {
        Security = new BasicHttpSecurity
        {
            Mode = BasicHttpSecurityMode.Transport,
            Transport = new HttpTransportSecurity
            {
                ClientCredentialType = HttpClientCredentialType.InheritedFromHost
            }
        }
    }, "/BasicWcfService/basichttp.svc");
}
```
4. Decorate your service implementation
```csharp
[ServiceContract]
public interface ISecuredService
{
    [OperationContract]
    string ReadOperation();
    [OperationContract]
    void WriteOperation(string value);
}

public class SecuredService : ISecuredService
{
    [Authorize]
    public string ReadOperation() => "Hello world";
    
    [Authorize(Policy = "WritePolicy")]
    public void WriteOperation(string value) { } 
}
```
### Supported bindings

ASP.NET Core Authorization policies support is implemented in http based bindings:
- `BasicHttpBinding`
- `WSHttpBinding`
- `WebHttpBinding`

### Authorization evaluation position in CoreWCF request pipeline

There's an important difference regarding the "when" authorization evaluation occurs between `ServiceAuthorizationManager` usage and the ASP.NET Core Authorization usage.

When using ASP.NET Core Authorization, ths below steps will be executed **before** authorization which didn't when using `ServiceAuthorizationManager`.

- When setup, dynamic quota throttle acquisition.
- Calls to registered `IDispatchMessageInspector.AfterReceiveRequest`
- Concurrency lock acquisition

Another impact is that authorization will now run on a captured `SynchronizationContext`. This point can impact CoreWCF services hosted in a UI thread (WPF or WinForms app).

### Exclusiveness of ASP.NET Core Authorization policies and `ServiceAuthorizationManager`

Having `ClientCredentialType` set to `InheritedFromHost` disables the execution of an authorization logic implemented in `ServiceAuthorizationManager`.

### `ServiceAuthenticationManager` and `ServiceAuthorizationManager` API modernization

These two classes now have async versions of the virtual methods which you can override. The existing synchronous method have been deprecated using the `Obsolete` attribute and will cause a build warning if you override them. If you are overriding one of the existing synchronous virtual methods, your code will continue to function the same as it always has and will continue to do so for all future 1.x releases. The synchronous variations of the methods will likely be removed in a future 2.x release. You can safely suppress the build warning until you have migrated your implementation to the async methods.

### Conclusion
CoreWCF provides flexibility around authentication and authorization allowing implementation of more up to date security standards and programming patterns well known from developers.