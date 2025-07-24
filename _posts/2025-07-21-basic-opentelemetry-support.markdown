---
# Layout
layout: post
title:  "Basic OpenTelemetry support"
date:   2025-07-21 12:00:00 -0800
categories: release
# Author
author: Stefan Nikolei(https://github.com/stefannikolei)
---

# Introduction

Observability has become a cornerstone of modern application development, and OpenTelemetry has emerged as the industry standard for collecting telemetry data. While the CoreWCF project didn't have built-in OpenTelemetry support, the .NET Framework-based WCF had an existing implementation that served as the foundation for this integration.

The community request for OpenTelemetry support was first raised three years ago in [GitHub issue #790](https://github.com/CoreWCF/CoreWCF/issues/790). Now after that long time CoreWCF includes native OpenTelemetry support, bringing modern observability capabilities to CoreWCF services.


# Implementation Details

## Configuring a CoreWCF Service for OpenTelemetry

Let's walk through setting up a CoreWCF service with OpenTelemetry integration. This example uses .NET Aspire to simplify the setup and provide excellent tooling for visualizing traces, but the integration works with any OpenTelemetry-compatible setup.

### Sample Calculator Service

Here's the `Program.cs` for our WCF calculator service:


```csharp
var builder = WebApplication.CreateBuilder();

// Register CoreWCF services
builder.Services.AddServiceModelServices();
builder.Services.AddServiceModelMetadata();

// This call sets up OpenTelemetry, including CoreWCF instrumentation
builder.AddServiceDefaults();

var app = builder.Build();
app.MapDefaultEndpoints();

// Configure the WCF service
app.UseServiceModel(serviceBuilder =>
{
    serviceBuilder.AddService<ICalculator.Calculator>();
    serviceBuilder.AddServiceEndpoint<ICalculator.Calculator, ICalculator>(
        new BasicHttpBinding(BasicHttpSecurityMode.Transport),
        "/CalculatorService.svc");
});

app.Run();
```

### OpenTelemetry Configuration

The key to enabling OpenTelemetry in CoreWCF is the `builder.AddServiceDefaults()` call, which internally calls `ConfigureOpenTelemetry`. Here's the implementation:

```csharp
public static TBuilder ConfigureOpenTelemetry<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
{
    // Configure OpenTelemetry logging
    builder.Logging.AddOpenTelemetry(logging =>
    {
        logging.IncludeFormattedMessage = true;
        logging.IncludeScopes = true;
    });

    builder.Services.AddOpenTelemetry()
        .WithMetrics(metrics =>
        {
            metrics.AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddRuntimeInstrumentation();
        })
        .WithTracing(tracing =>
        {
            tracing.AddSource(builder.Environment.ApplicationName)
                .AddSource("CoreWCF.Primitives")  // This enables CoreWCF tracing
                .AddHttpClientInstrumentation();
        });

    builder.AddOpenTelemetryExporters();

    return builder;
}
```

### The Critical Configuration Line

The most important line for CoreWCF OpenTelemetry integration is:

```csharp
.AddSource("CoreWCF.Primitives")
```

This tells OpenTelemetry to listen for traces emitted by the CoreWCF instrumentation. The `"CoreWCF.Primitives"` source name is the identifier used by CoreWCF's internal activity source to emit telemetry data.

## Observing the Results

Once configured, your CoreWCF service will automatically start emitting detailed traces for all service operations. Here's what the traces look like for our calculator service:

![OpenTelemetry Traces](/_images/traces.png)

### Comparison: With vs Without CoreWCF Instrumentation

For comparison, here are the same service calls, traced with only the standard ASP.NET Core instrumentation (using `AddAspNetCoreInstrumentation()` instead of `AddSource("CoreWCF.Primitives")`):

![Standard Traces](/_images/traces_no_detail.png)

As you can see, the CoreWCF-specific instrumentation provides much richer details about WCF operation execution, including operation names, binding information, and service-specific context that's not available with the standard instrumentation.

The source code for this example can be found under:
[Example](https://github.com/stefannikolei/CoreWCF-Telemetry-Sample)

## Getting Started

To start using OpenTelemetry with your CoreWCF services:

1. Ensure you're using the latest version of CoreWCF that includes this integration
2. Add OpenTelemetry packages to your project
3. Configure OpenTelemetry with the `"CoreWCF.Primitives"` source
4. Deploy and observe your service traces


## Contributing

This marks the beginning of OpenTelemetry support in CoreWCF. It is only the most basic implementation. If you discover areas that could benefit from additional telemetry or have suggestions for improvements, please [create an issue](https://github.com/CoreWCF/CoreWCF/issues) on the CoreWCF GitHub repository.