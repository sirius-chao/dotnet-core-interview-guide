# ASP.NET Core 性能优化深度指南

## 1. 性能优化哲学深度思考

### 1.1 性能优化的本质理解

**性能优化不仅仅是代码优化，更是一种系统思维的深度体现**：

**性能优化的认知层次**：
1. **代码层面**：具体的代码优化和算法改进
2. **架构层面**：系统架构层面的性能优化
3. **思维层面**：培养性能优化的思维方式

**性能优化的核心原则**：
- **测量优先**：基于测量结果驱动优化
- **瓶颈识别**：识别真正的性能瓶颈
- **权衡考虑**：在性能和其他因素间找到平衡
- **持续优化**：建立持续优化的机制

### 1.2 性能优化的方法论

**性能优化流程**：
1. **性能目标设定**：明确性能优化目标
2. **性能基准测试**：建立性能基准
3. **性能瓶颈识别**：识别性能瓶颈
4. **优化方案设计**：设计优化方案
5. **优化实施验证**：实施并验证优化效果

## 2. 中间件管道性能优化

### 2.1 中间件顺序优化

**中间件执行顺序的深度优化**：
```csharp
// 性能优化的中间件配置
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // 1. 异常处理（最外层，捕获所有异常）
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseExceptionHandler("/Error");
    
    // 2. HTTPS 重定向（早期处理，避免后续处理）
    app.UseHttpsRedirection();
    
    // 3. 静态文件（可能短路，减少后续中间件执行）
    app.UseStaticFiles(new StaticFileOptions
    {
        OnPrepareResponse = ctx =>
        {
            // 设置缓存策略
            ctx.Context.Response.Headers.Append(
                "Cache-Control", "public,max-age=31536000");
        }
    });
    
    // 4. 路由（确定请求路径，避免不必要的中间件执行）
    app.UseRouting();
    
    // 5. 认证（早期验证，避免未授权访问后续资源）
    app.UseAuthentication();
    
    // 6. 授权（基于认证结果进行授权）
    app.UseAuthorization();
    
    // 7. 业务中间件（核心业务逻辑）
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}

// 自定义性能监控中间件
public class PerformanceMonitoringMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMonitoringMiddleware> _logger;
    private readonly IMetricsCollector _metricsCollector;
    
    public PerformanceMonitoringMiddleware(
        RequestDelegate next, 
        ILogger<PerformanceMonitoringMiddleware> logger,
        IMetricsCollector metricsCollector)
    {
        _next = next;
        _logger = logger;
        _metricsCollector = metricsCollector;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        var startMemory = GC.GetTotalMemory(false);
        
        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();
            var endMemory = GC.GetTotalMemory(false);
            var memoryDelta = endMemory - startMemory;
            
            // 记录性能指标
            _metricsCollector.RecordRequestMetrics(
                context.Request.Path,
                sw.ElapsedMilliseconds,
                memoryDelta,
                context.Response.StatusCode);
            
            // 记录慢请求
            if (sw.ElapsedMilliseconds > 100)
            {
                _logger.LogWarning(
                    "Slow request: {Path} took {Elapsed}ms, memory delta: {MemoryDelta} bytes", 
                    context.Request.Path, sw.ElapsedMilliseconds, memoryDelta);
            }
        }
    }
}

// 性能指标收集器
public interface IMetricsCollector
{
    void RecordRequestMetrics(string path, long duration, long memoryDelta, int statusCode);
}

public class MetricsCollector : IMetricsCollector
{
    private readonly ConcurrentDictionary<string, RequestMetrics> _pathMetrics = new();
    
    public void RecordRequestMetrics(string path, long duration, long memoryDelta, int statusCode)
    {
        var metrics = _pathMetrics.GetOrAdd(path, _ => new RequestMetrics());
        metrics.RecordRequest(duration, memoryDelta, statusCode);
    }
    
    public IEnumerable<PathPerformanceReport> GetPerformanceReport()
    {
        return _pathMetrics.Select(kvp => new PathPerformanceReport
        {
            Path = kvp.Key,
            TotalRequests = kvp.Value.TotalRequests,
            AverageDuration = kvp.Value.AverageDuration,
            P95Duration = kvp.Value.P95Duration,
            P99Duration = kvp.Value.P99Duration,
            AverageMemoryDelta = kvp.Value.AverageMemoryDelta,
            ErrorRate = kvp.Value.ErrorRate
        });
    }
}

public class RequestMetrics
{
    private readonly ConcurrentQueue<long> _durations = new();
    private readonly ConcurrentQueue<long> _memoryDeltas = new();
    private long _totalRequests;
    private long _errorRequests;
    
    public void RecordRequest(long duration, long memoryDelta, int statusCode)
    {
        Interlocked.Increment(ref _totalRequests);
        if (statusCode >= 400)
            Interlocked.Increment(ref _errorRequests);
        
        _durations.Enqueue(duration);
        _memoryDeltas.Enqueue(memoryDelta);
        
        // 保持队列大小，避免内存泄漏
        if (_durations.Count > 1000)
        {
            _durations.TryDequeue(out _);
            _memoryDeltas.TryDequeue(out _);
        }
    }
    
    public long TotalRequests => _totalRequests;
    public double AverageDuration => _durations.Any() ? _durations.Average() : 0;
    public double P95Duration => CalculatePercentile(_durations.ToArray(), 95);
    public double P99Duration => CalculatePercentile(_durations.ToArray(), 99);
    public double AverageMemoryDelta => _memoryDeltas.Any() ? _memoryDeltas.Average() : 0;
    public double ErrorRate => _totalRequests > 0 ? (double)_errorRequests / _totalRequests : 0;
    
    private double CalculatePercentile(long[] values, int percentile)
    {
        if (!values.Any()) return 0;
        
        Array.Sort(values);
        var index = (int)Math.Ceiling((percentile / 100.0) * values.Length) - 1;
        return values[Math.Max(0, index)];
    }
}
```

### 2.2 中间件短路优化

**短路机制的深度实现**：
```csharp
// 短路中间件示例
public class ShortCircuitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ShortCircuitMiddleware> _logger;
    private readonly IConfiguration _configuration;
    
    public ShortCircuitMiddleware(
        RequestDelegate next, 
        ILogger<ShortCircuitMiddleware> logger,
        IConfiguration configuration)
    {
        _next = next;
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 检查是否需要短路
        if (ShouldShortCircuit(context))
        {
            _logger.LogInformation("Request short-circuited for path: {Path}", context.Request.Path);
            context.Response.StatusCode = 404;
            await context.Response.WriteAsync("Resource not found");
            return; // 短路，不调用下一个中间件
        }
        
        await _next(context);
    }
    
    private bool ShouldShortCircuit(HttpContext context)
    {
        var path = context.Request.Path.Value;
        
        // 1. 无效的 API 路径
        if (path.StartsWith("/api/") && !IsValidApiPath(path))
            return true;
        
        // 2. 恶意请求检测
        if (IsMaliciousRequest(context))
            return true;
        
        // 3. 维护模式检查
        if (IsMaintenanceMode() && !IsMaintenanceBypass(context))
            return true;
        
        return false;
    }
    
    private bool IsValidApiPath(string path)
    {
        // 检查 API 版本和路径有效性
        var validPatterns = new[]
        {
            @"/api/v1/.*",
            @"/api/v2/.*",
            @"/api/health",
            @"/api/metrics"
        };
        
        return validPatterns.Any(pattern => Regex.IsMatch(path, pattern));
    }
    
    private bool IsMaliciousRequest(HttpContext context)
    {
        // 简单的恶意请求检测
        var userAgent = context.Request.Headers["User-Agent"].ToString();
        var suspiciousPatterns = new[]
        {
            "bot",
            "crawler",
            "scanner",
            "sqlmap"
        };
        
        return suspiciousPatterns.Any(pattern => 
            userAgent.Contains(pattern, StringComparison.OrdinalIgnoreCase));
    }
    
    private bool IsMaintenanceMode()
    {
        return _configuration.GetValue<bool>("Maintenance:Enabled", false);
    }
    
    private bool IsMaintenanceBypass(HttpContext context)
    {
        var bypassToken = context.Request.Headers["X-Maintenance-Bypass"].ToString();
        var expectedToken = _configuration["Maintenance:BypassToken"];
        return bypassToken == expectedToken;
    }
}

// 条件中间件执行
public class ConditionalMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;
    private readonly IFeatureManager _featureManager;
    
    public ConditionalMiddleware(
        RequestDelegate next, 
        IConfiguration configuration,
        IFeatureManager featureManager)
    {
        _next = next;
        _configuration = configuration;
        _featureManager = featureManager;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 根据条件决定是否执行中间件
        if (ShouldExecuteMiddleware(context))
        {
            await _next(context);
        }
        else
        {
            // 跳过中间件执行，直接返回
            context.Response.StatusCode = 200;
            await context.Response.WriteAsync("Middleware skipped");
        }
    }
    
    private bool ShouldExecuteMiddleware(HttpContext context)
    {
        var environment = _configuration["Environment"];
        var path = context.Request.Path.Value;
        
        // 1. 环境条件
        if (environment == "Development" && path.StartsWith("/debug/"))
            return true;
        
        // 2. 功能开关
        if (path.StartsWith("/beta/") && !_featureManager.IsEnabledAsync("BetaFeatures").Result)
            return false;
        
        // 3. 用户角色条件
        if (path.StartsWith("/admin/") && !context.User.IsInRole("Admin"))
            return false;
        
        // 4. 时间条件
        if (IsOutsideBusinessHours() && path.StartsWith("/business/"))
            return false;
        
        return true;
    }
    
    private bool IsOutsideBusinessHours()
    {
        var now = DateTime.UtcNow;
        var businessStart = TimeSpan.FromHours(9); // 9 AM UTC
        var businessEnd = TimeSpan.FromHours(17);  // 5 PM UTC
        
        return now.TimeOfDay < businessStart || now.TimeOfDay > businessEnd;
    }
}
```

## 3. 依赖注入性能优化

### 3.1 服务注册优化

**服务注册的性能优化策略**：
```csharp
// 服务注册性能优化
public static class ServiceCollectionPerformanceExtensions
{
    public static IServiceCollection AddOptimizedServices(this IServiceCollection services)
    {
        // 1. 使用 TryAdd 避免重复注册
        services.TryAddSingleton<IConfigurationService, ConfigurationService>();
        services.TryAddScoped<IDataService, DataService>();
        services.TryAddTransient<IUtilityService, UtilityService>();
        
        // 2. 批量注册服务
        services.AddServicesByConvention(typeof(Startup).Assembly);
        
        // 3. 条件注册
        services.AddConditionalServices();
        
        return services;
    }
    
    public static IServiceCollection AddServicesByConvention(this IServiceCollection services, Assembly assembly)
    {
        var serviceTypes = assembly.GetTypes()
            .Where(t => t.IsClass && !t.IsAbstract && t.GetInterfaces().Any())
            .Where(t => t.Name.EndsWith("Service") || t.Name.EndsWith("Repository"));
        
        foreach (var serviceType in serviceTypes)
        {
            var interfaces = serviceType.GetInterfaces()
                .Where(i => i != typeof(IDisposable));
            
            foreach (var serviceInterface in interfaces)
            {
                // 根据命名约定决定生命周期
                var lifetime = DetermineLifetime(serviceType);
                
                switch (lifetime)
                {
                    case ServiceLifetime.Singleton:
                        services.TryAddSingleton(serviceInterface, serviceType);
                        break;
                    case ServiceLifetime.Scoped:
                        services.TryAddScoped(serviceInterface, serviceType);
                        break;
                    case ServiceLifetime.Transient:
                        services.TryAddTransient(serviceInterface, serviceType);
                        break;
                }
            }
        }
        
        return services;
    }
    
    private static ServiceLifetime DetermineLifetime(Type serviceType)
    {
        var typeName = serviceType.Name.ToLower();
        
        // 缓存、配置等应该是 Singleton
        if (typeName.Contains("cache") || typeName.Contains("config") || typeName.Contains("factory"))
            return ServiceLifetime.Singleton;
        
        // 数据访问、业务逻辑等应该是 Scoped
        if (typeName.Contains("repository") || typeName.Contains("service") || typeName.Contains("manager"))
            return ServiceLifetime.Scoped;
        
        // 工具类、辅助类等可以是 Transient
        return ServiceLifetime.Transient;
    }
    
    public static IServiceCollection AddConditionalServices(this IServiceCollection services)
    {
        var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
        
        // 根据环境注册不同的服务实现
        if (environment == "Development")
        {
            services.AddSingleton<ILogger, DevelopmentLogger>();
            services.AddSingleton<ICacheService, InMemoryCacheService>();
        }
        else
        {
            services.AddSingleton<ILogger, ProductionLogger>();
            services.AddSingleton<ICacheService, RedisCacheService>();
        }
        
        return services;
    }
}

// 服务工厂优化
public class OptimizedServiceFactory
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ConcurrentDictionary<Type, object> _singletonCache = new();
    private readonly ConcurrentDictionary<Type, Func<object>> _factoryCache = new();
    
    public OptimizedServiceFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public T GetService<T>()
    {
        var serviceType = typeof(T);
        
        // 检查是否是 Singleton
        if (IsSingleton(serviceType))
        {
            return (T)_singletonCache.GetOrAdd(serviceType, _ => _serviceProvider.GetService<T>());
        }
        
        // 使用工厂缓存
        var factory = _factoryCache.GetOrAdd(serviceType, _ => 
            new Func<object>(() => _serviceProvider.GetService<T>()));
        
        return (T)factory();
    }
    
    private bool IsSingleton(Type serviceType)
    {
        // 检查服务描述符中的生命周期
        var serviceDescriptor = _serviceProvider.GetService<IServiceCollection>()
            ?.FirstOrDefault(sd => sd.ServiceType == serviceType);
        
        return serviceDescriptor?.Lifetime == ServiceLifetime.Singleton;
    }
}
```

### 3.2 服务解析优化

**服务解析的性能优化**：
```csharp
// 服务解析性能优化
public class ServiceResolverOptimizer
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ConcurrentDictionary<Type, object> _resolvedServices = new();
    private readonly ConcurrentDictionary<Type, Type[]> _constructorParameters = new();
    
    public ServiceResolverOptimizer(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public T Resolve<T>()
    {
        var serviceType = typeof(T);
        
        // 检查缓存
        if (_resolvedServices.TryGetValue(serviceType, out var cached))
        {
            return (T)cached;
        }
        
        // 解析服务
        var service = _serviceProvider.GetService<T>();
        if (service != null)
        {
            _resolvedServices.TryAdd(serviceType, service);
        }
        
        return service;
    }
    
    public T ResolveWithCachedParameters<T>()
    {
        var serviceType = typeof(T);
        
        // 获取构造函数参数类型（缓存）
        var parameterTypes = _constructorParameters.GetOrAdd(serviceType, type =>
        {
            var constructors = type.GetConstructors();
            if (constructors.Length == 0) return Array.Empty<Type>();
            
            var bestConstructor = constructors
                .OrderByDescending(c => c.GetParameters().Length)
                .First();
            
            return bestConstructor.GetParameters()
                .Select(p => p.ParameterType)
                .ToArray();
        });
        
        // 预解析所有依赖
        var parameters = parameterTypes.Select(t => _serviceProvider.GetService(t)).ToArray();
        
        // 创建实例
        var instance = Activator.CreateInstance(serviceType, parameters);
        return (T)instance;
    }
}

// 延迟服务解析
public class LazyServiceResolver<T>
{
    private readonly Lazy<T> _lazyService;
    
    public LazyServiceResolver(IServiceProvider serviceProvider)
    {
        _lazyService = new Lazy<T>(() => serviceProvider.GetService<T>());
    }
    
    public T Service => _lazyService.Value;
}

// 服务解析性能监控
public class ServiceResolutionMonitor
{
    private readonly ConcurrentDictionary<Type, ResolutionMetrics> _metrics = new();
    
    public void RecordResolution(Type serviceType, long duration, bool success)
    {
        var metrics = _metrics.GetOrAdd(serviceType, _ => new ResolutionMetrics());
        metrics.RecordResolution(duration, success);
    }
    
    public IEnumerable<ServiceResolutionReport> GetReport()
    {
        return _metrics.Select(kvp => new ServiceResolutionReport
        {
            ServiceType = kvp.Key.Name,
            TotalResolutions = kvp.Value.TotalResolutions,
            AverageResolutionTime = kvp.Value.AverageResolutionTime,
            SuccessRate = kvp.Value.SuccessRate,
            LastResolutionTime = kvp.Value.LastResolutionTime
        });
    }
}

public class ResolutionMetrics
{
    private long _totalResolutions;
    private long _successfulResolutions;
    private long _totalTime;
    private DateTime _lastResolutionTime;
    
    public void RecordResolution(long duration, bool success)
    {
        Interlocked.Increment(ref _totalResolutions);
        if (success)
            Interlocked.Increment(ref _successfulResolutions);
        
        Interlocked.Add(ref _totalTime, duration);
        _lastResolutionTime = DateTime.UtcNow;
    }
    
    public long TotalResolutions => _totalResolutions;
    public double AverageResolutionTime => _totalResolutions > 0 ? (double)_totalTime / _totalResolutions : 0;
    public double SuccessRate => _totalResolutions > 0 ? (double)_successfulResolutions / _totalResolutions : 0;
    public DateTime LastResolutionTime => _lastResolutionTime;
}
```

## 4. 配置系统性能优化

### 4.1 配置绑定优化

**配置绑定的性能优化**：
```csharp
// 配置绑定性能优化
public class OptimizedConfigurationBinder
{
    private readonly IConfiguration _configuration;
    private readonly ConcurrentDictionary<Type, object> _boundConfigurations = new();
    private readonly ConcurrentDictionary<string, object> _sectionCache = new();
    
    public OptimizedConfigurationBinder(IConfiguration configuration)
    {
        _configuration = configuration;
    }
    
    public T GetConfiguration<T>(string sectionName = null) where T : class, new()
    {
        var configType = typeof(T);
        var cacheKey = sectionName ?? configType.FullName;
        
        // 检查缓存
        if (_boundConfigurations.TryGetValue(configType, out var cached))
        {
            return (T)cached;
        }
        
        // 绑定配置
        var config = new T();
        var section = string.IsNullOrEmpty(sectionName) ? _configuration : _configuration.GetSection(sectionName);
        
        // 使用优化的绑定方法
        BindConfigurationOptimized(section, config);
        
        // 缓存结果
        _boundConfigurations.TryAdd(configType, config);
        
        return config;
    }
    
    private void BindConfigurationOptimized(IConfiguration configuration, object target)
    {
        var targetType = target.GetType();
        var properties = targetType.GetProperties(BindingFlags.Public | BindingFlags.Instance);
        
        foreach (var property in properties)
        {
            if (!property.CanWrite) continue;
            
            var configValue = configuration[property.Name];
            if (configValue == null) continue;
            
            try
            {
                var convertedValue = ConvertValue(configValue, property.PropertyType);
                property.SetValue(target, convertedValue);
            }
            catch (Exception ex)
            {
                // 记录转换错误但继续处理
                Debug.WriteLine($"Failed to convert configuration value for {property.Name}: {ex.Message}");
            }
        }
    }
    
    private object ConvertValue(string value, Type targetType)
    {
        if (targetType == typeof(string))
            return value;
        
        if (targetType == typeof(int) || targetType == typeof(int?))
            return int.TryParse(value, out var intResult) ? intResult : 0;
        
        if (targetType == typeof(bool) || targetType == typeof(bool?))
            return bool.TryParse(value, out var boolResult) && boolResult;
        
        if (targetType == typeof(double) || targetType == typeof(double?))
            return double.TryParse(value, out var doubleResult) ? doubleResult : 0.0;
        
        if (targetType == typeof(DateTime) || targetType == typeof(DateTime?))
            return DateTime.TryParse(value, out var dateResult) ? dateResult : DateTime.MinValue;
        
        // 默认返回原始值
        return value;
    }
    
    // 批量配置绑定
    public void BindMultipleConfigurations<T>(IEnumerable<string> sectionNames) where T : class, new()
    {
        var tasks = sectionNames.Select(sectionName => Task.Run(() =>
        {
            try
            {
                GetConfiguration<T>(sectionName);
            }
            catch (Exception ex)
            {
                Debug.WriteLine($"Failed to bind configuration for section {sectionName}: {ex.Message}");
            }
        }));
        
        Task.WaitAll(tasks.ToArray());
    }
}

// 配置验证优化
public class OptimizedConfigurationValidator
{
    private readonly ConcurrentDictionary<Type, IValidator> _validators = new();
    
    public ValidationResult ValidateConfiguration<T>(T configuration) where T : class
    {
        var configType = typeof(T);
        
        // 获取或创建验证器
        var validator = _validators.GetOrAdd(configType, type =>
        {
            // 使用反射创建验证器实例
            var validatorType = typeof(AbstractValidator<>).MakeGenericType(type);
            return (IValidator)Activator.CreateInstance(validatorType);
        });
        
        if (validator == null)
            return new ValidationResult();
        
        // 执行验证
        var context = new ValidationContext<T>(configuration);
        return validator.Validate(context);
    }
    
    // 批量验证
    public IEnumerable<ValidationResult> ValidateMultipleConfigurations<T>(IEnumerable<T> configurations) where T : class
    {
        return configurations.Select(ValidateConfiguration);
    }
}
```

### 4.2 配置热重载优化

**配置热重载的性能优化**：
```csharp
// 优化的配置热重载
public class OptimizedConfigurationReloadService : IDisposable
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<OptimizedConfigurationReloadService> _logger;
    private readonly FileSystemWatcher _fileWatcher;
    private readonly Timer _reloadTimer;
    private readonly object _lock = new object();
    private readonly ConcurrentDictionary<string, DateTime> _lastModifiedTimes = new();
    private readonly ConcurrentDictionary<string, object> _configurationCache = new();
    
    public OptimizedConfigurationReloadService(
        IConfiguration configuration, 
        ILogger<OptimizedConfigurationReloadService> logger)
    {
        _configuration = configuration;
        _logger = logger;
        
        // 设置文件监控
        var configPath = Path.GetDirectoryName(GetConfigFilePath());
        _fileWatcher = new FileSystemWatcher(configPath, "*.json")
        {
            NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.Size,
            EnableRaisingEvents = true,
            IncludeSubdirectories = false
        };
        
        _fileWatcher.Changed += OnConfigurationChanged;
        
        // 设置重载定时器（防抖）
        _reloadTimer = new Timer(ReloadConfiguration, null, Timeout.Infinite, Timeout.Infinite);
        
        // 初始化缓存
        InitializeConfigurationCache();
    }
    
    private void InitializeConfigurationCache()
    {
        try
        {
            var configFiles = Directory.GetFiles(Path.GetDirectoryName(GetConfigFilePath()), "*.json");
            foreach (var file in configFiles)
            {
                var fileInfo = new FileInfo(file);
                _lastModifiedTimes[file] = fileInfo.LastWriteTimeUtc;
                
                // 预加载配置到缓存
                var configContent = File.ReadAllText(file);
                _configurationCache[file] = JsonSerializer.Deserialize<JsonElement>(configContent);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to initialize configuration cache");
        }
    }
    
    private void OnConfigurationChanged(object sender, FileSystemEventArgs e)
    {
        try
        {
            var fileInfo = new FileInfo(e.FullPath);
            var lastModified = fileInfo.LastWriteTimeUtc;
            
            // 检查文件是否真的发生了变化
            if (_lastModifiedTimes.TryGetValue(e.FullPath, out var previousModified) &&
                lastModified <= previousModified)
            {
                return; // 文件没有真正变化
            }
            
            // 更新最后修改时间
            _lastModifiedTimes[e.FullPath] = lastModified;
            
            // 防抖处理，避免频繁重载
            _reloadTimer.Change(500, Timeout.Infinite);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling configuration change for {File}", e.FullPath);
        }
    }
    
    private void ReloadConfiguration(object state)
    {
        lock (_lock)
        {
            try
            {
                _logger.LogInformation("Reloading configuration");
                
                // 重新加载配置
                if (_configuration is IConfigurationRoot configRoot)
                {
                    configRoot.Reload();
                }
                
                // 更新缓存
                UpdateConfigurationCache();
                
                // 触发配置变更事件
                OnConfigurationReloaded();
                
                _logger.LogInformation("Configuration reloaded successfully");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to reload configuration");
            }
        }
    }
    
    private void UpdateConfigurationCache()
    {
        try
        {
            var configFiles = Directory.GetFiles(Path.GetDirectoryName(GetConfigFilePath()), "*.json");
            foreach (var file in configFiles)
            {
                var configContent = File.ReadAllText(file);
                _configurationCache[file] = JsonSerializer.Deserialize<JsonElement>(configContent);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to update configuration cache");
        }
    }
    
    private void OnConfigurationReloaded()
    {
        // 通知其他服务配置已更新
        ConfigurationReloaded?.Invoke(this, EventArgs.Empty);
    }
    
    private string GetConfigFilePath()
    {
        return "appsettings.json";
    }
    
    public event EventHandler ConfigurationReloaded;
    
    public void Dispose()
    {
        _fileWatcher?.Dispose();
        _reloadTimer?.Dispose();
    }
}
```

## 5. 日志系统性能优化

### 5.1 日志管道优化

**日志系统的性能优化**：
```csharp
// 高性能日志提供者
public class HighPerformanceLogProvider : ILoggerProvider
{
    private readonly IConfiguration _configuration;
    private readonly ConcurrentDictionary<string, HighPerformanceLogger> _loggers = new();
    private readonly Channel<LogEntry> _logChannel;
    private readonly CancellationTokenSource _cancellationTokenSource;
    private readonly Task _processingTask;
    
    public HighPerformanceLogProvider(IConfiguration configuration)
    {
        _configuration = configuration;
        _cancellationTokenSource = new CancellationTokenSource();
        
        // 创建高性能日志通道
        _logChannel = Channel.CreateUnbounded<LogEntry>(new UnboundedChannelOptions
        {
            SingleReader = true,
            SingleWriter = false,
            AllowSynchronousContinuations = false
        });
        
        // 启动后台处理任务
        _processingTask = Task.Run(ProcessLogsAsync);
    }
    
    public ILogger CreateLogger(string categoryName)
    {
        return _loggers.GetOrAdd(categoryName, name => 
            new HighPerformanceLogger(name, _logChannel.Writer));
    }
    
    private async Task ProcessLogsAsync()
    {
        var reader = _logChannel.Reader;
        var batchSize = _configuration.GetValue<int>("Logging:BatchSize", 100);
        var batchTimeout = _configuration.GetValue<int>("Logging:BatchTimeoutMs", 1000);
        
        var batch = new List<LogEntry>();
        var timer = new Timer(_ => ProcessBatch(batch), null, batchTimeout, Timeout.Infinite);
        
        try
        {
            while (await reader.WaitToReadAsync(_cancellationTokenSource.Token))
            {
                while (reader.TryRead(out var logEntry))
                {
                    batch.Add(logEntry);
                    
                    if (batch.Count >= batchSize)
                    {
                        ProcessBatch(batch);
                        batch.Clear();
                        timer.Change(batchTimeout, Timeout.Infinite);
                    }
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 正常取消
        }
        finally
        {
            timer.Dispose();
            // 处理剩余的日志
            if (batch.Any())
            {
                ProcessBatch(batch);
            }
        }
    }
    
    private void ProcessBatch(List<LogEntry> batch)
    {
        try
        {
            // 批量写入日志
            var logPath = _configuration["Logging:FilePath"] ?? "logs/app.log";
            var logDir = Path.GetDirectoryName(logPath);
            
            if (!Directory.Exists(logDir))
                Directory.CreateDirectory(logDir);
            
            var logLines = batch.Select(entry => JsonSerializer.Serialize(entry) + Environment.NewLine);
            File.AppendAllLines(logPath, logLines);
        }
        catch (Exception ex)
        {
            // 记录错误但不抛出异常
            Debug.WriteLine($"Failed to process log batch: {ex.Message}");
        }
    }
    
    public void Dispose()
    {
        _cancellationTokenSource.Cancel();
        _processingTask?.Wait(TimeSpan.FromSeconds(5));
        _cancellationTokenSource.Dispose();
    }
}

public class HighPerformanceLogger : ILogger
{
    private readonly string _categoryName;
    private readonly ChannelWriter<LogEntry> _logWriter;
    
    public HighPerformanceLogger(string categoryName, ChannelWriter<LogEntry> logWriter)
    {
        _categoryName = categoryName;
        _logWriter = logWriter;
    }
    
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter)
    {
        if (!IsEnabled(logLevel))
            return;
        
        var message = formatter(state, exception);
        var logEntry = new LogEntry
        {
            Timestamp = DateTime.UtcNow,
            Level = logLevel.ToString(),
            Category = _categoryName,
            Message = message,
            Exception = exception?.ToString(),
            EventId = eventId.Id
        };
        
        // 异步写入日志通道
        _ = _logWriter.WriteAsync(logEntry);
    }
    
    public bool IsEnabled(LogLevel logLevel)
    {
        var minLevel = Environment.GetEnvironmentVariable("LOG_LEVEL") switch
        {
            "Trace" => LogLevel.Trace,
            "Debug" => LogLevel.Debug,
            "Information" => LogLevel.Information,
            "Warning" => LogLevel.Warning,
            "Error" => LogLevel.Error,
            "Critical" => LogLevel.Critical,
            _ => LogLevel.Information
        };
        
        return logLevel >= minLevel;
    }
    
    public IDisposable BeginScope<TState>(TState state) => NullScope.Instance;
    
    private class NullScope : IDisposable
    {
        public static NullScope Instance { get; } = new NullScope();
        public void Dispose() { }
    }
}

public class LogEntry
{
    public DateTime Timestamp { get; set; }
    public string Level { get; set; }
    public string Category { get; set; }
    public string Message { get; set; }
    public string Exception { get; set; }
    public int EventId { get; set; }
}
```

## 6. 面试重点深度解析

### 6.1 高频技术问题

**性能优化深度理解**
- **中间件优化**：如何优化中间件管道的性能
- **依赖注入优化**：如何优化依赖注入的性能
- **配置系统优化**：如何优化配置系统的性能
- **日志系统优化**：如何优化日志系统的性能

### 6.2 架构设计问题

**高性能架构设计**
- **性能监控**：如何设计性能监控系统
- **缓存策略**：如何设计缓存策略
- **异步处理**：如何设计异步处理架构
- **资源管理**：如何优化资源管理

### 6.3 实战案例分析

**企业级应用性能优化案例**
- **高并发系统**：如何优化高并发系统的性能
- **大数据处理**：如何优化大数据处理的性能
- **微服务架构**：如何优化微服务架构的性能
- **云原生应用**：如何优化云原生应用的性能

## 总结

ASP.NET Core 性能优化是一个系统性的工程，要真正掌握性能优化，需要：

1. **深入理解性能原理**：理解性能优化的核心原理和机制
2. **掌握优化策略**：掌握各种性能优化的策略和方法
3. **建立监控体系**：建立完整的性能监控体系
4. **平衡各种因素**：在性能、复杂度、成本之间找到平衡
5. **持续优化改进**：持续优化和改进系统性能

只有深入理解这些原理，才能在面试中展现出真正的技术深度，也才能在项目中做出正确的性能优化决策。
