---
layout: post
title:  .NET Options Pattern
categories: [dotnet,options]
---

## Strongly Typed Configuration Values
Using the options pattern allows you to define classes that provide strongly typed configuration groups available in dependecy injection. There are 3 main types that are used when requesting options from DI, `IOptions<T>`,` IOptionsMonitor<T>`, `IOptionsSnapshot<T>`. In order to properly bind to the defined classes, they must be non-abstract public classes with a public parameterless constructor as well as having public read-write properties.

[Options Pattern Example Source Code](https://github.com/mroberts91/OptionsPatternExample)

## Configuration Binding
Define as class that meets the requirements for binding:
```csharp
public class OptionsClass
{
  public string SomeValue { get; set; }
  public int AnotherValue { get; set; }
}
```

In your `Startup.cs` or wherever your building your `IServiceCollection`, bind a configuration section from `appsettings.json` to that type.
```json
//appsettings.json
{
  "MyOptions":{
    "SomeValue": "I love .NET",
    "AnotherValue": 10
  }
}
```
```csharp
// Startup.ConfigureServices
services.AddOptions<OptionsClass>()
  .BindConfiguration("MyOptions");
```

## IOptions\<T>
Injecting the options into a class.
```csharp
[ApiController]
[Route("[controller]")]
public class ConfigurationController : ControllerBase
{
    private readonly ServiceAOptions _serviceAOptions;

    public ConfigurationController(IOptions<ServiceAOptions> serviceAOptions)
    {
        _serviceAOptions = serviceAOptions.Value;
    }

    [HttpGet]
    public IActionResult Get() => Ok(_serviceAOptions);
}
```

One thing to note when using `IOptions<T>` is that if you update `appsettings.json` while the application is still running, the changes to configuration will not be picked up until the application is restarted. But luckily the .NET team, as usual, has a solution to your problem.
  
## IOptionsSnapshot\<T>
Going the options snapshot route may be a good option for you if there are values in the application that are updated on a regular basis or critical to in an emergency, which a loss in service to users. `IOptionsSnapshot<T>` is a scoped service and provides a snapshot of the options at the time the options object is constructed. Options snapshots are designed for use with transient and scoped dependencies.
```csharp
[ApiController]
[Route("[controller]")]
public class ConfigurationController : ControllerBase
{
    private readonly IServiceOptions _serviceAOptions;
    private readonly IServiceOptions _serviceBOptions;

    public ConfigurationController(IOptions<ServiceAOptions> serviceAOptions,
                                   IOptionsSnapshot<ServiceBOptions> serviceBOptions)
    {
        _serviceAOptions = serviceAOptions.Value;
        _serviceBOptions = serviceBOptions.Value;
    }

    [HttpGet]
    public IActionResult Get()
        => Ok(new {ServiceA = _serviceAOptions.GetOptions(), ServiceB = _serviceBOptions.GetOptions());

}
```
App settings at app startup and result from calling the configuration route.
```json
// appsettings.json
{
  "ServiceA": {
    "ServiceUrl": "http://ServiceA.com/v1",
    "AppIdentifier": "8",
  },
  "ServiceB": {
    "ServiceUrl": "http://ServiceB.com/v1",
    "AppIdentifier": "5",
  },
}
```
```json
// /Configuration route response
{
  "serviceA": {
    "serviceUrl": "http://ServiceA.com/v1",
    "appIdentifier": "8"
  },
  "serviceB": {
    "serviceUrl": "http://ServiceB.com/v1",
    "appIdentifier": "5"
  }
}
```

Now lets update both AppIdentifier values while the application is still running.
```json
// appsettings.json
{
  "ServiceA": {
    "ServiceUrl": "http://ServiceA.com/v1",
    "AppIdentifier": "90000",
  },
  "ServiceB": {
    "ServiceUrl": "http://ServiceB.com/v1",
    "AppIdentifier": "90010",
  },
}
```
```json
// /Configuration route response
{
  "serviceA": {
    "serviceUrl": "http://ServiceA.com/v1",
    "appIdentifier": "8"
  },
  "serviceB": {
    "serviceUrl": "http://ServiceB.com/v1",
    "appIdentifier": "90010"
  }
}
```

Becuase for ServiceB `IOptionsSnapshot<ServiceBOptions>` was injected into the controller, the changes were picked up on the fly.

[Options Pattern Example Source Code](https://github.com/mroberts91/OptionsPatternExample)

## IOptionsMonitor\<T>
`IOptionsMonitor<T>`, which is similar to `IOptionsSnapshot<T>`, is a singleton service that retrieves current option values at any time, which makes it useful for injecting options into singleton services since you can not inject a scoped `IOptionsSnapshot<T>` since the constructor for the singleton is only called on the initial instantiation.
