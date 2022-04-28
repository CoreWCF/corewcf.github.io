---
# Layout
layout: post
title:  "CoreWCF 1.0.0 Release"
date:   2022-04-28 08:00:00 -0800
categories: release
# Author
author: Matt Connew
---
Today we've hit a big milestone and have released version 1.0.0 of CoreWCF. This is the end of the beginning of a long journey for me that started just over 5 years ago back in January of 2017. I was given 3 weeks to put together a basic prototype of what a WCF service implementation would look like built on .NET Core. At the end of the 3 weeks, I had a working implementation which could host a service using BasicHttpBinding. My prototype then sat there gathering dust as a proof of concept while it was decided what to do with it. There weren't the resources to develop this into a full product with feature parity to WCF, but there were many customers blocked unable to move to .NET Core without doing a complete rewrite of their service. It was eventually decided that I would spend some time cleaning up my prototype implementation, including adding NetTcp support, and the code would be donated to the open source community to see if this was something that a community would build around for it to live outside of Microsoft.  

I have a personal passion for WCF as it solves many difficult problems in interesting and often complicated ways and I enjoy solving interesting and complicated problems. I was asked if I wanted to personally own the project. I was hesitent at first as I was worried that I would be personally committing to porting most of the code base on my own. Shortly after the project began development in the open I was contacted by Biroj Nayak from Amazon AWS asking how they could help contribute to Core WCF. They had their own customers asking what could be done to enable porting their WCF services to the cloud. This started a multi year collaboration with Amazon where they ported some very large and significant functionality from WCF to Core WCF. Rebuilding the channel layer on top of ASP.NET Core requires a significant refactoring of much of the code base and some features involved a lot of code that needed to be committed in one large piece. Biroj took on the multi-month task of porting some of the larger missing features to CoreWCF.

After a while, we started getting some smaller contributions from the community. Adding support for narrow scenarios which hadn't been included, or fixing an edge case which the new code didn't handle. As time has gone on, the size and number of community contributions has gradually and continuously increased. We've seen more companies contribute developer resources to porting significant features. My worry about being the only person working on porting WCF to .NET Core has been completely dispelled. We recently hit a milestone that I have contributed less than half the commits to the Core WCF repo. I now spend a large part of the time I have available for Core WCF reviewing others code and taking more of an architect role to enable others to contribute. We'd like to express a big thank you to all those who have contributed to this project to make it a success.

#### What does the v1 label mean?
Besides naming your variables, one of the toughest questions in software development is when is it ready for release? If we waited for feature parity with WCF, we might never get to v1 as some features have missing dependencies. I decided that I would be willing to apply the v1 label when Core WCF is "useful" for use in production by a large number of WCF customers. Being useful is a very vague and blurry bar so I had to decide what that meant. What I came up with is being able to use SOAP with the HTTP transports, having a sessionful transport, and being able to generate the WSDL for a service. I had already implemented NetTcp on top of the connection handler feature of ASP.NET Core so supporting a sessionful transport was covered. The major thing left to implement was WSDL support. Along the way the community decided to contribute support for TransportWithMessageCredentials, WS-Federation, Configuration, WebHttpBinding for RESTful services, and many other smaller features including some which don't even exist on WCF. With the recent completion of WSDL generation, we're now at a point where I believe Core WCF should be useful to many developers using WCF.

There are still some notable features missing. For example, we don't have tracing support yet, and you need to configure HTTP authentication in ASP.NET Core and not via the binding. If this is your first time looking at using CoreWCF, I recommend reading the prior blog posts as they contain many answers on how to port your service to Core WCF.

#### The feature I need is missing, what do I do?
Missing features fall into two categories.
* The implementation is there but isn't public
* The implementation is completely absent

When the implementation is there but not public, it's because we don't have tests for it yet. Making an API public without having tested that there are no problems with any changes made in the port will lead to a lot of noise and bad experiences. If you discover there's an extensibility point that you need which is internal, the fastest way to get it supported is to submit a PR making it public along with some tests verifying that the extensibility point is working as expected.  

If the feature you need is completely absent, you have two options. The first option is to check to see if it's on the feature roadmap issue [here](https://github.com/CoreWCF/CoreWCF/issues/234) and if it isn't, add it. Then upvote the feature following the instructions at the top of the issue. We strongly weight demand when deciding on which feature to work on next. The second option is to contribute developer resources to porting the feature. The WebHttp feature is an example of this happening. Porting WebHttpBinding was too far down the priority list for one company which needed it so with some guidance, they ported the feature.  

Another alternative might be to modify your service to use a different feature which provides the same capabilities. For example, switching to NetTcpBinding if you are currently using NetNamedPipeBinding.

#### What's new since 0.4.0
The following new features have been added since Core WCF 0.4.0 was released:  
* WebHttpBinding support with OpenApi feature - Jonathan Hope, Digimarc (@JonathanHopeDMRC)
* WS-Federation support - Biroj Nayak, Amazon AWS (@birojnayak)
* WSDL support, including ServiceDebugBehavior - Matt Connew, Microsoft (@mconnew)
* New support for injecting HttpContext, HttpRequest, and HttpResponse objects into service implementation methods. This also includes supporting the ASP.NET Core FromServicesAttribute in addition to the CoreWCF InjectedAttribute - Guillaume Delahaye (@g7ed6e)
* Custom Binding support for Configuration - (@kbrowdev)

There are 3 new blog posts talking about some of these new features:  
-[WebHttpBinding support](https://corewcf.github.io/blog/2022/04/13/webhttp)  
-[WSDL support](https://corewcf.github.io/blog/2022/04/26/wsdl)  
-[WS-Federation support](https://corewcf.github.io/blog/2022/04/27/wsfed)  

#### Official Microsoft support
With the v1.0.0 release of Core WCF, Microsoft are providing support. The current support lifecycle can be found at [http://aka.ms/corewcf/support](http://aka.ms/corewcf/support). Microsoft has published a blog post explaining the support policy for Core WCF.
	