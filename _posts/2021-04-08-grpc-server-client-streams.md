---
layout: post
title:  gRPC Server to Client Streams
categories: [gRPC,aspnetcore,streams]
---

## HTTP/2 and gRPC
HTTP/2 has solved many of the problems that web technologies faced with using HTTP/1.x protocols. Features like multiplexing over a single TCP connection and binary transport have foster adoption of enhanced data transfer tech such a [gRPC](https://grpc.io/) which is a Cloud Native Computing Foundation incubation project.

## ASP.NET Core Tooling
With the release of .NET Core 3, some really great gRPC tooling has been introduced which makes it pretty seamless to add gRPC support to new or existing ASP.NET Core applications simply by opening up the Connected Services tab within Visual Studio and creating a protocol buffer definition file (.proto file) like the one from my sample project below which defines 3 endpoints with their strongly typed request and response objects.

[Sample Application Source Code](https://github.com/mroberts91/GrpcStreams)

```protobuf
// symbol.proto
syntax = "proto3";

option csharp_namespace = "Streams.Stocks";

package stocks;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// RPC endpoints for an example stock pricing application
service StockSymbols {
	// Returns an list of symbols
	rpc Symbols (google.protobuf.Empty) returns (SymbolsResponse);

	// Return the data of the requested symbol
	rpc Symbol (SymbolRequest) returns (SymbolResponse);

	// Open a stream of symbol updates for the requested symbol
	rpc SymbolStream (SymbolRequest) returns (stream SymbolResponse);
}

// Request and Response Type Definitions
message SymbolsResponse{
	repeated string symbolList = 1;
}

message SymbolRequest{
	string symbol = 1;
}

message SymbolResponse{
	string symbol = 1;
	double currentPrice = 2;
	google.protobuf.Timestamp updated = 3;
}
```

## Implementation
Once the protobuf file is added and configured, from the Connected Services tab you can choose to generate a class or classes based on the how you are going to use the service in your application, either Server, Client or both Server and Client. 

### Server Side
Creating an implementation class that overrides the virtual methods in the generated base class to provide your required functionality for each of the 3 RPC endpoints.

```csharp
// SymbolService.cs
// Methods bodies omitted
public class SymbolsService : StockSymbols.StockSymbolsBase
{
    public override async Task SymbolStream(SymbolRequest request, IServerStreamWriter<SymbolResponse> responseStream, ServerCallContext context) { }

    public override Task<SymbolsResponse> Symbols(Empty request, ServerCallContext context) { }

    public override Task<SymbolResponse> Symbol(SymbolRequest request, ServerCallContext context) { }
}
```

### Client Side
Creating a gRPC client can be done in the same way as the server, you can either create a new class that inherits from the generated base class or just use the client directly depending on how you want to encapsulate your request and response logic. I like encapsulating in a new service as to not expose implementation details to consumers of the data, which makes for cleaner code in my opinion.

```csharp
// SymbolService.cs
public class SymbolService : StockSymbols.StockSymbolsClient, ISymbolService
{
    public SymbolService(GrpcChannel channel) : base(channel) { }

    public IAsyncEnumerable<SymbolResponse> SymbolStream(SymbolRequest request, CancellationToken cancellationToken = default) =>
        base.SymbolStream(request, cancellationToken: cancellationToken).ResponseStream.ReadAllAsync(cancellationToken);

    public async Task<IEnumerable<string>> SymbolsAsync(CancellationToken cancellationToken = default)
    {
        var response = await base.SymbolsAsync(new(), cancellationToken: cancellationToken).ResponseAsync;
        return response?.SymbolList?.ToList() ?? new List<string>();
    }

    public async Task<SymbolResponse> SymbolAsync(SymbolRequest request, CancellationToken cancellationToken = default) =>
        await base.SymbolAsync(request, cancellationToken: cancellationToken).ResponseAsync;
}
```

Now all that is left to do it wire up your dependency injection and your off to the races.

```csharp
// GrpcExtensions.cs
public static class GrpcExtensions
{
    public static IServiceCollection AddGrpcClient(this IServiceCollection services, IConfiguration configuration)
    {
        var host = configuration.GetValue<string>("BACKEND_HTTPS_SERVICE_HOST");
        var port = configuration.GetValue<string>("BACKEND_HTTPS_SERVICE_PORT");
        var protocol = configuration.GetValue<string>("BACKEND_HTTPS_SERVICE_PROTOCOL");

        // Add a transient GRPC channel to the IOC
        services.AddTransient(sp => GrpcChannel.ForAddress($"{protocol}://{host}:{port}"));
        services.AddTransient<ISymbolService, SymbolService>();

        return services;
    }
}
```

The sample application uses a Blazor frontend to consume the gRPC data from the backend.

```csharp
@page "/stock/{symbol?}"
@inject ISymbolService SymbolService
@inject NavigationManager NavigationManager

// .. HTML 

@code {
    ...

    private async Task GetStreamDataAsync()
    {
        var stream = SymbolService.SymbolStream(new() { Symbol = Symbol }, _tokenSource.Token);
        await foreach (SymbolResponse response in stream)
        {
            SymbolData = response;
            await InvokeAsync(() => StateHasChanged());
        }
    }

    private void OnLocationChanged(object? sender, LocationChangedEventArgs? args)
    {
        _tokenSource.Cancel();
    }
}
```


