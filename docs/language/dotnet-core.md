# .NET Core 框架

## 1. 依赖注入 (Dependency Injection)

**依赖注入的设计哲学**
依赖注入（Dependency Injection，DI）是一种设计模式，它实现了控制反转（Inversion of Control，IoC）原则。通过DI容器，应用程序的组件不再负责创建和管理其依赖对象，而是由容器负责提供这些依赖。

**DI的核心价值**：
1. **松耦合**：组件之间通过接口交互，降低直接依赖
2. **可测试性**：可以轻松替换依赖，便于单元测试
3. **可维护性**：依赖关系清晰，便于理解和修改
4. **可扩展性**：可以轻松添加新的实现或装饰器
5. **生命周期管理**：容器自动管理对象的创建和销毁

**服务生命周期的选择策略**：
- **Singleton**：适用于无状态服务，如配置服务、日志服务
- **Scoped**：适用于有状态服务，如数据库上下文、业务服务
- **Transient**：适用于轻量级服务，如工具类、辅助服务

**生命周期详解**：

**Scoped（作用域）**：
- **一个请求作用域内**：同一个对象实例
- **跨方法共享**：在同一个 HTTP 请求中，无论经过多少个方法调用，注入的都是同一个对象实例
- **生命周期**：从请求开始到请求结束

**Transient（瞬时）**：
- **每次注入都创建新实例**：每次从容器中获取服务时，都会创建一个全新的对象实例
- **方法间独立**：即使在同一个请求中，不同方法注入的也是不同的对象实例

### 1.1 服务生命周期
```csharp
// 在 Program.cs 或 Startup.cs 中
services.AddSingleton<IMyService, MyService>();        // 单例，整个应用生命周期
services.AddScoped<IRepository, Repository>();         // 作用域，每个请求一个实例
services.AddTransient<IHelper, Helper>();              // 瞬时，每次请求都创建新实例

// 工厂模式注册
services.AddSingleton<IMyService>(sp => new MyService(sp.GetRequiredService<ILogger>()));

**生命周期区别示例**：
```csharp
// 注册服务
services.AddScoped<IMyService, MyService>();      // 作用域
services.AddTransient<IHelper, Helper>();         // 瞬时

// 控制器
public class MyController : ControllerBase
{
    private readonly IMyService _scopedService;    // 作用域服务
    private readonly IHelper _transientHelper;     // 瞬时服务
    
    public MyController(IMyService scopedService, IHelper transientHelper)
    {
        _scopedService = scopedService;
        _transientHelper = transientHelper;
    }
    
    [HttpGet]
    public IActionResult Get()
    {
        // 调用业务服务
        var result = _businessService.Process(_scopedService, _transientHelper);
        return Ok(result);
    }
}

// 业务服务
public class BusinessService
{
    private readonly IMyService _scopedService;    // 作用域服务
    private readonly IHelper _transientHelper;     // 瞬时服务
    
    public BusinessService(IMyService scopedService, IHelper transientHelper)
    {
        _scopedService = scopedService;
        _transientHelper = transientHelper;
    }
    
    public string Process(IMyService scopedParam, IHelper transientParam)
    {
        // 在同一个请求中：
        // _scopedService 和 scopedParam 是同一个对象实例
        // _transientHelper 和 transientParam 是不同的对象实例
        
        return "Processed";
    }
}
```

### 1.2 服务注册最佳实践
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

// 服务定位器（不推荐，但有时必要）
public class MyService
{
    private readonly IServiceProvider _serviceProvider;
    
    public MyService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public void DoWork()
    {
        using var scope = _serviceProvider.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<ITempService>();
        // 使用服务
    }
}
```

### 1.3 高级注册模式
```csharp
// 条件注册
if (environment.IsDevelopment())
{
    services.AddScoped<IDataService, MockDataService>();
}
else
{
    services.AddScoped<IDataService, RealDataService>();
}

// 装饰器模式
services.AddScoped<IDataService, RealDataService>();
services.Decorate<IDataService, CachingDataService>();
services.Decorate<IDataService, LoggingDataService>();

// 命名服务
services.AddKeyedSingleton<ICache>("UserCache", new UserCache());
services.AddKeyedSingleton<ICache>("ProductCache", new ProductCache());
```

## 2. 配置系统

**配置系统的设计理念**
.NET Core的配置系统提供了一个统一的、类型安全的配置管理方案。它支持多种配置源，支持配置热重载，并且可以轻松扩展。

**配置系统的核心特性**：
1. **多源支持**：支持JSON、环境变量、命令行参数、用户密钥等多种配置源
2. **层次化配置**：支持嵌套配置，便于组织复杂配置
3. **类型安全**：支持强类型配置绑定，编译时检查
4. **热重载**：支持运行时配置更新，无需重启应用
5. **验证支持**：支持配置验证，确保配置的正确性

**配置源优先级**（从高到低）：
- 命令行参数
- 环境变量
- 用户密钥（仅开发环境）
- appsettings.{Environment}.json
- appsettings.json
- 默认值

### 2.1 配置源
```csharp
var builder = WebApplication.CreateBuilder(args);

// 添加配置源
builder.Configuration
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables()
    .AddCommandLine(args)
    .AddUserSecrets<Program>()
    .AddAzureKeyVault("https://your-vault.vault.azure.net/");
```

### 2.2 强类型配置
```csharp
public class AppSettings
{
    public string ConnectionString { get; set; }
    public int MaxRetryCount { get; set; }
    public LoggingSettings Logging { get; set; }
}

public class LoggingSettings
{
    public string Level { get; set; }
    public bool EnableConsole { get; set; }
}

// 在 Program.cs 中
builder.Configuration.GetSection("App").Bind(new AppSettings());
// 或者
builder.Services.Configure<AppSettings>(builder.Configuration.GetSection("App"));
```

### 2.3 配置验证
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

// 自定义验证
builder.Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection("App"))
    .Validate(settings =>
    {
        if (string.IsNullOrEmpty(settings.ConnectionString))
            return false;
        return true;
    }, "Connection string is required");
```

### 2.4 配置热重载
```csharp
// 监听配置变化
builder.Services.Configure<AppSettings>(builder.Configuration.GetSection("App"));
builder.Services.AddSingleton<IOptionsMonitor<AppSettings>>();

// 在服务中使用
public class MyService
{
    private readonly IOptionsMonitor<AppSettings> _options;
    
    public MyService(IOptionsMonitor<AppSettings> options)
    {
        _options = options;
        
        // 监听配置变化
        _options.OnChange(settings =>
        {
            // 处理配置变化
            Console.WriteLine($"Connection string changed to: {settings.ConnectionString}");
        });
    }
}
```

## 3. 日志系统

**日志系统的设计原则**
.NET Core的日志系统提供了一个统一的日志抽象，支持多种日志提供程序，并且对性能影响最小。它支持结构化日志、日志级别、日志作用域等高级特性。

**日志系统的核心价值**：
1. **统一抽象**：提供统一的ILogger接口，支持多种日志提供程序
2. **结构化日志**：支持结构化数据，便于日志分析和查询
3. **性能优化**：使用延迟评估和条件编译，最小化性能影响
4. **可扩展性**：支持自定义日志提供程序和格式化器
5. **配置灵活**：支持运行时配置日志级别和输出目标

**日志级别的最佳实践**：
- **Trace**：详细的调试信息，仅在开发环境使用
- **Debug**：调试信息，帮助开发人员理解程序流程
- **Information**：一般信息，记录重要的业务事件
- **Warning**：警告信息，表示潜在问题但不影响功能
- **Error**：错误信息，表示功能失败但系统仍可运行
- **Critical**：严重错误，表示系统级故障

### 3.1 基本使用
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

### 3.2 结构化日志
```csharp
// 使用占位符
_logger.LogInformation("User {UserId} logged in from {IPAddress}", userId, ipAddress);

// 使用范围
using (_logger.BeginScope("Processing user {UserId}", userId))
{
    _logger.LogInformation("Starting user processing");
    // 处理逻辑
    _logger.LogInformation("User processing completed");
}
```

### 3.3 日志配置
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "Console": {
      "LogLevel": {
        "Default": "Information"
      }
    },
    "File": {
      "Path": "logs/app-{Date}.txt",
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}
```

### 3.4 自定义日志提供程序
```csharp
public class CustomLoggerProvider : ILoggerProvider
{
    public ILogger CreateLogger(string categoryName)
    {
        return new CustomLogger(categoryName);
    }

    public void Dispose() { }
}

public class CustomLogger : ILogger
{
    private readonly string _categoryName;

    public CustomLogger(string categoryName)
    {
        _categoryName = categoryName;
    }

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter)
    {
        // 自定义日志逻辑
        var message = formatter(state, exception);
        Console.WriteLine($"[{logLevel}] {_categoryName}: {message}");
    }

    public bool IsEnabled(LogLevel logLevel) => true;
    public IDisposable BeginScope<TState>(TState state) => null;
}

// 注册自定义日志提供程序
builder.Logging.AddProvider(new CustomLoggerProvider());
```

## 4. 中间件管道

**中间件管道的设计理念**
中间件管道是ASP.NET Core的核心概念，它允许请求在到达应用程序之前和响应返回客户端之前经过一系列组件。每个中间件都可以处理请求、修改响应，或者决定是否将请求传递给下一个中间件。

**中间件管道的核心特性**：
1. **管道模式**：请求按顺序流经每个中间件，形成处理管道
2. **短路能力**：中间件可以决定不调用下一个中间件，直接返回响应
3. **双向处理**：中间件可以在请求处理前后执行逻辑
4. **可组合性**：中间件可以灵活组合，实现不同的处理流程
5. **性能优化**：支持条件中间件，避免不必要的处理

**中间件执行顺序的重要性**：
- **异常处理**：异常处理中间件应该放在最前面
- **认证授权**：认证应该在授权之前执行
- **路由**：路由中间件应该在端点执行之前
- **静态文件**：静态文件中间件应该在路由之前
- **CORS**：CORS中间件应该在认证之前

### 4.1 中间件顺序
```csharp
var app = builder.Build();

app.UseExceptionHandler("/Error");
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

### 4.2 自定义中间件
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

### 4.3 中间件短路
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

### 4.4 条件中间件
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
```

## 5. 生命周期管理

**应用程序生命周期的管理策略**
.NET Core提供了完整的应用程序生命周期管理机制，包括启动、运行、停止和关闭等阶段。通过生命周期管理，应用程序可以优雅地处理启动和关闭过程。

**生命周期管理的核心价值**：
1. **优雅启动**：应用程序启动时执行必要的初始化工作
2. **后台服务**：支持长时间运行的后台任务
3. **优雅关闭**：应用程序关闭时执行清理工作
4. **资源管理**：确保资源在应用程序生命周期内正确管理
5. **状态监控**：监控应用程序的运行状态

**生命周期阶段的处理策略**：
- **启动阶段**：配置服务、初始化资源、启动后台服务
- **运行阶段**：处理请求、执行后台任务、监控系统状态
- **停止阶段**：停止接受新请求、完成正在处理的请求
- **关闭阶段**：释放资源、保存状态、关闭连接

### 5.1 IHostedService
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

### 5.2 BackgroundService 基类
```csharp
public class MyBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // 正常取消
                break;
            }
            catch (Exception ex)
            {
                // 记录错误并继续
                _logger.LogError(ex, "Error in background service");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
        }
    }

    private async Task DoWorkAsync(CancellationToken stoppingToken)
    {
        // 执行工作
    }
}
```

### 5.3 优雅关闭
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
            Console.WriteLine("Application is stopping...");
        });

        lifetime.ApplicationStopped.Register(() =>
        {
            // 应用已完全停止
            Console.WriteLine("Application has stopped");
        });

        await host.RunAsync();
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **依赖注入生命周期**：Singleton、Scoped、Transient的区别和选择
2. **配置系统**：多环境配置、热重载、验证
3. **中间件管道**：执行顺序、短路、自定义中间件
4. **日志系统**：结构化日志、性能影响、自定义提供程序
5. **生命周期管理**：后台服务、优雅关闭

### 6.2 代码示例准备
- 自定义中间件的实现
- 依赖注入的高级用法
- 配置验证和热重载
- 后台服务的实现
- 优雅关闭的处理

### 6.3 性能优化要点
- 合理选择服务生命周期
- 避免在中间件中执行耗时操作
- 使用结构化日志提高查询效率
- 合理配置日志级别
- 后台服务的异常处理和重试机制
