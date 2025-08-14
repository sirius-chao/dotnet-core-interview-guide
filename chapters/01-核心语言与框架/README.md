# 1. 核心语言与框架

## 1.1 C# 高级特性

### 异步编程 (async/await)

#### 核心概念
- **异步编程模型**：基于 Task 的异步编程模式
- **Task 和 Task<T> 的区别**：Task 表示无返回值的异步操作，Task<T> 表示有返回值的异步操作
- **ConfigureAwait(false) 的作用**：避免死锁，提高性能，不强制回到原始上下文
- **异步方法的异常处理**：使用 try-catch 包装 await 调用
- **取消令牌 (CancellationToken)**：支持取消长时间运行的异步操作

#### 面试重点
```csharp
// 异步方法示例
public async Task<string> GetDataAsync(CancellationToken cancellationToken = default)
{
    try
    {
        using var httpClient = new HttpClient();
        var response = await httpClient.GetAsync("https://api.example.com/data", cancellationToken);
        return await response.Content.ReadAsStringAsync();
    }
    catch (OperationCanceledException)
    {
        // 处理取消操作
        return string.Empty;
    }
}

// ConfigureAwait(false) 使用
await GetDataAsync().ConfigureAwait(false);
```

### LINQ 高级用法

#### 延迟执行 vs 立即执行
- **延迟执行**：Where、Select、OrderBy 等操作不会立即执行，只有在枚举时才执行
- **立即执行**：ToList、ToArray、Count 等操作会立即执行并返回结果

#### 自定义扩展方法
```csharp
public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereIf<T>(this IEnumerable<T> source, bool condition, Func<T, bool> predicate)
    {
        return condition ? source.Where(predicate) : source;
    }
}
```

#### 复杂查询优化
- 避免 N+1 查询问题
- 使用 AsNoTracking() 提高只读查询性能
- 合理使用投影减少数据传输

#### PLINQ 并行查询
```csharp
var result = source.AsParallel()
    .Where(x => x.IsValid)
    .Select(x => x.Process())
    .ToList();
```

### 泛型与约束

#### 泛型约束类型
```csharp
// 接口约束
public class Repository<T> where T : IEntity

// 基类约束
public class Service<T> where T : BaseEntity

// 构造函数约束
public class Factory<T> where T : new()

// 值类型约束
public class Cache<T> where T : struct

// 引用类型约束
public class Handler<T> where T : class
```

#### 协变和逆变
```csharp
// 协变 (out)
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // 协变

// 逆变 (in)
Action<object> objectAction = obj => Console.WriteLine(obj);
Action<string> stringAction = objectAction; // 逆变
```

### 反射与元数据

#### Type 类的使用
```csharp
var type = typeof(MyClass);
var properties = type.GetProperties();
var methods = type.GetMethods();
var attributes = type.GetCustomAttributes();
```

#### 动态创建对象
```csharp
var instance = Activator.CreateInstance<MyClass>();
var instance2 = Activator.CreateInstance(type, "constructor parameter");
```

#### 性能考虑
- 反射操作性能较低，避免在循环中使用
- 考虑使用表达式树或代码生成替代反射
- 缓存反射结果提高性能

### 表达式树 (Expression Trees)

#### 表达式树构建
```csharp
Expression<Func<Person, bool>> expression = p => p.Age > 18 && p.Name.StartsWith("A");
```

#### 动态查询构建
```csharp
public static IQueryable<T> WhereDynamic<T>(this IQueryable<T> source, string propertyName, object value)
{
    var parameter = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(parameter, propertyName);
    var constant = Expression.Constant(value);
    var comparison = Expression.GreaterThan(property, constant);
    var lambda = Expression.Lambda<Func<T, bool>>(comparison, parameter);
    
    return source.Where(lambda);
}
```

### 模式匹配 (Pattern Matching)

#### switch 表达式
```csharp
var result = obj switch
{
    string s => $"String: {s}",
    int i => $"Integer: {i}",
    Person p => $"Person: {p.Name}",
    _ => "Unknown type"
};
```

#### 类型模式
```csharp
if (obj is string s)
{
    Console.WriteLine($"String length: {s.Length}");
}
```

#### 属性模式
```csharp
if (person is { Age: > 18, Name: { Length: > 0 } })
{
    Console.WriteLine("Valid adult person");
}
```

## 1.2 .NET Core 框架

### 依赖注入 (DI) 容器

#### 服务生命周期
- **Singleton**：整个应用程序生命周期内只有一个实例
- **Scoped**：每个请求范围内有一个实例
- **Transient**：每次请求都创建新实例

#### 服务注册方式
```csharp
// 在 Program.cs 或 Startup.cs 中
services.AddSingleton<IMyService, MyService>();
services.AddScoped<IRepository, Repository>();
services.AddTransient<IHelper, Helper>();

// 工厂模式注册
services.AddSingleton<IMyService>(sp => new MyService(sp.GetRequiredService<ILogger>()));
```

#### 构造函数注入 vs 属性注入
```csharp
// 构造函数注入（推荐）
public class MyController
{
    private readonly IMyService _service;
    
    public MyController(IMyService service)
    {
        _service = service;
    }
}

// 属性注入
public class MyController
{
    [Inject]
    public IMyService Service { get; set; }
}
```

### 配置系统

#### 配置源优先级
1. 命令行参数
2. 环境变量
3. appsettings.{Environment}.json
4. appsettings.json
5. 默认值

#### 强类型配置绑定
```csharp
public class AppSettings
{
    public string ConnectionString { get; set; }
    public int MaxRetryCount { get; set; }
}

// 在 Program.cs 中
builder.Configuration.GetSection("App").Bind(new AppSettings());
// 或者
builder.Services.Configure<AppSettings>(builder.Configuration.GetSection("App"));
```

#### 配置验证
```csharp
public class AppSettings
{
    [Required]
    public string ConnectionString { get; set; }
    
    [Range(1, 10)]
    public int MaxRetryCount { get; set; }
}

// 验证配置
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("App"))
    .ValidateDataAnnotations();
```

### 日志框架

#### ILogger 接口使用
```csharp
public class MyService
{
    private readonly ILogger<MyService> _logger;
    
    public MyService(ILogger<MyService> logger)
    {
        _logger = logger;
    }
    
    public void DoWork()
    {
        _logger.LogInformation("Starting work");
        try
        {
            // 执行工作
            _logger.LogInformation("Work completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error occurred while doing work");
        }
    }
}
```

#### 结构化日志
```csharp
_logger.LogInformation("User {UserId} logged in from {IPAddress}", userId, ipAddress);
```

#### 日志级别配置
```csharp
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  }
}
```

### 中间件管道

#### 中间件执行顺序
```csharp
app.UseExceptionHandler("/Error");
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

#### 自定义中间件开发
```csharp
public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CustomMiddleware> _logger;
    
    public CustomMiddleware(RequestDelegate next, ILogger<CustomMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation("Request started: {Path}", context.Request.Path);
        
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        
        _logger.LogInformation("Request completed in {ElapsedMs}ms", sw.ElapsedMilliseconds);
    }
}

// 扩展方法
public static class CustomMiddlewareExtensions
{
    public static IApplicationBuilder UseCustomMiddleware(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<CustomMiddleware>();
    }
}
```

#### 中间件短路
```csharp
public async Task InvokeAsync(HttpContext context)
{
    if (context.Request.Path.StartsWithSegments("/api"))
    {
        // 短路，不调用下一个中间件
        context.Response.StatusCode = 404;
        return;
    }
    
    await _next(context);
}
```

### 生命周期管理

#### IHostedService 接口
```csharp
public class BackgroundService : IHostedService
{
    private Timer _timer;
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }
    
    private void DoWork(object state)
    {
        // 执行后台工作
    }
}

// 注册服务
services.AddHostedService<BackgroundService>();
```

#### 优雅关闭
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();
        
        // 注册关闭事件处理
        var lifetime = host.Services.GetRequiredService<IHostApplicationLifetime>();
        lifetime.ApplicationStopping.Register(() =>
        {
            // 执行清理工作
        });
        
        await host.RunAsync();
    }
}
```

## 面试重点总结

### 高频面试题
1. **async/await 的工作原理是什么？**
2. **Task 和 Thread 的区别？**
3. **ConfigureAwait(false) 的作用？**
4. **依赖注入的生命周期有哪些？**
5. **中间件的执行顺序如何控制？**

### 代码示例准备
- 异步方法的正确实现
- 自定义中间件的开发
- 依赖注入的配置
- 表达式树的构建
- 模式匹配的使用

### 性能优化要点
- 避免 async void
- 合理使用 ConfigureAwait(false)
- 缓存反射结果
- 避免在循环中使用 LINQ
- 合理选择服务生命周期
