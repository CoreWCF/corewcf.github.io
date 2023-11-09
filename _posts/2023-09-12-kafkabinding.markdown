---
# Layout
layout: post
title:  "Introducing Kafka transport support in CoreWCF"
date:   2023-10-21 13:00:00 +0200
categories: release
# Author
author: Guillaume Delahaye (https://github.com/g7ed6e)
---
### Introduction
CoreWCF 1.4 release introduced Apache Kafka transport support through the publish of 2 new nuget packages `CoreWCF.Kafka` and `CoreWCF.Kafka.Client`. The Kafka protocol implementation is provided by taking a dependency on the `Confluent.Kafka` nuget package and the underlying `librdkafka` C/C++ library.

### Extensibility

Both server and client packages expose a `KafkaBinding` that should be sufficient to configure security and transport to the broker. 
However in certain scenario it could be useful to finegrain properties of `Confluent.Kafka` / `librdkafka`, this can be achieved by creating a `CustomBinding` and pulling out the `KafkaTransportBindingElement`. This element exposes all the properties exposed by `ConsumerConfig` and `ProducerConfig` from `Confluent.Kafka`.
```c#
var binding = new KafkaBinding(KafkaDeliverySemantics.AtMostOnce)
{
    AutoOffsetReset = AutoOffsetReset.Earliest,
    GroupId = "my-group"
};
var customBinding = new CustomBinding(binding);
KafkaTransportBindingElement transport = customBinding.Elements.Find<KafkaTransportBindingElement>();
transport.Debug = "all";
```

### Security

The table below summarizes the mapping between Confluent.Kafka and KafkaBinding security modes.

| SecurityProtocol | SaslMechanism | ClientCertfiicate | CoreWCF KafkaBinding configuration | 
|--|--|--|--|
| `Plaintext` | N/A |  N/A | `KafkaSecurityMode.None` + `KafkaCredentialType.None` 
| `Ssl`|  N/A | No | `KafkaSecurityMode.Transport` + `KafkaCredentialType.None` + requires configuring CaPem
| |  N/A | Yes | `KafkaSecurityMode.Transport` + `KafkaCredentialType.SslKeyPairCertificate` + requires configuring CaPem + providing a `SslKeyPairCredential` instance
| `SaslPlaintext`| `Gssapi`|  N/A | supported through custom binding  
| | `Plain`|  N/A| `KafkaSecurityMode.TransportCredentialOnly` + `KafkaCredentialType.SaslPlain` + providing a `SaslUsernamePasswordCredential` instance
| | `ScramSha256`|N/A|  supported through custom binding  
| | `ScramSha512`|  N/A|supported through custom binding  
| | `OAuthBearer`|  N/A|supported through custom binding  
| `SaslSsl`| `Gssapi` | N/A|supported through custom binding 
| | `Plain` | N/A|`KafkaSecurityMode.Transport` + `KafkaCredentialType.SaslPlain` + requires configuring CaPem + providing a `SaslUsernamePassword` instance
| | `ScramSha256` | N/A|supported through custom binding
| | `ScramSha512` | N/A|supported through custom binding 
| | `OAuthBearer` | N/A|supported through custom binding 

### Getting started 

First, configure CoreWCF to consume the topic `my-topic` with consumer group id `my-consumer-group`. To specify from which offset the consumer want to start consuming messages the `AutoOffsetReset` property should be provided.
```c#
var builder = WebApplication.CreateBuilder();
builder.Services.AddServiceModelServices().AddQueueTransport()

var app = builder.Build();

app.UseServiceModel(serviceBuilder => 
{
    services.AddService<Service>();
    services.AddServiceEndpoint<Service, IService>(new CoreWCF.Kafka.KafkaBinding
    {
        AutoOffsetReset = AutoOffsetReset.Earliest,
        DeliverySemantics = KafkaDeliverySemantics.AtMostOnce,
        GroupId = "my-consumer-group"
    }, $"net.kafka://localhost:9092/my-topic");
});
```
Then, configure a client to produce messages to topic `my-topic`.
```c#
CoreWCF.ServiceModel.Channels.KafkaBinding kafkaBinding = new();
var factory = new System.ServiceModel.ChannelFactory<IService>(kafkaBinding,
    new System.ServiceModel.EndpointAddress(new Uri($"net.kafka://localhost:9092/my-topic")));
IService channel = factory.CreateChannel();
await channel.CallServiceAsync(name);
```
#### DeliverySemantics

The delivery semantic can be configured at the binding level to `AtLeastOnce` or `AtMostOnce`.
```c#
CoreWCF.Kafka.KafkaBinding = new KafkaBinding
{
    DeliverySemantics = KafkaDeliverySemantics.AtLeastOnce;
}
```

#### ErrorHandlingStrategy and DLQ support

The error handling strategy can be configured at the binding level to `Ignore` or `DeadLetterQueue`.
When specifying `DeadLetterQueue`, `DeadLetterQueueTopic` should also be provided.
```c#
CoreWCF.Kafka.KafkaBinding = new KafkaBinding
{
    ErrorHandlingStrategy = KafkaErrorHandlingStrategy.DeadLetterQueue,
    DeadLetterQueueTopic = "my-topic-DLQ",
}
```