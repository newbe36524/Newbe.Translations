---
title: ASP.NET Core 中的内存管理和常用模式
author: rick-anderson
description: 了解如何在 ASP.NET Core 中管理内存，了解垃圾收集器 (GC) 的工作方式。
ms.author: riande
ms.custom: mvc
ms.date: 4/05/2019
no-loc:
  - "ASP.NET Core Identity"
  - cookie
  - Cookie
  - Blazor
  - "Blazor Server"
  - "Blazor WebAssembly"
  - "Identity"
  - "Let's Encrypt"
  - Razor
  - SignalR
uid: performance/memory
---

# ASP.NET Core 中的内存管理和垃圾回收 (GC)

由[Sébastien Ros](https://github.com/sebastienros) 和 [Rick Anderson](https://twitter.com/RickAndMSFT) 编著

内存管理非常复杂，即使在 .NET之类的托管框架中也是如此。 分析和理解内存问题可能极具挑战。 这篇文章:

* 立足于很多 *内存泄露* 和 *GC 无法正常工作* 的问题. 这些问题大多是由于不了解内存在 .NET Core 中的工作方式，或者不了解它的测量方式。
* 演示有问题的内存使用方法，并提出建议使用的其他方法。

## 垃圾回收 (GC) 在 .NET Core 中的工作方式

GC 分配内存“堆”段，其中每个段是一个连续的内存范围。 放置在堆中的对象被分类为 3 个代次中的一个: 0 代、1 代 或者 2 代 代次将确定 GC 尝试释放内存的频率，被释放的内存是指在应用程序中不再引用的托管对象。 代次越小的代次将会被 GC 越频繁的分析并回收。

对象根据其生命周期从一个代次移动到另一代次。 随着对象的生命周期变长，它们会被移动到更高的代次中。 如前文所述，更高的代次被分析和回收的频率越低。 短生命周期对象始终保留在第 0 代中。 例如，在 Web 请求的生命周期中引用的对象是短生命周期的。 应用程序级别的 [单例对象](xref:fundamentals/dependency-injection#service-lifetimes) 通常会被迁移到第 2 代中。

当 ASP.NET Core 应用程序启动时， GC将会:

* 为初始堆段保留一些内存。
* 在运行时加载时提交一小部分内存。

先前的内存分配是出于性能原因而考虑的。 性能将得益于连续的内存中的堆段。

### 调用 GC 回收

显式调用 [GC.Collect](xref:System.GC.Collect*)，需要考虑:

* **不应该**在生产环境的 ASP.NET Core 应用程序中做此尝试。
* 这对内存调查内存泄露非常有用。
* 在调查是，可以验证 GC 已从内存中除去所有不确定对象，以便可以测量内存。

## 分析应用程序的内存使用情况

专用工具可帮助分析内存使用情况:

- 计算对象引用
- 测量GC对CPU使用有多大影响
- 测量用于每个代次的内存空间

可以使用以下工具来分析内存使用情况:

* [dotnet-trace](/dotnet/core/diagnostics/dotnet-trace): 这可以用于生产环境。
* [在没有 Visual Studio 调试器时分析内存使用情况](/visualstudio/profiling/memory-usage-without-debugging2)
* [在 Visual Studio 中的评估内存使用情况](/visualstudio/profiling/memory-usage)

### 检测内存问题

任务管理器可用于了解 ASP.NET 应用程序正在使用多少内存。 任务管理器中的内存值:

* 表示 ASP.NET 进程使用的内存量。
* 包括应用的托管对象和其他内存消费对象，例如非托管内存的使用量。

如果任务管理器内存值无限增加且永不会稳定，那么说明应用程序正发生内存泄漏。 以下部分将演示并说明若干内存的使用模式。

## 展示内存使用方式的样例应用

[MemoryLeak 样例应用](https://github.com/sebastienros/memoryleak) 源码公开在 GitHub 上。 该应用：

* 包含一个诊断信息的 controller，用于收集应用程序的实时内存和 GC 数据。
* 包含一个用于展示内存和 GC 数据的首页。 该页面每秒会自动刷新一次。
* 包含提供各种内存负载模式的 API controller 。
* 虽然这不是一个长期维护的工具，不过仍然可用于演示 ASP.NET Core 应用程序的内存使用模式。

运行 MemoryLeak 应用时， 内存将会在 GC 发生时被回收。 内存占用会随着该工具分配自定义对象时而增加。 下面的图片显示当 Gen 0 GC 发生时 MemoryLeak 首页显示的情况。 该图表显示当前有 0 个 RPS (每秒请求数 ) ，因为没有调用者调用 API controller 。

![preceding chart](memory/_static/0RPS.png)

此图表显示内存使用率的两个值:

- Allocated: 托管对象占用的内存量
- [Working set](/windows/win32/memory/working-set): 当前驻留在物理内存中的进程的虚拟地址空间中的页集。 Working set 和任务管理器显示的数值是相同的。

### 瞬时对象

The following API creates a 10-KB String instance and returns it to the client. On each request, a new object is allocated in memory and written to the response. Strings are stored as UTF-16 characters in .NET so each character takes 2 bytes in memory.

```csharp
[HttpGet("bigstring")]
public ActionResult<string> GetBigString()
{
    return new String('x', 10 * 1024);
}
```

The following graph is generated with a relatively small load in to show how memory allocations are impacted by the GC.

![preceding chart](memory/_static/bigstring.png)

The preceding chart shows:

* 4K RPS (Requests per second).
* Generation 0 GC collections occur about every two seconds.
* The working set is constant at approximately 500 MB.
* CPU is 12%.
* The memory consumption and release (through GC) is stable.

The following chart is taken at the max throughput that can be handled by the machine.

![preceding chart](memory/_static/bigstring2.png)

The preceding chart shows:

* 22K RPS
* Generation 0 GC collections occur several times per second.
* Generation 1 collections are triggered because the app allocated significantly more memory per second.
* The working set is constant at approximately 500 MB.
* CPU is 33%.
* The memory consumption and release (through GC) is stable.
* The CPU (33%) is not over-utilized, therefore the garbage collection can keep up with a high number of allocations.

### Workstation GC vs. Server GC

The .NET Garbage Collector has two different modes:

* **Workstation GC**: Optimized for the desktop.
* **Server GC**. The default GC for ASP.NET Core apps. Optimized for the server.

The GC mode can be set explicitly in the project file or in the *runtimeconfig.json* file of the published app. The following markup shows setting `ServerGarbageCollection` in the project file:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

Changing `ServerGarbageCollection` in the project file requires the app to be rebuilt.

**Note:** Server garbage collection is **not** available on machines with a single core. For more information, see <xref:System.Runtime.GCSettings.IsServerGC>.

The following image shows the memory profile under a 5K RPS using the Workstation GC.

![preceding chart](memory/_static/workstation.png)

The differences between this chart and the server version are significant:

- The working set drops from 500 MB to 70 MB.
- The GC does generation 0 collections multiple times per second instead of every two seconds.
- GC drops from 300 MB to 10 MB.

On a typical web server environment, CPU usage is more important than memory, therefore the Server GC is better. If memory utilization is high and CPU usage is relatively low, the Workstation GC might be more performant. For example, high density hosting several web apps where memory is scarce.

<a name="sc"></a>

### GC using Docker and small containers

When multiple containerized apps are running on one machine, Workstation GC might be more preformant than Server GC. For more information, see [Running with Server GC in a Small Container](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-0/) and [Running with Server GC in a Small Container Scenario Part 1 – Hard Limit for the GC Heap](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-1-hard-limit-for-the-gc-heap/).

### Persistent object references

The GC cannot free objects that are referenced. Objects that are referenced but no longer needed result in a memory leak. If the app frequently allocates objects and fails to free them after they are no longer needed, memory usage will increase over time.

The following API creates a 10-KB String instance and returns it to the client. The difference with the previous example is that this instance is referenced by a static member, which means it's never available for collection.

```csharp
private static ConcurrentBag<string> _staticStrings = new ConcurrentBag<string>();

[HttpGet("staticstring")]
public ActionResult<string> GetStaticString()
{
    var bigString = new String('x', 10 * 1024);
    _staticStrings.Add(bigString);
    return bigString;
}
```

The preceding code:

* Is an example of a typical memory leak.
* With frequent calls, causes app memory to increases until the process crashes with an `OutOfMemory` exception.

![preceding chart](memory/_static/eternal.png)

In the preceding image:

* Load testing the `/api/staticstring` endpoint causes a linear increase in memory.
* The GC tries to free memory as the memory pressure grows, by calling a generation 2 collection.
* The GC cannot free the leaked memory. Allocated and working set increase with time.

Some scenarios, such as caching, require object references to be held until memory pressure forces them to be released. The <xref:System.WeakReference> class can be used for this type of caching code. A `WeakReference` object is collected under memory pressures. The default implementation of <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> uses `WeakReference`.

### Native memory

Some .NET Core objects rely on native memory. Native memory can **not** be collected by the GC. The .NET object using native memory must free it using native code.

.NET provides the <xref:System.IDisposable> interface to let developers release native memory. Even if <xref:System.IDisposable.Dispose*> is not called, correctly implemented classes call `Dispose` when the [finalizer](/dotnet/csharp/programming-guide/classes-and-structs/destructors) runs.

Consider the following code:

```csharp
[HttpGet("fileprovider")]
public void GetFileProvider()
{
    var fp = new PhysicalFileProvider(TempPath);
    fp.Watch("*.*");
}
```

[PhysicalFileProvider](/dotnet/api/microsoft.extensions.fileproviders.physicalfileprovider?view=dotnet-plat-ext-3.0) is a managed class, so any instance will be collected at the end of the request.

The following image shows the memory profile while invoking the `fileprovider` API continuously.

![preceding chart](memory/_static/fileprovider.png)

The preceding chart shows an obvious issue with the implementation of this class, as it keeps increasing memory usage. This is a known problem that is being tracked in [this issue](https://github.com/dotnet/aspnetcore/issues/3110).

The same leak could be happen in user code, by one of the following:

* Not releasing the class correctly.
* Forgetting to invoke the `Dispose`method of the dependent objects that should be disposed.

### Large objects heap

Frequent memory allocation/free cycles can fragment memory, especially when allocating large chunks of memory. Objects are allocated in contiguous blocks of memory. To mitigate fragmentation, when the GC frees memory, it tries to defragment it. This process is called **compaction**. Compaction involves moving objects. Moving large objects imposes a performance penalty. For this reason the GC creates a special memory zone for _large_ objects, called the [large object heap](/dotnet/standard/garbage-collection/large-object-heap) (LOH). Objects that are greater than 85,000 bytes (approximately 83 KB) are:

* Placed on the LOH.
* Not compacted.
* Collected during generation 2 GCs.

When the LOH is full, the GC will trigger a generation 2 collection. Generation 2 collections:

* Are inherently slow.
* Additionally incur the cost of triggering a collection on all other generations.

The following code compacts the LOH immediately:

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();
```

See <xref:System.Runtime.GCSettings.LargeObjectHeapCompactionMode> for information on compacting the LOH.

In containers using .NET Core 3.0 and later, the LOH is automatically compacted.

The following API that illustrates this behavior:

```csharp
[HttpGet("loh/{size=85000}")]
public int GetLOH1(int size)
{
   return new byte[size].Length;
}
```

The following chart shows the memory profile of calling the `/api/loh/84975` endpoint, under maximum load:

![preceding chart](memory/_static/loh1.png)

The following chart shows the memory profile of calling the `/api/loh/84976` endpoint, allocating *just one more byte*:

![preceding chart](memory/_static/loh2.png)

Note: The `byte[]` structure has overhead bytes. That's why 84,976 bytes triggers the 85,000 limit.

Comparing the two preceding charts:

* The working set is similar for both scenarios, about 450 MB.
* The under LOH requests (84,975 bytes) shows mostly generation 0 collections.
* The over LOH requests generate constant generation 2 collections. Generation 2 collections are expensive. More CPU is required and throughput drops almost 50%.

Temporary large objects are particularly problematic because they cause gen2 GCs.

For maximum performance, large object use should be minimized. If possible, split up large objects. For example, [Response Caching](xref:performance/caching/response) middleware in ASP.NET Core split the cache entries into blocks less than 85,000 bytes.

The following links show the ASP.NET Core approach to keeping objects under the LOH limit:

* [ResponseCaching/Streams/StreamUtilities.cs](https://github.com/dotnet/AspNetCore/blob/v3.0.0/src/Middleware/ResponseCaching/src/Streams/StreamUtilities.cs#L16)
* [ResponseCaching/MemoryResponseCache.cs](https://github.com/aspnet/ResponseCaching/blob/c1cb7576a0b86e32aec990c22df29c780af29ca5/src/Microsoft.AspNetCore.ResponseCaching/Internal/MemoryResponseCache.cs#L55)

For more information, see:

* [Large Object Heap Uncovered](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [Large object heap](/dotnet/standard/garbage-collection/large-object-heap)

### HttpClient

Incorrectly using <xref:System.Net.Http.HttpClient> can result in a resource leak. System resources, such as database connections, sockets, file handles, etc.:

* Are more scarce than memory.
* Are more problematic when leaked than memory.

Experienced .NET developers know to call <xref:System.IDisposable.Dispose*> on objects that implement <xref:System.IDisposable>. Not disposing objects that implement `IDisposable` typically results in leaked memory or leaked system resources.

`HttpClient` implements `IDisposable`, but should **not** be disposed on every invocation. Rather, `HttpClient` should be reused.

The following endpoint creates and disposes a new  `HttpClient` instance on every request:

```csharp
[HttpGet("httpclient1")]
public async Task<int> GetHttpClient1(string url)
{
    using (var httpClient = new HttpClient())
    {
        var result = await httpClient.GetAsync(url);
        return (int)result.StatusCode;
    }
}
```

Under load, the following error messages are logged:

```
fail: Microsoft.AspNetCore.Server.Kestrel[13]
      Connection id "0HLG70PBE1CR1", Request id "0HLG70PBE1CR1:00000031":
      An unhandled exception was thrown by the application.
System.Net.Http.HttpRequestException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted --->
    System.Net.Sockets.SocketException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted
   at System.Net.Http.ConnectHelper.ConnectAsync(String host, Int32 port,
    CancellationToken cancellationToken)
```

Even though the `HttpClient` instances are disposed, the actual network connection takes some time to be released by the operating system. By continuously creating new connections, _ports exhaustion_ occurs. Each client connection requires its own client port.

One way to prevent port exhaustion is to reuse the same `HttpClient` instance:

```csharp
private static readonly HttpClient _httpClient = new HttpClient();

[HttpGet("httpclient2")]
public async Task<int> GetHttpClient2(string url)
{
    var result = await _httpClient.GetAsync(url);
    return (int)result.StatusCode;
}
```

The `HttpClient` instance is released when the app stops. This example shows that not every disposable resource should be disposed after each use.

See the following for a better way to handle the lifetime of an `HttpClient` instance:

* [HttpClient and lifetime management](../fundamentals/http-requests.md#httpclient-and-lifetime-management)
* [HTTPClient factory blog](https://devblogs.microsoft.com/aspnet/asp-net-core-2-1-preview1-introducing-httpclient-factory/)

### Object pooling

The previous example showed how the `HttpClient` instance can be made static and reused by all requests. Reuse prevents running out of resources.

Object pooling:

* Uses the reuse pattern.
* Is designed for objects that are expensive to create.

A pool is a collection of pre-initialized objects that can be reserved and released across threads. Pools can define allocation rules such as limits, predefined sizes, or growth rate.

The NuGet package [Microsoft.Extensions.ObjectPool](https://www.nuget.org/packages/Microsoft.Extensions.ObjectPool/) contains classes that help to manage such pools.

The following API endpoint instantiates a `byte` buffer that is filled with random numbers on each request:

```csharp
        [HttpGet("array/{size}")]
        public byte[] GetArray(int size)
        {
            var random = new Random();
            var array = new byte[size];
            random.NextBytes(array);

            return array;
        }
```

The following chart display calling the preceding API with moderate load:

![preceding chart](memory/_static/array.png)

In the preceding chart, generation 0 collections happen approximately once per second.

The preceding code can be optimized by pooling the `byte` buffer by using [ArrayPool\<T>](xref:System.Buffers.ArrayPool`1). A static instance is reused across requests.

What's different with this approach is that a pooled object is returned from the API. That means:

* The object is out of your control as soon as you return from the method.
* You can't release the object.

To set up disposal of the object:

* Encapsulate the pooled array in a disposable object.
* Register the pooled object with [HttpContext.Response.RegisterForDispose](xref:Microsoft.AspNetCore.Http.HttpResponse.RegisterForDispose*).

`RegisterForDispose` will take care of calling `Dispose`on the target object so that it's only released when the HTTP request is complete.

```csharp
private static ArrayPool<byte> _arrayPool = ArrayPool<byte>.Create();

private class PooledArray : IDisposable
{
    public byte[] Array { get; private set; }

    public PooledArray(int size)
    {
        Array = _arrayPool.Rent(size);
    }

    public void Dispose()
    {
        _arrayPool.Return(Array);
    }
}

[HttpGet("pooledarray/{size}")]
public byte[] GetPooledArray(int size)
{
    var pooledArray = new PooledArray(size);

    var random = new Random();
    random.NextBytes(pooledArray.Array);

    HttpContext.Response.RegisterForDispose(pooledArray);

    return pooledArray.Array;
}
```

Applying the same load as the non-pooled version results in the following chart:

![preceding chart](memory/_static/pooledarray.png)

The main difference is allocated bytes, and as a consequence much fewer generation 0 collections.

## Additional resources

* [Garbage Collection](/dotnet/standard/garbage-collection/)
* [Understanding different GC modes with Concurrency Visualizer](https://blogs.msdn.microsoft.com/seteplia/2017/01/05/understanding-different-gc-modes-with-concurrency-visualizer/)
* [Large Object Heap Uncovered](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [Large object heap](/dotnet/standard/garbage-collection/large-object-heap)