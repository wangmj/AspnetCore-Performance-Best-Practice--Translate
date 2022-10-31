# ASP.NET Core 性能最佳实践

本文提供 ASP.NET Core 性能最佳实践的引导。

## 攻击性缓存

本文档有多个地方讨论缓存，详细信息链接如下 Response caching in ASP.NET Core | Microsoft Learn

## 掌握热代码路径

本文档中，热代码是指，经常被调用的或能够被很多次执行的代码。热代码通常限制了程序的扩展（缩放）和性能，并且该文档中有多个地方讨论到。

## 避免死循环调用

ASP.NET Core 应用程序应该被设计为可以同时处理多个请求。异步 API 应该允许一个小的线程池处理上千个同时的请求，并且不用等待被锁住的请求。而不是等待一个长时间运行的任务去完成，这个线程可以处理其它的请求。

一个常见的性能问题是 ASP.NET Core 应用阻断了可以异步处理的请求。更多同步请求阻断问题可以移步 线程池饥饿，并且减少响应时间。

**不要做：**

- 使用 Task.wait 或者 Task<TResult>.Result 阻断异步程序的执行。

- 获取公共代码路径中的锁。ASP.NET Core 应用的架构在 parallel 上会有更好的性能。

- 调用 Task.Run，然后立即调用 await，ASP.NET Core 已经运行在普通的线程池内线程内运行该代码，于是调用 Task.Run 只会导致不必要的额外的线程池的调度。甚至这个调度可能会锁住该线程，Task.Run 不能阻止这样的状态发生

**应该这样：**

- 保证热代码是异步的

- 调用数据权限，输入/输出，或者长时间运行的 API 尽量使用异步的 API，不要使用 Task.Run 将一个同步的 API 转变为异步

- 使 Controller/Razor 页面的 Action 异步，这样整个调用堆栈都是异步的，以便从 async/await 模式中收益。

一些性能分析工具，如 PerfView，能够查看频繁的添加到线程池中的线程。Microsoft-Windows-DotNETRuntime/ThreadPoolWorkerThread/Start 事件标明了一个线程被添加到线程池中。

## 从多个小的页面返回一个大的集合

一个 web 页面不应该一次性加载大量的数据。当返回一个大集合的数据时，应该要考虑是否会引起性能问题。确定这种设计是否会引起以下问题：

- 内存溢出异常或高内存占用（消费）
- 线程池饥饿（查看下面的 IAsyncEnumerable<T>章节）
- 响应时间过长
- 频繁的垃圾收集处理

正确做法是添加分页来减轻上述情况。使用页面大小和页面索引参数，开发人员能够支持返回部分结果的设计。当必须输出一个详细的结果时，分页应该能够异步填充结果集以避免阻塞服务器资源。
更多的分页和限制返回结果集数量的信息，请查看：

- 性能参考

- 在 ASP.NET Core 程序内添加分页

## 返回 IEnumerable<T>或 IAsyncEnumerable<T>

使用序列化程序，从 action 的中返回的 IEnumerble<T>，这样做的结果会导致调用阻塞和潜在的线程池饥饿。为了避免同步的列举（enumeration），在返回 IEnumerable 类型前应使用 ToListAsync。
从 ASP.NET Core 3.0 开始，IAsyncEnumerable<T>可以替代 IEnumerable<T>实现异步的列举，更多信息，请查看控制器动作返回类型。

## 尽量减少大对象的分配

在 ASP.NET Core 中 GC 自动管理内存的回收和释放。自动的垃圾会搜通常意味着开发人员不需要考虑内存的释放，然而清除不需要的对象是需要消费 CPU 时间的，所以开发人员应该尽量减少对热路径下对象的分配。垃圾回收对大对象（大于 85K bytes）的清理尤其耗时（贵）。大对象应该保存在大对象堆（large object heap），然后需要一个 GC（第二代）去清理。不同于第 0 代和第 1 代的垃圾回收系统，第二代垃圾回收系统需要临时暂停程序的执行。对大对象频繁的分配和取消分配操作会导致不好的性能。

**建议：**

- 对频繁使用的大对象考虑是否可以缓存。缓存后就不会导致对大对象进行频繁的分配动作。
- 使用 ArraryPool<T>池缓冲区来保存大的数组。
- 在热代码路径内不要分配多的、生命周期很短的大对象。

内存问题，就像上面提到的这些，可以通过 PerfView 查看垃圾回收（GC）状态来诊断和详细检查：

- GC 暂停时间
- 垃圾回收时间占处理时间的百分比
- 使用了多少 GC，并且他们的是第几代的

更多信息，请参阅垃圾回收与性能

## 优化数据存储和输入输出（I/O）

在 ASP.NET Core 程序中，数据存储和与其他远程服务交互通常是最慢的部分。高效的数据读写对于良好的性能至关重要。

**建议：**

- 异步调用所有的数据读写接口
- 不要获取非必要的数据。编写查询应该只返回对本次 HTTP 请求必要的数据。
- 对频繁从数据库或远程服务获取的数据，如果这些数据可以接受在短时间内的过期，应该考虑对这些数据进行缓存。取决于具体场景，可以使用 MemoryCache 和 DistributionCache。更多信息，请参考在 ASP.NET Core 的响应缓存。

- 减少网络往返。尽量在一次请求中获取必要的数据，而不是多次请求。
- 在 EFCore 中，当查询的数据只是用来读的时候，应当使用不追踪的查询（no-tracking queries）。EF Core 能够更有效率的返回不追踪的查询对象。
- 过滤和聚合 Linq 查询（比如 where,Select,Sum），以便这些过滤由数据库来执行。
- 考虑 EFCore 中的有些查询是否在客户端（Client）执行，这些查询会拖慢查询性能，更多信息，请参考客户端评估性能问题。
- 不要在集合上使用投影查询，这样会导致类似“N+1”的 sql 查询，更多信息，请参考：相关子查询的优化。

参考 EF 高性能，了解更多在大规模应用中提升性能的方法：

- 共用 DbContext
- 精确编译的查询

我们建议在提交基础代码前横梁上诉高性能方法的影响。一些复杂的查询并不适合性能的提升。

可以通过应用程序洞察力或分析工具来判断应用程序花费在读取数据上的时间来检测查询问题。大部分数据库也提供了对于频繁执行的查询的统计信息。

## 使用 HttpClientFactory 管理 Http 连接池

尽管 HttpClient 实现了 IDisposable 接口，在设计时考虑服用。关闭 HttpClient 实例会让该实例打开的 socket 端口在 TIME_WAIT 状态等待一段时间。如果代码中频繁的创建和销毁 HttpClient 实例，则有可能会用尽所有可用的 socket 端口连接。HttpClientFactory 在 ASP.NET Core 2.1 中为了解决这个问题而添加的。它能共用 Http 连接以优化性能和提高可靠性。

**建议：**

- 不要直接创建和销毁 HttpClient 实例
- 使用 HttpClientFactory 获取 HttpClient 实例，更多信息，请参考 使用 HttpClientFacotry 实现可服用的 Http 请求

## 保持公共代码的快速的执行

你希望你所有的代码都快速执行。频繁调用的代码是最需要优化的，这些包括：

- 请求处理管道里的中间件，尤其是处于管道前面部分的。这些组件对性能有很大的影响。
- 每次请求都会运行一次或多次的代码。比如：定制化的日志、认证处理、或初始化一些临时的服务。

**建议：**

- 不要使用自定义长时间运行的中间件组件。
- 使用性能检测工具，比如：Visual Studio Diagnostic Tools, PerfView 来检查常用代码性能。

## 在 Http 请求之外完成长时间运行的任务

大部分到 ASP.NET Core 程序的请求会被一个 Controller 或页面模型调用必要的服务执行，并返回 Http 响应。有一些请求可能会包含长时间运行的任务，最好是把这种请求做一个整体的请求-响应异步进程。

**建议：**

- 不要在普通的 Http 请求中等待长时间运行的任务完成
- 考虑将长时间运行的请求使用后台服务或进程外的 Azure Function 实现。在进程外完成 CPU 密集型的任务特别有益。
- 使用实时交互，如 SignalR，与客户端进行异步通讯。

## 缩小客户端资源

ASP.NET Core 程序使用复杂的前端技术的会频繁调用很多 js，css 或图片文件。提升初始化加载的性能可以考虑：

- Bunding，将多个文件合并成一个。
- Minifying（缩小），去掉文件内的空格和注释，减少文件的大小

**建议：**

- 参考打包和压缩引导，介绍了兼容的工具，并且展示了如何利用 ASP.NET Core 环境标签来处理开发和产品环境信息。
- 考虑第三方的一些工具，如 Webpack，管理客户端的资源。

## 压缩 Http 响应

减少响应的大小会非常显著的提升程序的响应性。一个方法就是减少响应负荷大小的方法就是压缩响应，更多详细信息，请参考压缩响应。

## 最小化异常

异常应该是罕见的。与其它代码流模式相比，抛出和捕捉异常是一个相对较慢的模式。基于此，异常不应该被用来控制正常的程序逻辑。

**建议：**

- 不要使用抛出和捕获异常来作为保证程序正常流程的一种手段，尤其是在常用代码中。
- 在程序中应包含 对可能触发的异常的检测和处理。
- 为意外或异常情况抛出或捕获异常。
  程序诊断工具，如 Application Insights，可以帮助检查程序可能影响性能的常见异常。

# 性能和可靠性

以下部分提供了性能提示和已知的可靠性问题及解决方案

## 在 HttpRequest 和 HttpRespose body 中避免同步读写

在 ASP.NET Core 所有的 I/O 操作都是异步的。服务实现了 Stream 接口，该接口有同步和异步方法的的重载。使用异步方法可以避免阻塞线程池线程。线程的阻塞可能会导致线程池饥饿。

**不要这样做：** 下面的示例使用了 ReadToEnd 犯法，阻塞了当前线程等待结果，这是一个同步异步的例子：

```csharp
public class BadStreamReadController:Controller
{
   [HttpGet("/contoso")]
   public ActionResult<ContosoData> Get()
   {
      var json=new StreamReader(Request.Body).ReadToEnd();
      return JsonSerializer.Deserialize<ContosoData>(json);
   }
}
```

在上面的例子中，同步读取整个请求体的内容到内存中。如果客户端上传比较慢，程序正在通过同步实现异步。程序通过同步实现异步是由于 Kestrel 不支持同步的读取。

**这样做：** 下面的例子使用了 ReadToEndAsync ，并且不会阻断线程的读取：

```csharp
public class BadStreamReadController:Controller
{
   [HttpGet("/contoso")]
   public async Task<ActionResult<ContosoData>> Get()
   {
      var json=await new StreamReader(Request.Body).ReadToEndAsync();
      return JsonSerializer.Deserialize<ContosoData>(json);
   }
}
```

在上面的例子中异步读取了整个 HTTP 请求的内容到内存中。

> 警告
> 如果请求过大，读取整个请求内容到内存中可能会导致内存溢出。内存溢出会导致服务崩溃。更多的信息，请参考[避免读取大报文或相应到内存中](#避免读取大报文或相应到内存中)

**这样做：** 下面是一个完整的不使用缓冲区的异步读取的例子

```csharp
public class GoodStreamReaderController : Controller
{
 [HttpGet("/contoso")]
 public async Task<ActionResult<ContosoData>> Get()
 {
    return await JsonSerializer.DeserializeAsync<ContosoData>(Request.Body);
 }
}
```

上面的例子展示了异步反序列化 RequestBody 到 C#对象中。

## 首选 ReadFormAsync 而不是 Request.Form

用 HttpContext.Request.ReadFormAsync 替代 HttpContext.Request.Form
HttpContext.Reqeust.Form 只有在下列情况才能安全读取：

- fom 已经被 ReadFormAsync 读取过
- 缓存的 form 值已经用 HttpContext.Request.Form 读取过

**不要这样做:** 下面的例子使用 HttpContext.Request.Form。HttpContext.Request.Form 导致了同步替代异步，这样可能会导致线程池饥饿。

```csharp
public class BadReadController : Controller
{
    [HttpPost("/form-body")]
    public IActionResult Post()
    {
        var form = HttpContext.Request.Form;
        Process(form["id"],form["name"]);
        return Accepted();
    }
}
```

**这样做:**下面的例子使用 HttpContext.Request.ReadFormAsync 异步读取 form Body

```csharp
public class BadReadController : Controller
{
    [HttpPost("/form-body")]
    public async Task<IActionResult> Post()
    {
        var form =await HttpContext.Request.ReadFormAsync();
        Process(form["id"],form["name"]);
        return Accepted();
    }
}
```

## 避免读取大报文或相应到内存中

在.NET 中，任何大于 85KB 的对象都会存放在大对象堆（large object heap）中,大对象在两方面开销大：

- 分配代价高，因为必须为新分配的大对象清理内存。CLR 需要保证为所有大对象分配的内存都是干净的。
- 大对象堆是一个集合，大对象取药一个完整的垃圾回收或第二代的回收。

下面是一个博客简单描述了发生的问题

> 当分配一个大对象时，它会被标记为第二代对象，不是 0 代的小对象。这样的后果就是当大对象堆出现内存溢出移仓时，他会清理所有管理的对，不仅仅是大对象堆。所以，他清理了第 0 代、第 1 代和第 2 代的数据，包含大对象堆，这被叫做完全垃圾回收，并且也是最耗时的垃圾回收方法。对很多程序来说，这是可以接受的。但绝对不是针对高性能的 web 服务，这些服务有很大的内存缓冲被需要去处理 web 请求（从 socket 读取数据、解压缩、decode json 等等）。

天真的将一个大的请求或响应存储到一个 byte 数组或字符串中：

- 很快在大对象堆中导致空间溢出
- 由于完全的 gc 运行，导致程序的性能问题

## API 处理同步数据

当使用序列化/反序列化时，只支持同步的读写（比如：Json.net）

- 在将数据传送给序列化/反序列化方法时，必须先将数据保存到内存中

> 警告
> 如果请求的报文很大，有可能导致内存溢出，内存溢出有可能会导致拒绝服务（服务崩溃）。更多信息，请查看本文中的[避免读取大报文或相应到内存中](#避免读取大报文或相应到内存中)
> ASP.NET Core 3.0 默认使用 System.Text.Json 进行序列化。
> [System.Text.Json](#https://learn.microsoft.com/en-us/dotnet/api/system.text.json)

- 对 json 的异步读写
- 对 utf-8 格式的文本进行了优化
- 比 Newtonsoft.Json 通常性能更好

## 自己不要保存 IHttpContextAccessor.HttpContext

IHttpContextAccessor.HttpContext 会返回当前访问线程的 HttpContext。IHttpContextAccessor.HttpContext 不应当保存在字段或变量中。

### 不要这样做

下面的列子将 HttpContext 保存在了属性中，并且试图在后面使用它。

```csharp
public class MyBadType
{
   private readonly HttpContext _context;
   public MyBadType(IHttpContextAccessor accssor)
   {
      _context=accssor.HttpContext;
   }

   public void CheckAdmin()
   {
      if(!_context.User.IsInRole("admin"))
      {
         throw new UnauthorizedAccessException("The current user isn't an admin");
      }
   }
}
```

上面的代码在构造函数中频繁的获取一个 null 或不正确的 HttpContext

### 这样做

下面的列子

- 将 IHttpContextAccessor 保存在字段中
- 在正确的时间点使用 HttpContext，并检查它是否为 null

```csharp
public class MyGoodType
{
   private readonly IHttpContextAccessor _accessor;
   public MyGoodType(IHttpContextAccessor accessor){
      _accessor=accessor;
   }

   public void CheckAdmin(){
      var context=_accessor.HttpContext;
      if(context!=null&&!context.User.IsInRole("admin"))
      {
         throw new UnauthorizedAccessException("The current user isn't an admin");
      }
   }
}
```

## 不要从多个线程访问 HttpContext

HttpContext 不是线程安全的，使用 Parallel 从多个线程访问 HttpContext 有可能会导致一些莫名其妙的问题，比如：程序悬停（hangs），崩溃，数据污染

### 不要这样做

下面的例子用三个 Parallel 请求，并且在 HttpRequest 进入和出去前记录了日志。访问路径在多个线程内被访问

```csharp
public class AsyncBadSearchController : Controller
{
   [HttpGet("/search")]
   public async Task<SearchResults> Get(string query)
   {
      var query1 = SearchAsync(SearchEngine.Google, query);
      var query2 = SearchAsync(SearchEngine.Bing, query);
      var query3 = SearchAsync(SearchEngine.DuckDuckGo, query);
      await Task.WhenAll(query1, query2, query3);
      var results1 = await query1;
      var results2 = await query2;
      var results3 = await query3;
      return SearchResults.Combine(results1, results2, results3);
   }
   private async Task<SearchResults> SearchAsync(SearchEngine engine,string query)
   {
      var searchResults = _searchService.Empty();
      try
      {
         _logger.LogInformation("Starting search query from {path}.",HttpContext.Request.Path);
         searchResults = _searchService.Search(engine, query);
         _logger.LogInformation("Finishing search query from {path}.",HttpContext.Request.Path);
      }
      catch (Exception ex)
      {
         _logger.LogError(ex, "Failed query from {path}",HttpContext.Request.Path);
      }
      return await searchResults;
   }
```

### 这样做

下面的例子在访问三个同步 parallel 请求时将所有的数据从请求中拷贝到变量中

```csharp
public class AsyncGoodSearchController : Controller
{
   [HttpGet("/search")]
   public async Task<SearchResults> Get(string query)
   {
      string path = HttpContext.Request.Path;
      var query1 = SearchAsync(SearchEngine.Google, query,path);
      var query2 = SearchAsync(SearchEngine.Bing, query, path);
      var query3 = SearchAsync(SearchEngine.DuckDuckGo, query, path);
      await Task.WhenAll(query1, query2, query3);
      var results1 = await query1;
      var results2 = await query2;
      var results3 = await query3;
      return SearchResults.Combine(results1, results2, results3);
   }
   private async Task<SearchResults> SearchAsync(SearchEngine engine,string query,string path)
   {
      var searchResults = _searchService.Empty();
      try
      {
         _logger.LogInformation("Starting search query from {path}.",path);
         searchResults = await _searchService.SearchAsync(engine, query);
         _logger.LogInformation("Finishing search query from {path}.",path);
      }
      catch (Exception ex)
      {
         _logger.LogError(ex, "Failed query from {path}", path);
      }
   return await searchResults;
}
```

## Request 已经结束后不要在使用 HttpContext

HttpContext 只有在当前 Http 请求在 ASP.NET Core 请求管道中才有效。整个的 ASP.NET Core 管道时一个同步的委托链，在这些委托处理每一个请求。当任务从这些委托链中完成时，HttpContext 就被回收了。

### 不要这样做

下面的例子使用了同步无返回值的方法，在该方法内执行第一个 await 语句时让 Http 请求完成

- 这在 ASP.NET Core 中总是一个不好的做法
- 在 Http 请求结束后访问 HttpResponse
- 进程崩溃

```csharp
public class AsyncBadVoidController : Controller
{
   [HttpGet("/async")]
   public async void Get()
   {
      await Task.Delay(1000);
      // The following line will crash the process. because of writing after the response has completed on a background thread. Notice async void Get()
      await Response.WriteAsync("Hello World");
   }
}
```

### 这样做

下面的例子展示了返回一个 Task 给框架，因此 Http 请求在 Action 完成前不会结束。

```csharp
 public class AsyncGoodTaskController : Controller
{
   [HttpGet("/async")]
   public async Task Get()
   {
      await Task.Delay(1000);
      await Response.WriteAsync("Hello World");
   }
}
```
