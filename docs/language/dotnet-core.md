# .NET Core 框架

## 1. 依赖注入 (Dependency Injection)

### 1.1 服务生命周期
```csharp
// 在 Program.cs 或 Startup.cs 中
services.AddSingleton<IMyService, MyService>();        // 单例，整个应用生命周期
services.AddScoped<IRepository, Repository>();         // 作用域，每个请求一个实例
services.AddTransient<IHelper, Helper>();              // 瞬时，每次请求都创建新实例

// 工厂模式注册
services.AddSingleton<IMyService>(sp => new MyService(sp.GetRequiredService<ILogger>()));
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
