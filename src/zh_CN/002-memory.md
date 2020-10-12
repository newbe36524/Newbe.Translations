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

先前的内存分配是出于性能原因而考虑的。 连续的内存中的堆段有利于性能的提升。

### 调用 GC 回收

显式调用 [GC.Collect](xref:System.GC.Collect*)，需要考虑:

* **不应该**在生产环境的 ASP.NET Core 应用程序中做此尝试。
* 这对排查内存泄露非常有用。
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

如果任务管理器内存值持续不停地增加，那么说明应用程序正发生内存泄漏。 以下部分将演示并说明若干内存的使用模式。

## 展示内存使用方式的样例应用

[MemoryLeak 样例应用](https://github.com/sebastienros/memoryleak) 源码公开在 GitHub 上。 该应用：

* 包含一个诊断信息的 controller，用于收集应用程序的实时内存和 GC 数据。
* 包含一个用于展示内存和 GC 数据的首页。 该页面每秒会自动刷新一次。
* 包含提供各种内存负载模式的 API controller 。
* 虽然这不是一个长期维护的工具，不过仍然可用于演示 ASP.NET Core 应用程序的内存使用模式。

运行 MemoryLeak 应用时， 内存将会在 GC 发生时被回收。 内存占用会随着该工具分配自定义对象时而增加。 下面的图片显示当 0 代 GC 发生时 MemoryLeak 首页显示的情况。 该图表显示当前有 0 个 RPS (每秒请求数 ) ，因为没有调用者调用 API controller 。

![preceding chart](memory/_static/0RPS.png)

此图表显示内存使用率的两个值:

- Allocated: 托管对象占用的内存量
- [Working set](/windows/win32/memory/working-set): 当前驻留在物理内存中的进程的虚拟地址空间中的页集。 Working set 和任务管理器显示的数值是相同的。

### 瞬时对象

以下 API 将创建 10-KB 字符串实例并将其返回给客户端。 每次请求时，都会有一个新对象会被分配到内存并写入响应。 字符串在 .NET 中存储为 UTF-16 字符，因此每个字符在内存中需要 2 个字节。

```csharp
[HttpGet("bigstring")]
public ActionResult<string> GetBigString()
{
    return new String('x', 10 * 1024);
}
```

下图展示了在负载相对较小时，内存分配是如何受到 GC 的影响。

![preceding chart](memory/_static/bigstring.png)

上图显示：

* 4K RPS (每秒请求数 )。
* 0 代 GC 收集大约每两秒钟发生一次。
* Working set 约为 500 MB 。
* CPU 为 12%。
* 内存消耗和释放 (通过 GC ) 是稳定的。

以下图为采用机器最大吞吐量负载时的情况。

![preceding chart](memory/_static/bigstring2.png)

上图显示：

* 22K RPS
* Gen 0 GC 收集每秒都会发生若干次。
* Gen 1 收集将会被触发，因为应用程序每秒分配的内存大大增加。
* Working set 约为 500 MB 。
* CPU 为 33%。
* 内存消耗和释放 (通过 GC ) 是稳定的。
* CPU（33%）的使用率并没有过高，因此垃圾收集可以跟上大量的内存分配。

### Workstation GC 和 Server GC

.NET 垃圾收集器具有两种不同的方式:

* **Workstation GC**: 专为桌面系统优化。
* **Server GC**. ASP.NET Core 应用程序的默认 GC 方式。 针对服务器环境进行优化。

GC 模式可以在项目文件或发布的应用程序的 *runtimeconfig.json* 文件中显式设置。 以下标记显示在项目文件中如何设置 `ServerGarbageCollection`:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

在项目文件中更改 `ServerGarbageCollection` 需要重新生成应用程序。

**注意：** Server GC **在有单个核心的机器是上不可用的**。 有关更多信息，请参阅 <xref:System.Runtime.GCSettings.IsServerGC>。

下图显示了使用 Workstation GC 在 5K RPS 下的内存概要情况。

![preceding chart](memory/_static/workstation.png)

此图表与服务器版本之间的差异很大:

- Working set 从 500 MB 下降到 70 MB。
- Gen 0 GC 每秒数次，而不是每两秒钟一次。
- GC 从 300 MB 下降到 10 MB。

在典型的 Web 服务器环境中， CPU 使用率比内存更重要，因此 Server GC 更好。 如果内存利用率较高且 CPU 使用率相对较低，那么 Workstation GC 可能更高性能。 例如，高密度托管多个内存不足的 Web 应用程序。

<a name="sc"></a>

### GC 在 Docker 和小型 container 场景中的使用

当多个容器化应用程序在一台机器上运行时，Workstation GC 可能比 Server GC 更具有优势。 有关更多信息，请参阅 [Running with Server GC in a Small Container](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-0/)和 [Running with Server GC in a Small Container Scenario Part 1 – Hard Limit for the GC Heap](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-1-hard-limit-for-the-gc-heap/)。

### 持续性的对象引用

GC 不能释放被引用的对象。 对象虽然被引用但是却不被使用会导致内存泄漏。 如果应用程序频繁分配对象，且在它们用完以后（后继也不再使用）不释放的话，内存使用率会随着时间增加。

下面，我们来创建一个10KB的字符串实例并将它返回给客户端。 和之前的例子不同的是，这个实例被静态成员所引用，这意味着它永远不可能被收集。

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

上面的代码:

* 是一个典型的内存泄漏示例。
* 通过频繁调用，导致应用程序内存增加，最终使进程崩溃引发 `OutOfMemory` 异常

![preceding chart](memory/_static/eternal.png)

上面的图中:

* 压力测试时 `/api/staticstring` 会导致内存的线性增加。
* GC在内存压力增加时，试图通过调用2代内存的回收来释放内存。
* GC无法释放（由于错误的用法而导致）泄漏的内存， 已分配内存和工作集随着时间增加。

在某些场景中（比如 缓存），需要对象持续保留，直到内存压力将它们强制释放。 <xref:System.WeakReference>类可以用于这种缓存场景。 A `WeakReference` object is collected under memory pressures. <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> 的默认实现为 `WeakReference`。

### 本机内存

一些 .NET Core 对象依赖于本机内存。 GC **不能**回收本机内存。 使用本机内存的 .NET 对象必须使用本机代码将其释放。

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