---
layout: post
title:  .NET Console App with Platform Extensions
categories: [dotnet,net5.0,console,dependency injection]
---

## .NET Platform Extensions FTW
Of all the things I have enjoyed with .NET core and the new unification in .NET 5.0, the new platform extensions have been one of my favorites. Never have I had such a simpler experience of adding logging or dependency injection to a basic console application than with the extensions suite that the dotnet team has added. NuGet Packages such as `Microsoft.Extensions.Logging` or `Microsoft.Extensions.DependencyInjection`.


Pre .NET core 3.0, if I was going to need a little one off utility to perform some quick tasks, let's say to format/clean an Excel dataset and POST it to a service, I would probably have been reaching for Powershell. But now I can stay in the programing language I love, `C#`, and write these utilities in less time while being more comfortable in the decisions I am making in the code.

## Setting up a console app on steroids
### Traditional Control Flow
Depending on how complex your control flow needs to be or how much cleaning and validation you need to do this can get messy and confusing in a hurry.

```csharp
// Program.cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text.Json;

public class Program
{
    private static readonly Queue<SimpleDataStructure> _queue = new();
    static void Main(string[] args)
    {
        Console.WriteLine("Starting Utility ...");
        Console.WriteLine("Reading data into memory");

        ReadAndLoadDataToQueue();
        PrepDataForRequest();

        while (_queue.Any())
        {
            SimpleDataStructure record = _queue.Dequeue();
            using var httpClient = new HttpClient();
            var json = JsonSerializer.Serialize(record);
            StringContent content = new(json);
            httpClient.PostAsync("http://internalservice/api/data", content);
        }

        Console.WriteLine("All data sent to service");
    }
}
```

Another hidden problem in this code snippet is the recurring instantiation of an `HttpClient` in every iteration. One might assume that because with have the using statement in front of the declaration that we would be safe, but even though the object will be cleaned up at the end of the scope, the underlying socket with not be immediately released which can lead to a socket exhaustion condition. If you are familiar with ASP.NET Core development, you know we can quickly and easily add an IHttpClientFactory which supplies the lifetime management of the underlying `HttpMessageHandler`. More information on the subject can be found in this Microsoft link, [IHttpClientFactory Doc](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)

### Setup with Platform Extensions
For most use cases these 3 NuGet packages are my go to in any console setup.
```csharp
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="5.0.1" />
  <PackageReference Include="Microsoft.Extensions.Hosting" Version="5.0.0" />
  <PackageReference Include="Microsoft.Extensions.Logging" Version="5.0.0" />
</ItemGroup>
```

If I am going to be accessing or consuming any HTTP resources then I will add the Http extensions.
```csharp
<PackageReference Include="Microsoft.Extensions.Http" Version="5.0.0" />
```

Our `Program.cs` file changes from containing the majority of the control flow and logic to just what is required to start and stop the application.

```csharp
// Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System;
using System.Threading.Tasks;

public class Program
{
    public static async Task Main(string[] args)
    {
    
        using var host = Host.CreateDefaultBuilder()
            .ConfigureServices(services =>
            {
                services.AddSingleton<IQueueManager, QueueManager>();
                services.AddSingleton<App>();

                services.AddHttpClient("InternalService", config =>
                {
                    config.BaseAddress = new Uri("http://internalservice/api/");
                });
            })
            .Build();

        var logger = host.Services.GetRequiredService<ILogger<Program>>();

        try
        {
            await host.StartAsync();

            var app = host.Services.GetRequiredService<App>();

            // Run the Application, Main logic is in App.cs
            await app.RunAsync();

            await host.WaitForShutdownAsync()
                .ContinueWith(task =>
                {
                    logger.LogInformation("{msg}", "Host Shutting Down ...");
                    return task;
                });
        }
        catch (Exception ex)
        {
            logger.LogCritical(ex, "{msg}", "Host Terminated Unexpectedly ...");
            await host.StopAsync();
        }
    
    }
}
```

This gives you the flexibility to move logic into more structured locations and use more of an ASP.NET style programing model, by injecting services into the `App` class. For me, personally, just the added benefits of being able to inject an `IHttpClientFactory`, for HTTP intensive applications, makes it a go to for me ever time. It really just works, and you don't have the burden of having to manage client and message handler lifetimes.

```csharp
//App.cs
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

public class App
{
    private readonly IHostApplicationLifetime _host;
    private readonly ILogger<App> _logger;
    private readonly IHttpClientFactory _clientFactory;
    private readonly IQueueManager _queueManager;

    public App(IHostApplicationLifetime host, ILogger<App> logger,
                IHttpClientFactory clientFactory, IQueueManager queueManager)
    {
        _host = host;
        _logger = logger;
        _clientFactory = clientFactory;
        _queueManager = queueManager;
    }

    public async Task RunAsync()
    {
        await ReadAndLoadDataToQueueAsync();
        try
        {
            _logger.LogInformation("Preparing to post records");

            while (await _queueManager.DequeueAsync() is { } record)
            {
                using var client = _clientFactory.CreateClient("InternalService");
                var json = JsonSerializer.Serialize(record);
                await client.PostAsync("data", new StringContent(json));
            }

            _logger.LogInformation("Posting records complete");
            _host.StopApplication();
        }
        catch (Exception ex)
        {
            // Log the error and signal the host to stop the application.
            _logger.LogError(ex, "Encountered an unexpected exception: {message}", ex.Message);
            _host.StopApplication();
        }

    }
}
```

Another way to handle these kinds of applications where you are loading a bunch of data to a queue then processing all the records would be the Hosted Service, which I wll dig into in a later post. 
