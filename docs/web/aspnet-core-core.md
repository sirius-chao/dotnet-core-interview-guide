# ASP.NET Core 核心概念

## 📚 目录导航

- [中间件管道](#middleware-pipeline)
- [依赖注入系统](#dependency-injection)
- [配置系统](#configuration-system)
- [生命周期管理](#lifecycle-management)
- [路由系统](#routing-system)
- [日志系统](#logging-system)
- [面试常见问题](#interview-questions)

## 🎯 学习目标

深入理解 ASP.NET Core 的核心概念和底层原理，掌握面试中的高频考点：

- **中间件管道**: 理解请求处理流程和中间件设计模式
- **依赖注入**: 掌握服务生命周期和容器管理机制
- **配置系统**: 了解多源配置和热重载原理
- **生命周期**: 理解应用程序和请求的生命周期管理
- **路由系统**: 掌握路由匹配和端点选择机制
- **日志系统**: 了解结构化日志和性能优化

## 🚀 中间件管道 {#middleware-pipeline}

### 中间件管道的设计哲学

ASP.NET Core 的中间件管道基于责任链模式（Chain of Responsibility Pattern）设计，每个中间件都可以：
- 处理请求
- 调用下一个中间件
- 处理响应
- 决定是否短路请求

**管道的核心机制**：
1. **RequestDelegate 委托链**：每个中间件都是一个 RequestDelegate，通过 `_next(context)` 调用下一个中间件
2. **Use 扩展方法**：将中间件添加到管道中，形成委托链
3. **短路机制**：中间件可以选择不调用 `_next`，直接返回响应
4. **双向处理**：请求和响应都可以被中间件处理

### 中间件的执行顺序原理

**性能优化的中间件顺序**：
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // 1. 异常处理 - 必须放在最前面，捕获后续中间件的异常
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }

    // 2. HTTPS 重定向 - 在路由之前，确保安全连接
    app.UseHttpsRedirection();

    // 3. 静态文件 - 在路由之前，避免不必要的路由计算
    app.UseStaticFiles();

    // 4. 路由 - 确定请求的端点
    app.UseRouting();

    // 5. 认证授权 - 在端点执行之前验证用户身份
    app.UseAuthentication();
    app.UseAuthorization();

    // 6. 业务中间件 - 自定义业务逻辑
    app.UseCustomMiddleware();

    // 7. 端点 - 执行具体的业务逻辑
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

**为什么这个顺序很重要？**
- **异常处理**：必须最先执行，确保能捕获所有异常
- **HTTPS重定向**：在路由之前，避免重复重定向
- **静态文件**：在路由之前，提高性能
- **路由**：确定端点，避免不必要的处理
- **认证授权**：在业务逻辑之前验证身份
- **业务中间件**：处理具体的业务需求

### 自定义中间件实现

**性能监控中间件**：
```csharp
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMiddleware> _logger;

    public PerformanceMiddleware(RequestDelegate next, ILogger<PerformanceMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            var elapsed = stopwatch.ElapsedMilliseconds;
            
            if (elapsed > 1000) // 超过1秒记录警告
            {
                _logger.LogWarning("Request to {Path} took {Elapsed}ms", 
                    context.Request.Path, elapsed);
            }
            
            _logger.LogInformation("Request to {Path} completed in {Elapsed}ms", 
                context.Request.Path, elapsed);
        }
    }
}

// 扩展方法
public static class PerformanceMiddlewareExtensions
{
    public static IApplicationBuilder UsePerformanceMonitoring(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<PerformanceMiddleware>();
    }
}
```

**条件中间件**：
```csharp
public class ConditionalMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public ConditionalMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 根据配置决定是否启用某些功能
        if (_configuration.GetValue<bool>("EnableFeatureX"))
        {
            // 启用特性X的逻辑
            context.Response.Headers.Add("X-Feature-Enabled", "true");
        }

        await _next(context);
    }
}
```

## 🔧 依赖注入系统 {#dependency-injection}

### DI 容器的内部实现机制

ASP.NET Core 的 DI 容器基于 Microsoft.Extensions.DependencyInjection，其核心原理包括：

**服务注册机制**：
1. **服务描述符（ServiceDescriptor）**：包含服务类型、实现类型、生命周期等信息
2. **服务集合（IServiceCollection）**：管理所有服务描述符
3. **服务提供者（IServiceProvider）**：负责创建和管理服务实例

**生命周期管理的底层实现**：
- **Singleton**：在根容器中创建，整个应用程序生命周期内共享
- **Scoped**：在每个作用域内创建，作用域结束时释放
- **Transient**：每次请求都创建新实例，不缓存

### 服务注册最佳实践

**正确的服务注册方式**：
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 1. 基础服务
    services.AddControllers();
    services.AddDbContext<ApplicationDbContext>();
    
    // 2. 业务服务 - 使用接口注册
    services.AddScoped<IUserService, UserService>();
    services.AddScoped<IOrderService, OrderService>();
    
    // 3. 工具服务 - 无状态服务使用Transient
    services.AddTransient<IEmailService, EmailService>();
    services.AddTransient<ILogger, Logger>();
    
    // 4. 配置服务 - 使用选项模式
    services.Configure<DatabaseSettings>(
        Configuration.GetSection("Database"));
    
    // 5. 单例服务 - 谨慎使用，确保线程安全
    services.AddSingleton<ICacheService, CacheService>();
}
```

**避免的常见陷阱**：
```csharp
// ❌ 错误示例：Singleton 持有 Scoped 引用
public class SingletonService
{
    private readonly ScopedService _scopedService; // 问题：Singleton 持有 Scoped 引用
    
    public SingletonService(ScopedService scopedService)
    {
        _scopedService = scopedService; // 这会导致内存泄漏
    }
}

// ✅ 正确示例：使用工厂模式
public class SingletonService
{
    private readonly IServiceProvider _serviceProvider;
    
    public SingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public void Process()
    {
        using var scope = _serviceProvider.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<ScopedService>();
        // 使用 scopedService，作用域结束后自动释放
    }
}
```

### 循环依赖的处理

**循环依赖的解决方案**：
```csharp
// 1. 使用属性注入
public class ServiceA
{
    public ServiceB ServiceB { get; set; } // 属性注入
}

public class ServiceB
{
    public ServiceA ServiceA { get; set; } // 属性注入
}

// 2. 使用工厂模式
public class ServiceA
{
    private readonly Func<ServiceB> _serviceBFactory;
    
    public ServiceA(Func<ServiceB> serviceBFactory)
    {
        _serviceBFactory = serviceBFactory;
    }
    
    public void Process()
    {
        var serviceB = _serviceBFactory(); // 按需创建
    }
}

// 3. 重构设计（推荐）
// 提取共同接口，避免直接依赖
public interface ICommonService { }
public class ServiceA : ICommonService { }
public class ServiceB : ICommonService { }
```

## ⚙️ 配置系统 {#configuration-system}

### 配置源的加载机制

配置系统采用分层加载策略，优先级从高到低：

1. **命令行参数**：最高优先级，支持 `--key=value` 格式
2. **环境变量**：支持嵌套配置，如 `ConnectionStrings__DefaultConnection`
3. **用户密钥**：开发环境使用，存储在用户配置目录
4. **配置文件**：JSON、XML、INI 等格式
5. **默认值**：代码中设置的默认值

**配置绑定示例**：
```csharp
public class DatabaseSettings
{
    public string ConnectionString { get; set; }
    public int CommandTimeout { get; set; } = 30; // 默认值
    public bool EnableRetryOnFailure { get; set; } = true;
}

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // 强类型配置绑定
        services.Configure<DatabaseSettings>(
            Configuration.GetSection("Database"));
        
        // 配置验证
        services.AddOptions<DatabaseSettings>()
            .Bind(Configuration.GetSection("Database"))
            .ValidateDataAnnotations()
            .ValidateOnStart(); // 启动时验证
    }
}
```

**配置热重载实现**：
```csharp
public class DatabaseService
{
    private readonly IOptionsMonitor<DatabaseSettings> _settings;
    private readonly ILogger<DatabaseService> _logger;

    public DatabaseService(
        IOptionsMonitor<DatabaseSettings> settings,
        ILogger<DatabaseService> logger)
    {
        _settings = settings;
        _logger = logger;
        
        // 监听配置变化
        _settings.OnChange(newSettings =>
        {
            _logger.LogInformation("Database settings updated: {ConnectionString}", 
                newSettings.ConnectionString);
            // 处理配置更新逻辑
        });
    }
    
    public async Task<Connection> GetConnectionAsync()
    {
        var settings = _settings.CurrentValue; // 获取当前配置
        // 使用配置创建连接
        return new Connection(settings.ConnectionString);
    }
}
```

**配置验证**：
```csharp
public class DatabaseSettingsValidator : AbstractValidator<DatabaseSettings>
{
    public DatabaseSettingsValidator()
    {
        RuleFor(x => x.ConnectionString)
            .NotEmpty()
            .Must(BeValidConnectionString)
            .WithMessage("Invalid connection string format");
            
        RuleFor(x => x.CommandTimeout)
            .GreaterThan(0)
            .LessThanOrEqualTo(300);
    }
    
    private bool BeValidConnectionString(string connectionString)
    {
        // 验证连接字符串格式
        return !string.IsNullOrEmpty(connectionString) && 
               connectionString.Contains("Server=");
    }
}
```

## 🔄 生命周期管理 {#lifecycle-management}

### 应用程序生命周期

**IHostedService 接口**：
```csharp
public class BackgroundDataSyncService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<BackgroundDataSyncService> _logger;
    private readonly PeriodicTimer _timer;

    public BackgroundDataSyncService(
        IServiceProvider serviceProvider,
        ILogger<BackgroundDataSyncService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _timer = new PeriodicTimer(TimeSpan.FromMinutes(5)); // 每5分钟执行
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            while (await _timer.WaitForNextTickAsync(stoppingToken))
            {
                await SyncDataAsync(stoppingToken);
            }
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("Background service stopped");
        }
    }

    private async Task SyncDataAsync(CancellationToken stoppingToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var dataService = scope.ServiceProvider.GetRequiredService<IDataService>();
        
        try
        {
            await dataService.SyncDataAsync(stoppingToken);
            _logger.LogInformation("Data sync completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Data sync failed");
        }
    }

    public override void Dispose()
    {
        _timer?.Dispose();
        base.Dispose();
    }
}
```

**优雅关闭处理**：
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();
        
        // 注册优雅关闭处理
        var lifetime = host.Services.GetRequiredService<IHostApplicationLifetime>();
        lifetime.ApplicationStopping.Register(() =>
        {
            Console.WriteLine("Application is stopping...");
            // 执行清理工作
        });
        
        await host.RunAsync();
    }
}
```

### 请求生命周期

**请求作用域管理**：
```csharp
public class RequestScopedService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public RequestScopedService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public string GetRequestId()
    {
        var context = _httpContextAccessor.HttpContext;
        if (context?.TraceIdentifier != null)
        {
            return context.TraceIdentifier;
        }
        return "unknown";
    }
}

// 注册服务
services.AddHttpContextAccessor();
services.AddScoped<RequestScopedService>();
```

## 🛣️ 路由系统 {#routing-system}

### 路由匹配机制

**路由配置示例**：
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseRouting();
    
    app.UseEndpoints(endpoints =>
    {
        // 1. 控制器路由
        endpoints.MapControllers();
        
        // 2. 约定路由
        endpoints.MapControllerRoute(
            name: "default",
            pattern: "{controller=Home}/{action=Index}/{id?}");
        
        // 3. 属性路由
        endpoints.MapControllerRoute(
            name: "api",
            pattern: "api/{controller}/{action}/{id?}");
        
        // 4. 自定义端点
        endpoints.MapGet("/health", async context =>
        {
            await context.Response.WriteAsync("Healthy");
        });
        
        // 5. 区域路由
        endpoints.MapAreaControllerRoute(
            name: "admin",
            areaName: "Admin",
            pattern: "Admin/{controller=Home}/{action=Index}/{id?}");
    });
}
```

**自定义路由约束**：
```csharp
public class CustomRouteConstraint : IRouteConstraint
{
    private readonly string _pattern;
    
    public CustomRouteConstraint(string pattern)
    {
        _pattern = pattern;
    }
    
    public bool Match(HttpContext httpContext, IRouter route, string routeKey, 
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var stringValue = value?.ToString();
            if (!string.IsNullOrEmpty(stringValue))
            {
                return Regex.IsMatch(stringValue, _pattern);
            }
        }
        return false;
    }
}

// 使用自定义约束
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id:custom(^[A-Z]{2}\\d{4}$)}")] // 使用自定义约束
    public IActionResult Get(string id)
    {
        // id 必须匹配 AA1234 格式
        return Ok($"Product {id}");
    }
}
```

## 📝 日志系统 {#logging-system}

### 结构化日志配置

**日志配置示例**：
```csharp
public class Program
{
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureLogging((context, logging) =>
            {
                logging.ClearProviders();
                
                if (context.HostingEnvironment.IsDevelopment())
                {
                    logging.AddConsole();
                    logging.AddDebug();
                }
                else
                {
                    // 生产环境使用结构化日志
                    logging.AddJsonConsole(options =>
                    {
                        options.IncludeScopes = true;
                        options.TimestampFormat = "yyyy-MM-dd HH:mm:ss ";
                    });
                }
                
                // 添加自定义日志提供程序
                logging.AddProvider(new CustomLogProvider());
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

**自定义日志提供程序**：
```csharp
public class CustomLogProvider : ILoggerProvider
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
    
    public IDisposable BeginScope<TState>(TState state)
    {
        return NullScope.Instance;
    }
    
    public bool IsEnabled(LogLevel logLevel)
    {
        return logLevel >= LogLevel.Information;
    }
    
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, 
        Exception exception, Func<TState, Exception, string> formatter)
    {
        if (!IsEnabled(logLevel)) return;
        
        var message = formatter(state, exception);
        var logEntry = new
        {
            Timestamp = DateTime.UtcNow,
            Level = logLevel.ToString(),
            Category = _categoryName,
            Message = message,
            Exception = exception?.ToString()
        };
        
        // 输出到控制台或文件
        Console.WriteLine(JsonSerializer.Serialize(logEntry));
    }
}

public class NullScope : IDisposable
{
    public static NullScope Instance { get; } = new NullScope();
    public void Dispose() { }
}
```

## ❓ 面试常见问题 {#interview-questions}

### Q1: 中间件的执行顺序为什么很重要？

**A**: 中间件的执行顺序直接影响：

**1. 异常处理机制**
- **异常处理中间件必须最先执行**：确保能捕获后续中间件执行过程中的所有异常
- **开发环境 vs 生产环境**：开发环境使用 `UseDeveloperExceptionPage()`，生产环境使用 `UseExceptionHandler()`
- **异常日志记录**：在异常处理中间件中记录详细的异常信息

**2. 性能优化考虑**
- **静态文件中间件在路由之前**：避免对静态文件请求进行不必要的路由计算
- **缓存中间件位置**：缓存中间件应该在需要缓存的内容之前执行
- **压缩中间件**：响应压缩应该在内容生成之后，发送之前执行

**3. 安全性保障**
- **HTTPS重定向在路由之前**：避免重复重定向和路由计算
- **认证授权顺序**：认证必须在授权之前，授权必须在业务逻辑之前
- **CORS策略**：CORS中间件应该在认证之前执行

**4. 业务逻辑依赖**
- **依赖注入顺序**：确保所有依赖服务在业务中间件执行前已准备就绪
- **会话管理**：会话中间件应该在需要会话数据的中间件之前执行

**代码示例**：
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // 1. 异常处理 - 必须放在最前面
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }

    // 2. HTTPS重定向 - 在路由之前
    app.UseHttpsRedirection();

    // 3. 静态文件 - 在路由之前，避免不必要的路由计算
    app.UseStaticFiles();

    // 4. 路由 - 确定请求的端点
    app.UseRouting();

    // 5. 认证授权 - 在端点执行之前
    app.UseAuthentication();
    app.UseAuthorization();

    // 6. 业务中间件
    app.UseCustomMiddleware();

    // 7. 端点
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

**常见错误顺序及后果**：
- **静态文件在路由之后**：所有静态文件请求都会经过路由计算，降低性能
- **认证在路由之前**：无法根据路由信息进行细粒度的认证控制
- **异常处理在中间**：无法捕获前面中间件的异常

### Q2: Singleton 服务中注入 Scoped 服务会有什么问题？

**A**: 会导致内存泄漏和资源管理问题，具体原因如下：

**1. 生命周期不匹配问题**
- **Singleton 服务**：在整个应用程序生命周期内存在，直到应用程序关闭
- **Scoped 服务**：在每个 HTTP 请求结束时被释放和回收
- **问题根源**：如果 Singleton 持有 Scoped 服务的引用，Scoped 服务永远不会被释放，导致内存泄漏

**2. 具体场景分析**
- **数据库上下文**：EF Core 的 `DbContext` 通常是 Scoped 服务，包含数据库连接池
- **缓存服务**：如果 Singleton 缓存服务持有 Scoped 的数据库上下文，会导致连接池耗尽
- **日志服务**：Singleton 日志服务不应该持有 Scoped 的请求相关服务

**3. 解决方案**

**方案一：使用工厂模式（推荐）**
```csharp
public class SingletonService
{
    private readonly IServiceProvider _serviceProvider;
    
    public SingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task ProcessAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        
        try
        {
            await scopedService.DoWorkAsync();
        }
        catch (Exception ex)
        {
            // 记录日志
            var logger = scope.ServiceProvider.GetRequiredService<ILogger<SingletonService>>();
            logger.LogError(ex, "Error processing in singleton service");
        }
    }
}
```

**方案二：使用 IServiceScopeFactory**
```csharp
public class SingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public SingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task ProcessAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scopedService.DoWorkAsync();
    }
}
```

**方案三：重构服务设计**
```csharp
// 将需要 Scoped 服务的逻辑提取到 Scoped 服务中
public interface IWorkService
{
    Task ProcessAsync();
}

public class WorkService : IWorkService
{
    private readonly IScopedService _scopedService;
    
    public WorkService(IScopedService scopedService)
    {
        _scopedService = scopedService;
    }
    
    public async Task ProcessAsync()
    {
        await _scopedService.DoWorkAsync();
    }
}

// Singleton 服务只负责协调，不直接使用 Scoped 服务
public class SingletonService
{
    private readonly IWorkService _workService;
    
    public SingletonService(IWorkService workService)
    {
        _workService = workService;
    }
    
    public async Task ProcessAsync()
    {
        await _workService.ProcessAsync();
    }
}
```

**4. 最佳实践**
- **避免在 Singleton 中注入 Scoped 服务**
- **使用工厂模式或 Scope 工厂创建临时作用域**
- **考虑重构服务设计，将 Scoped 依赖提取到合适的服务中**
- **在单元测试中验证服务生命周期配置**

### Q3: 如何实现配置热重载？

**A**: 配置热重载允许在不重启应用程序的情况下更新配置，主要通过以下方式实现：

**1. 配置源支持**
- **文件配置源**：`appsettings.json`、`appsettings.{Environment}.json`
- **环境变量**：支持环境变量覆盖配置文件
- **命令行参数**：支持命令行参数覆盖配置
- **用户密钥**：开发环境下的用户密钥存储

**2. 配置接口选择**

**IOptions<T>（不推荐用于热重载）**
```csharp
public class ConfigService
{
    private readonly AppSettings _settings;
    
    public ConfigService(IOptions<AppSettings> options)
    {
        _settings = options.Value; // 只在服务启动时读取一次
    }
}
```

**IOptionsMonitor<T>（推荐用于热重载）**
```csharp
public class ConfigService
{
    private readonly IOptionsMonitor<AppSettings> _settings;
    private readonly ILogger<ConfigService> _logger;
    
    public ConfigService(IOptionsMonitor<AppSettings> settings, ILogger<ConfigService> logger)
    {
        _settings = settings;
        _logger = logger;
        
        // 监听配置变化
        _settings.OnChange(newSettings =>
        {
            _logger.LogInformation("Configuration updated for {Service}", nameof(ConfigService));
            HandleConfigurationChange(newSettings);
        });
    }
    
    public AppSettings CurrentSettings => _settings.CurrentValue;
    
    private void HandleConfigurationChange(AppSettings newSettings)
    {
        // 处理配置更新逻辑
        // 例如：重新初始化数据库连接、更新缓存等
    }
}
```

**IOptionsSnapshot<T>（请求级别快照）**
```csharp
public class ConfigService
{
    private readonly IOptionsSnapshot<AppSettings> _settings;
    
    public ConfigService(IOptionsSnapshot<AppSettings> settings)
    {
        _settings = settings;
    }
    
    public AppSettings CurrentSettings => _settings.Value; // 每个请求获取最新配置
}
```

**3. 配置热重载实现示例**

**配置模型**：
```csharp
public class AppSettings
{
    public DatabaseSettings Database { get; set; }
    public CacheSettings Cache { get; set; }
    public LoggingSettings Logging { get; set; }
}

public class DatabaseSettings
{
    public string ConnectionString { get; set; }
    public int CommandTimeout { get; set; }
    public int MaxRetryCount { get; set; }
}
```

**服务注册**：
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 配置选项
    services.Configure<AppSettings>(Configuration);
    
    // 注册配置服务
    services.AddScoped<IConfigService, ConfigService>();
    
    // 配置验证
    services.AddOptions<AppSettings>()
        .Bind(Configuration)
        .ValidateDataAnnotations()
        .ValidateOnStart();
}
```

**配置更新处理**：
```csharp
public class ConfigUpdateHandler : IHostedService
{
    private readonly IOptionsMonitor<AppSettings> _settings;
    private readonly ILogger<ConfigUpdateHandler> _logger;
    private IDisposable _changeToken;
    
    public ConfigUpdateHandler(IOptionsMonitor<AppSettings> settings, ILogger<ConfigUpdateHandler> logger)
    {
        _settings = settings;
        _logger = logger;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _changeToken = _settings.OnChange(HandleConfigurationChange);
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _changeToken?.Dispose();
        return Task.CompletedTask;
    }
    
    private void HandleConfigurationChange(AppSettings newSettings)
    {
        _logger.LogInformation("Configuration updated at {Time}", DateTime.UtcNow);
        
        // 处理数据库连接更新
        if (newSettings.Database != null)
        {
            UpdateDatabaseConnection(newSettings.Database);
        }
        
        // 处理缓存配置更新
        if (newSettings.Cache != null)
        {
            UpdateCacheConfiguration(newSettings.Cache);
        }
    }
    
    private void UpdateDatabaseConnection(DatabaseSettings dbSettings)
    {
        // 实现数据库连接更新逻辑
    }
    
    private void UpdateCacheConfiguration(CacheSettings cacheSettings)
    {
        // 实现缓存配置更新逻辑
    }
}
```

**4. 最佳实践**
- **使用 IOptionsMonitor<T> 实现真正的热重载**
- **在配置更新时记录日志，便于问题排查**
- **实现配置验证，确保配置更新的正确性**
- **考虑配置更新的原子性，避免部分更新导致的问题**
- **在配置更新时优雅地处理依赖服务的重新初始化**

### Q4: 如何优化中间件性能？

**A**: 中间件性能优化是提高 ASP.NET Core 应用程序响应速度的关键，主要通过以下方式实现：

**1. 条件中间件优化**
- **环境条件判断**：根据环境变量决定是否启用特定中间件
- **配置条件判断**：根据配置文件决定中间件的行为
- **功能开关**：使用功能标志控制中间件的启用/禁用

**2. 短路优化策略**
- **早期返回**：在中间件开始时就判断是否需要继续处理
- **路径过滤**：只对特定路径应用中间件逻辑
- **请求类型过滤**：根据 HTTP 方法或请求头决定处理逻辑

**3. 异步操作优化**
- **异步方法**：使用 `async/await` 避免阻塞线程
- **并行处理**：对于独立的操作使用并行处理
- **取消令牌支持**：实现 `CancellationToken` 支持，提高响应性

**4. 缓存策略优化**
- **结果缓存**：缓存中间件的计算结果
- **配置缓存**：缓存配置信息，避免重复读取
- **静态资源缓存**：对静态资源实现有效的缓存策略

**代码示例**：

**条件中间件实现**：
```csharp
public class ConditionalMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _config;
    private readonly ILogger<ConditionalMiddleware> _logger;
    
    public ConditionalMiddleware(RequestDelegate next, IConfiguration config, ILogger<ConditionalMiddleware> logger)
    {
        _next = next;
        _config = config;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 1. 环境条件判断
        if (!_config.GetValue<bool>("EnableFeature"))
        {
            await _next(context);
            return;
        }
        
        // 2. 路径过滤
        if (!context.Request.Path.StartsWithSegments("/api"))
        {
            await _next(context);
            return;
        }
        
        // 3. 请求类型过滤
        if (context.Request.Method != "POST")
        {
            await _next(context);
            return;
        }
        
        // 4. 执行特定逻辑
        var stopwatch = Stopwatch.StartNew();
        try
        {
            await ProcessFeature(context);
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation("Feature processing took {ElapsedMs}ms", stopwatch.ElapsedMilliseconds);
        }
    }
    
    private async Task ProcessFeature(HttpContext context)
    {
        // 实现特定的业务逻辑
        await Task.Delay(10); // 模拟处理时间
    }
}
```

**性能优化中间件**：
```csharp
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly ILogger<PerformanceMiddleware> _logger;
    
    public PerformanceMiddleware(RequestDelegate next, IMemoryCache cache, ILogger<PerformanceMiddleware> logger)
    {
        _next = next;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var cacheKey = $"perf:{context.Request.Path}:{context.Request.Method}";
        
        // 1. 缓存检查
        if (_cache.TryGetValue(cacheKey, out var cachedResult))
        {
            context.Response.StatusCode = 200;
            await context.Response.WriteAsync(cachedResult.ToString());
            return;
        }
        
        // 2. 性能监控
        var stopwatch = Stopwatch.StartNew();
        var originalBodyStream = context.Response.Body;
        
        try
        {
            using var memoryStream = new MemoryStream();
            context.Response.Body = memoryStream;
            
            await _next(context);
            
            stopwatch.Stop();
            
            // 3. 缓存结果（仅对成功的 GET 请求）
            if (context.Response.StatusCode == 200 && context.Request.Method == "GET")
            {
                memoryStream.Position = 0;
                var result = await new StreamReader(memoryStream).ReadToEndAsync();
                
                var cacheOptions = new MemoryCacheEntryOptions()
                    .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                    .SetAbsoluteExpiration(TimeSpan.FromMinutes(30));
                
                _cache.Set(cacheKey, result, cacheOptions);
            }
            
            // 4. 写回响应
            memoryStream.Position = 0;
            await memoryStream.CopyToAsync(originalBodyStream);
        }
        finally
        {
            context.Response.Body = originalBodyStream;
            _logger.LogInformation("Request {Path} processed in {ElapsedMs}ms", 
                context.Request.Path, stopwatch.ElapsedMilliseconds);
        }
    }
}
```

**异步优化中间件**：
```csharp
public class AsyncOptimizationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SemaphoreSlim _semaphore;
    
    public AsyncOptimizationMiddleware(RequestDelegate next)
    {
        _next = next;
        _semaphore = new SemaphoreSlim(100, 100); // 限制并发数量
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 1. 并发控制
        await _semaphore.WaitAsync();
        try
        {
            // 2. 取消令牌支持
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(context.RequestAborted);
            cts.CancelAfter(TimeSpan.FromSeconds(30)); // 30秒超时
            
            // 3. 并行处理（如果有多个独立操作）
            var tasks = new List<Task>();
            
            // 并行执行独立操作
            if (context.Request.Headers.ContainsKey("X-Parallel-Process"))
            {
                tasks.Add(ProcessTask1(cts.Token));
                tasks.Add(ProcessTask2(cts.Token));
                tasks.Add(ProcessTask3(cts.Token));
                
                await Task.WhenAll(tasks);
            }
            
            await _next(context);
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    private async Task ProcessTask1(CancellationToken cancellationToken)
    {
        await Task.Delay(100, cancellationToken);
    }
    
    private async Task ProcessTask2(CancellationToken cancellationToken)
    {
        await Task.Delay(150, cancellationToken);
    }
    
    private async Task ProcessTask3(CancellationToken cancellationToken)
    {
        await Task.Delay(200, cancellationToken);
    }
}
```

**5. 最佳实践总结**
- **合理使用条件判断**：避免不必要的中间件执行
- **实现短路逻辑**：尽早返回，减少处理时间
- **使用异步操作**：避免阻塞线程池
- **实现缓存策略**：减少重复计算和 I/O 操作
- **监控性能指标**：使用日志和指标监控中间件性能
- **控制并发数量**：避免资源耗尽
- **支持取消操作**：提高应用程序的响应性

## 📚 总结

ASP.NET Core 的核心概念是面试中的高频考点，需要深入理解：

1. **中间件管道**：理解责任链模式和执行顺序的重要性
2. **依赖注入**：掌握生命周期管理和避免常见陷阱
3. **配置系统**：了解多源配置和热重载机制
4. **生命周期**：理解应用程序和请求的生命周期管理
5. **路由系统**：掌握路由匹配和自定义约束
6. **日志系统**：了解结构化日志和性能优化

这些知识点不仅要在理论上理解，更要能够用代码示例来说明，在面试中展现出真正的技术深度。
