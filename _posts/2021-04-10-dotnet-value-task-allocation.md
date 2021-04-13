---
layout: post
title:  Why use ValueTask<T>?
categories: [dotnet,valuetask,perf,async]
---

## Allocations and Scalability
`Task` is a reference type, which means every time you await a task, it is allocated on the heap. As in many applications today, you are likely to have a method in your code that either runs synchronously or asynchronously depending on a condition, like whether or not some data is already stored in memory or it needs to be loaded from a longer running I/O operation. A common example in a web application would be storing infrequently changing data into a caching layer to increase the performance of subsequent reads of that data.

```csharp
public async Task<SomeData> GetSomeDataAsync()
{
    const string dataKey = "SOME_DATA_KEY";

    // Synchronous operation
    if (_cache.GetValue<SomeData>(dataKey) is {} data)
        return data;
    
    // Becomes asynchronous operation
    SomeData freshData = await ... // Get from I/O operation

    _cache.SetValue(dataKey, freshData, TimeSpan.FromHours(1))

    return freshData;

}
```

If you have the above method being called from a highly hit API endpoint, the majority of the time this code path will be a synchronous operation, which means that the allocation of the `Task` is essentially wasted memory because by the time the result is returned to the caller, the task is complete. This memory allocation is not terribly large but in a highly scaled scenario it can add up to a significant amount of work for the Garbage Collector that could be used elsewhere.

## Advantage of `ValueTask<T>`
The benefit of using `ValueTask<T>` is that it is a discriminated union like `struct`. Depending on if you are returning synchronously or asynchronously from a method determines if the value is `T` or `Task<T>`.

```csharp
// ValueTask<T> constructors
public ValueTask(Task<TResult> task);
public ValueTask(TResult result);
```

```csharp

public class Program
{
    private SomeData _cachedData;

    public async Task Main(string[] args)
    {
        // Task<SomeData> allocated because async call to DB is required
        var someDataFirst = await GetSomeDataAsync();   
        
        /**
        SomeData is directly returned because the _cachedData field was
        already initialized with data so no Task<SomeData> is allocated.
        **/
        var someDataSecond = await GetSomeDataAsync();
    }
    
    public async Task<SomeData> GetSomeDataAsync()
    {
        if (_cachedData is {} data)
            return data;
        
        return _cachedData = await _dbContext.SomeData.FirstOrDefault();
    }
}
```

## Memory Allocation Benchmark
When running a benchmark against two similar methods where one returns `Task<T>` and one returns `ValueTask<T>` and calling each method 10 times to simulate subsequent calls in a real application. The time performance of the 2 were comparable but the memory allocation for `ValueTask<T>` was almost half, 48.65% reduction in allocated bytes.

[Benchmark Source Code](https://github.com/mroberts91/ValueTasks)

``` ini
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.19041.630 (2004/?/20H1)
Intel Core i5-8350U CPU 1.70GHz (Kaby Lake R), 1 CPU, 8 logical and 4 physical
.NET Core SDK=5.0.201
  [Host]     : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614
  DefaultJob : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614)
```

<br />
<table>
<thead><tr><th>            Method</th><th>Mean</th><th>Error</th><th>StdDev</th><th>Allocated</th>
</tr>
</thead><tbody><tr><td>GetProductsReferenceTask</td><td>18.47 &mu;s</td><td>1.922 &mu;s</td><td>5.544 &mu;s</td><td>1480 B</td>
</tr><tr><td>GetProductsValueTask</td><td>23.15 &mu;s</td><td>2.881 &mu;s</td><td>8.359 &mu;s</td><td>760 B</td>
</tr></tbody></table>
