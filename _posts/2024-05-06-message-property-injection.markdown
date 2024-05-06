---
# Layout
layout: post
title:  "Add IncomingMessageProperty injection support anf Kafka transport enhancements in CoreWCF"
date:   2024-05-06 13:00:00 +0200
categories: release
# Author
author: Guillaume Delahaye (https://github.com/g7ed6e)
---
### Introduction
The CoreWCF 1.6.0 release introduced a new feature that allows to inject incoming message properties as operation contract parameter.

### Implementation
This feature is implemented in the `OperationContractParameterGenerator` which already support injecting service registered in the DI container or `HttpContext` into the operation contract parameters.
The `InjectedAttribute.PropertyName` is exposed to specify the message property to be injected.
```c#
namespace CoreWCF
{
    [AttributeUsage(AttributeTargets.Parameter)]
    public sealed class InjectedAttribute : Attribute
    {
        public string PropertyName { get; set; }
    }
}
```
### NetTcpBinding usage

A service exposing a `NetTcpBinding` can pull the `RemoteEndpointMessageProperty`.

```c#
public interface IHelloWorldService
{
    string SayHello();
}

public class HelloWorldService : IHelloService
{
    public string SayHello([Injected(PropertyName = RemoteEndpointMessageProperty.Name)] RemoteEndpointMessageProperty remoteEndpointMessageProperty)
    {
        return $"Hello from {remoteEndpointMessageProperty.Address}:{remoteEndpointMessageProperty.Port}";
    }
}
```
### BasicHttpBinding usage
A service exposing a `BasicHttpBinding` can pull the `HttpRequestMessageProperty`.

```c#
public interface IHelloWorldService
{
    string SayHello();
}

public class HelloWorldService : IHelloService
{
    public string SayHello([Injected(PropertyName = HttpRequestMessageProperty.Name)] HttpRequestMessageProperty httpRequestMessageProperty)
    {
        return $"Hello from {remoteEndpointMessageProperty.Address}:{remoteEndpointMessageProperty.Port}";
    }
}
```
### Compilation guards
When `PropertyName` is provided a `null` or empty string value a `COREWCF_103` build error is triggered.

### Kafka transport enhancements
In addition, a `KafkaMessageProperty` has been added to the Kafka transport at both client and service ends to provide control over the partition key and the headers attached to the message and transported through the kafka topic.
```c#
namespace CoreWCF.ServiceModel.Channels;

public sealed class KafkaMessageProperty
{
    public static readonly string Name = "CoreWCF.ServiceModel.Channels.KafkaMessageProperty";

    public IList<KafkaMessageHeader> Headers { get; } = new List<KafkaMessageHeader>();
    public byte[] PartitionKey { get; set; }
}
```

```c#
namespace CoreWCF.Channels;

public sealed class KafkaMessageProperty
{
    private readonly IList<KafkaMessageHeader> _headers = new List<KafkaMessageHeader>();

    public const string Name = "CoreWCF.Channels.KafkaMessageProperty";
    
    internal KafkaMessageProperty(ConsumeResult<byte[], byte[]> consumeResult)
    {
        foreach (IHeader messageHeader in consumeResult.Message.Headers)
        {
            _headers.Add(new KafkaMessageHeader(messageHeader.Key, messageHeader.GetValueBytes()));
        }

        PartitionKey = consumeResult.Message.Key;
        Topic = consumeResult.Topic;
    }
    
    public IReadOnlyCollection<KafkaMessageHeader> Headers => _headers as IReadOnlyCollection<KafkaMessageHeader>;
    public ReadOnlyMemory<byte> PartitionKey { get; }
    public string Topic { get; }
}
```

Client side the partition key and the headers can be provided using an `OperationContextScope`.
```c#
using (var scope = new System.ServiceModel.OperationContextScope((System.ServiceModel.IContextChannel)channel))
{
    ServiceModel.Channels.KafkaMessageProperty outgoingProperty = new();
    outgoingProperty.Headers.Add(new ServiceModel.Channels.KafkaMessageHeader("header1", Encoding.UTF8.GetBytes("header1Value")));
    outgoingProperty.PartitionKey = Encoding.UTF8.GetBytes("key");
    System.ServiceModel.OperationContext.Current.OutgoingMessageProperties[ServiceModel.Channels.KafkaMessageProperty.Name] =
        outgoingProperty;
    channel.DoSomething();
}
```
Service side the implementer can get these values back by pulling the `KafkaLessageProperty`
```c#
public void DoSomething([Injected(PropertyName = KafkaMessageProperty.Name)] KafkaMessageProperty kafkaMessageProperty)
{
    IReadOnlyCollection<KafkaMessageHeader> headers = kafkaMessageProperty.Headers;
    ReadOnlyMemory<byte> partitionKey = kafkaMessageProperty.PartitionKey;
    string topic = kafkaMessageProperty.Topic;
}
```
