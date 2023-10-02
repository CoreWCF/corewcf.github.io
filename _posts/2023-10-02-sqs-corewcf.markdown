---
# Layout
layout: post
title:  "Introducing AWS SQS binding extension support in CoreWCF and WCF Client"
date:   2023-10-02 13:00:00 -0800
categories: release
# Author
author: Abhi Gujjewar,Biroj Nayak (https://github.com/birojnayak) , PJ Pittle (https://github.com/ppittle)
---
# Introduction
AWS has published two NuGet packages, AWS.CoreWCF.Extensions (server side) and AWS.WCF.Extensions (client side), to enable WCF clients and CoreWCF services to communicate via AWS SQS Queue Transport. This is primarily to enable users to migrate services using MSMQ binding from on-premises to cloud. With AWS SQS transport binding, customers can send SOAP messages to AWS SQS via the WCF client and run CoreWCF services to receive and process those messages without changing any contract or service implementations.

Along with moving SOAP services to the AWS cloud, this transport provides the extensibility to attach callback methods that can trigger SNS notifications, AWS lambda invocations, etc. once a message is processed. By using this transport, customers can get all AWS SQS metrics out of the box. SQS metrics can also be used to cost-effectively scale your services utilizing the EC2 Autoscaling functionality. [How to use SQS for Autoscaling?](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-using-sqs-queue.html)

# How this works

### WCF Client
On the WCF client side you add the AWS.WCF.Extensions package from Nuget. Once package added you can initialize the client as shown below

```csharp
    var queueName = "your-aws-sqs-queue";
    var sqsClient = new AmazonSQSClient("Your AWS AccessKey", "your aws secret key");
    var sqsBinding = new AWS.WCF.Extensions.SQS.AwsSqsBinding(sqsClient, queueName);
    var endpointAddress = new EndpointAddress(new Uri(sqsBinding.QueueUrl));
    var factory = new ChannelFactory<YourFooService>(sqsBinding, endpointAddress);
    var channel = factory.CreateChannel();
    ((System.ServiceModel.Channels.IChannel)channel).Open();
    channel.InvokeFoo("Hello there");
```
To transmit messages, you must first identify the SQS queue and the credentials you will need. [Setting up SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-setting-up.html)

```csharp
var sqsClient = new AmazonSQSClient("Your AWS AccessKey", "your aws secret key");
```
AmazonSQSClient can be initialized by providing your AWS Credentials directly. If you prefer using configuration based approach, here is the documentation. [using config based](https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/net-dg-config-netcore.html)


### CoreWCF Service
On the CoreWCF server side, you need to add the AWS.CoreWCF.Extensions package from NuGet [TODO - ADD NUGET LINK]. Then you can initialize the service as shown below.

```csharp
public class Program
{
    static void Main(String[] args)
    {
        var host = WebHost.CreateDefaultBuilder(Array.Empty<string>()).UseStartup<Startup>().Build();
        host.Run();
    }

    public class Startup
    {
        private static readonly string _queueName = "your-aws-sqs-queue";
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<LoggingService>();
            services.AddServiceModelServices();
            services.AddQueueTransport();

            //Additional info needed for AWS
            AWSOptions option = new AWSOptions();
            option.Credentials = new BasicAWSCredentials("your access key", "your secret key");
            services.AddDefaultAWSOptions(option);
            services.AddSQSClient(_queueName);
            //end of it
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            var queueUrl = app.EnsureSqsQueue(_queueName);
            app.UseServiceModel(services =>
            {
                services.AddService<LoggingService>();
                services.AddServiceEndpoint<LoggingService, ILoggingService>(
                   new AwsSqsBinding(),
                    queueUrl
                );
            });
        }
    }
```

There are a few things to note here. In the ConfigureServices method, you are passing AWS credentials for the SQS queue via the `AddDefaultAWSOptions` method and calling the `AddSQSClient` extension method to initialize the SQS client.
In the `Configure` method, `EnsureSqsQueue` is called which ensures that the queue exists. If the queue doesn't already exist, it will be created and returns the url for the queue. If the queue already exists, it returns the url for the existing queue.
Optionally, you can pass several parameters to create the queue if needed via `CreateQueueRequestBuilder`.

We've added a property named 'DispatchCallbacksCollection' to the 'AwsSqsBinding' class. This property is of type 'IDispatchCallbacksCollection' which is defined as follows.

```csharp
 public interface IDispatchCallbacksCollection
 {
     NotificationDelegate NotificationDelegateForSuccessfulDispatch { get; set; }

     NotificationDelegate NotificationDelegateForFailedDispatch { get; set; }
 }
 public delegate Task NotificationDelegate(IServiceProvider services, QueueMessageContext context);
```
You can provide an implementations of `IDispatchCallbacksCollection` to customize the service behavior when a message has completed being processed. If a message is successfully dispatched with no exceptions thrown, the delegate `NotificationDelegateForSuccessfulDispatch` will be called. If there is a problem while dispatching a message, either in deserializing the message and selecting the operation, or the service implementation throws, the delegate `NotificationDelegateForFailedDispatch` will be called. Some examples of what an implementation could do are notifying other consumers, triggering AWS Lambda functions, notifying SNS subscribers, etc.

By default, AwsSqsBinding uses the a default concurrency of 1, meaning the client will fetch 10 messages in a batch and one thread will process them one at a time. If you increase the concurrency level to more than 1, your messages will be pulled from SQS in batches by a single thread, and processed concurrently on multiple threads.


There is a sample available [here][link to repo]. Please try and provide feedback [here] [link to our repo].

#### Thanks

Thanks to AWS .NET open source team (https://aws.amazon.com/blogs/opensource/category/programing-language/dot-net/) for coming up with this project.
