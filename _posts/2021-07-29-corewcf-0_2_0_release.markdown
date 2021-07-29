---
# Layout
layout: post
title:  "CoreWCF 0.2.0 Release"
date:   2021-07-29 13:00:00 -0800
categories: release
# Author
author: Matt Connew
---
Today we're releasing our second release of CoreWCF, version 0.2.0. We have a few new features in this release, so I'm going to give a brief summary of what they are and how to use them.

#### Enable role based authorization via new AuthorizeRoleAttribute.

In WCF you were able to restrict the ability for a user to be able to call certain methods based on their Active Directory roles. You would do this by applying the PrincipalPermissionAttribute to your service implementation method and specify the Role required to execute the method. This depended on a .NET CLR feature where WCF would set the current thread principal to the authenticated WindowsIdentity and the CLR would do the enforcement for us using Code Access Security (CAS). Your code would look similar to this:

{% highlight csharp %}
    [PrincipalPermission(SecurityAction.Demand, Role=@"BUILTIN\Administrators")]
    [PrincipalPermission(SecurityAction.Demand, Role=@"BUILTIN\Managers")]
    public string APIFoo()
    {
        // Your API logic
    }
{% endhighlight %}
		
This CLR feature isn't available in .NET Core/.NET 5+, so we came up with another solution. We looked at how ASP.NET Core implements [Role based authentication](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-5.0) and designed something similar. We created a new attribute AuthorizeRoleAttribute which takes a comma delimited list of Active Directory roles. The equivalent attribute to the example above would look like this:

{% highlight csharp %}
    [AuthorizeRole("BUILTIN\Administrators, BUILTIN\Managers")]
    public string APIFoo()
    {
        // Your API logic
    }
{% endhighlight %}

On Windows, that's all you need to do. Unfortunately on Linux the Windows authentication component is unable to extract the list of roles from the client authentication token, so we need another mechanism to retrieve the list of roles. To do this we use LDAP to connect to the domain controller and make a query for the list of roles that the user has. You initialize the LdapSettings like this:

{% highlight csharp %}
LdapSettings ldapSettings = new LdapSettings("ldap_server.domain.local", "domain.local", "orgunit.domain.local");
ServiceCredential.WindowsAuthentication.LdapSetting = ldapSettings;
{% endhighlight %}

This should give you the same behavior as you had previously with WCF. This feature was built on top of another new feature. The AuthorizeRole attribute extends the public interface IAuthorizeOperation. That interface looks like this:

{% highlight csharp %}
    public interface IAuthorizeOperation
    {
        void BuildClaim(OperationDescription operationDescription, DispatchOperation dispatchOperation);
    }
{% endhighlight %}

This allows you to create your own attributes to restrict calling an operation based on any claims based criteria. In the BuildClaim method you populate the ConcurrentDictionary<string, List<Claim>> property DispatchOperation.AuthorizeClaims with a named list of required claims. For example, you could add a certificate issuer claim requiring a specific issuer for a client certificate in order to call the operation. After the AuthorizeClaims ConcurrentDictionary has been populated, CoreWCF will do the rest for you.

#### Extending support of security mode TransportWithMessageCredential for NetTCPBinding and BasicHttpBinding

We have added support for TransportWithMessageCredentiuals to NetTCPBinding and BasicHttpBinding. The supported client credential types are UserName, Certificate and Windows authentication. 

#### Support for .NET 5.0 (and .NET Core 3.1)

To run on .NET 5.0 in your project you need to specify the TargetFramework as net5.0, and you need to set the Project Sdk to "Microsoft.NET.Sdk.Web". You also need to remove any package reference to "Microsoft.AspNetCore".

#### New overloads to UseNetTcp

You can now specify the listening IPAddress when using the UseNetTcp extension method to add support. This change does bring with it a small change in behavior. The address populated in the IServerAddressesFeature feature used to have the hostname portion populated with localhost. This now returns the IPAddress passed to UseNetTcp.

#### MessageParameterAttribute public and added compatibility for System.ServiceModel equivalent

We now support using the MessageParameterAttribute in your service contract. If you are still sharing a contract definition with .NET Framework or a client library and are referencing the System.ServiceModel.Primitives package, we will support the use of the System.ServiceModel version of this attribute. If you don't need to bring in the client packages, we have the CoreWCF namespace version.

#### Enabled injecting ServiceBehaviorAttribute via DI

You can now inject an instance of ServiceBehaviorAttribute via DI in your Startup ConfigureServices method. This allows configuring CoreWCF to use the InstanceContextMode of Single early in the startup and skips creating a dummy instance which then needs to be optionally Dispose'd. This was causing a problem when DI is providing a Singleton and the type implements IDisposable as we would call Dispose on the Singleton instance. There's also no longer a need to have a dummy public constructor to satisfy CoreWCF if the type has been injected via DI.