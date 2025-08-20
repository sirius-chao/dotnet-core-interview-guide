# .NET Core 框架面试指南 🚀

## 📚 快速导航
- [面试高频问题](#面试高频问题)
- [依赖注入深度原理](#1-依赖注入-dependency-injection)
- [中间件架构](#2-中间件-middleware)
- [配置系统](#3-配置系统)
- [日志系统](#4-日志系统)
- [性能优化](#5-性能优化策略)

## ❓ 面试高频问题

### Q1: 依赖注入的生命周期如何选择？

**面试官想了解什么**：你对DI容器工作原理的理解，以及在实际项目中的使用经验。

**🎯 标准答案**：

| 生命周期 | 创建时机 | 销毁时机 | 适用场景 | 注意事项 |
|----------|----------|----------|----------|----------|
| **Singleton** | 应用启动时 | 应用关闭时 | 配置服务、日志服务 | 线程安全、无状态 |
| **Scoped** | 请求开始时 | 请求结束时 | 数据库上下文、业务服务 | 每个请求独立 |
| **Transient** | 每次注入时 | 使用完毕后 | 工具类、辅助服务 | 轻量级、无状态 |

**💡 面试加分点**：提到"我会根据服务的状态性和资源消耗来选择生命周期"

---

### Q2: 中间件的执行顺序如何控制？

**面试官想了解什么**：你对管道架构的理解。

**🎯 标准答案**：

**中间件管道执行顺序**：
```
请求 → 中间件1 → 中间件2 → 中间件3 → 端点 → 中间件3 → 中间件2 → 中间件1 → 响应
```

**控制方式**：
1. **注册顺序**：先注册的先执行
2. **UseWhen**：条件性执行
3. **Map**：路径映射
4. **Use**：无条件执行

**💡 面试加分点**：提到"我会使用中间件来记录请求日志、处理异常、添加安全头"

---

### Q3: 如何优化 .NET Core 应用性能？

**面试官想了解什么**：你的性能优化经验。

**🎯 标准答案**：

| 优化方向 | 具体措施 | 预期效果 |
|----------|----------|----------|
| **启动性能** | 延迟加载、AOT编译 | 减少启动时间 |
| **内存优化** | 对象池、Span<T> | 减少GC压力 |
| **网络性能** | HTTP/2、连接复用 | 提高吞吐量 |
| **数据库** | 连接池、查询优化 | 减少延迟 |

**💡 面试加分点**：提到"我会使用性能分析工具，如dotnet-trace和dotnet-counters"

---

## 🏗️ 实战场景分析

### 场景1：微服务架构设计

**业务需求**：设计支持100+微服务的企业级系统

**🎯 技术方案**：

```
客户端 → API网关 → 服务发现 → 微服务集群 → 数据库集群
   ↓         ↓         ↓          ↓           ↓
 负载均衡  路由转发  健康检查   服务注册    读写分离
```

**核心实现**：
1. **服务注册**：使用Consul或Eureka
2. **配置管理**：使用配置中心
3. **链路追踪**：集成Jaeger或Zipkin
4. **熔断降级**：使用Polly库

**🔑 关键决策**：选择Scoped生命周期，确保每个请求的服务实例独立

---

### 场景2：高并发Web应用

**业务需求**：支持10万+并发用户的电商系统

**🎯 技术方案**：

```
用户请求 → 负载均衡 → 应用集群 → 缓存层 → 数据库
   ↓         ↓          ↓         ↓         ↓
  CDN      Nginx      .NET Core  Redis    SQL Server
```

**核心实现**：
1. **异步编程**：全面使用async/await
2. **缓存策略**：多级缓存架构
3. **数据库优化**：读写分离、分库分表
4. **监控告警**：集成Prometheus和Grafana

---

## 🚀 技术要点总结

### 依赖注入核心概念

**服务生命周期选择指南**：
| 生命周期 | 适用场景 | 优势 | 注意事项 |
|----------|----------|------|----------|
| **Singleton** | 配置服务、日志服务、缓存服务 | 性能最好，内存效率高 | 避免状态，注意线程安全 |
| **Scoped** | 数据库上下文、业务服务、用户会话 | 请求内共享，资源合理 | 避免跨请求使用 |
| **Transient** | 工具类、辅助服务、轻量级服务 | 简单直接，无状态 | 避免频繁创建复杂对象 |

**服务注册最佳实践**：
```csharp
// ✅ 推荐：接口注册，便于测试和扩展
services.AddScoped<IRepository, Repository>();
services.AddScoped<IUserService, UserService>();

// ✅ 推荐：工厂模式，复杂对象创建
services.AddScoped<IDbConnection>(sp => 
    new SqlConnection(sp.GetRequiredService<IConfiguration>().GetConnectionString("Default")));

// ✅ 推荐：选项模式，配置注入
services.Configure<DatabaseOptions>(configuration.GetSection("Database"));
services.AddScoped<IRepository>(sp => 
    new Repository(sp.GetRequiredService<IOptions<DatabaseOptions>>().Value));

// ❌ 避免：直接注册具体类型
services.AddScoped<Repository>(); // 难以测试和扩展
```

---

## 🔧 实战应用指南

### 场景1：微服务架构设计

**业务需求**：构建可扩展的微服务架构，支持服务发现和负载均衡

**🎯 技术方案**：
```
服务注册 → 服务发现 → 负载均衡 → 服务调用 → 健康检查
    ↓         ↓         ↓         ↓         ↓
  服务注册   自动发现   智能路由     远程调用    状态监控
```

**核心实现**：
1. **服务注册**：使用Consul或Eureka进行服务注册
2. **服务发现**：实现服务发现客户端
3. **负载均衡**：集成负载均衡器（如Nginx、HAProxy）
4. **健康检查**：实现健康检查端点

**依赖注入配置**：
```csharp
// 微服务配置
services.Configure<ServiceDiscoveryOptions>(configuration.GetSection("ServiceDiscovery"));
services.AddScoped<IServiceDiscovery, ConsulServiceDiscovery>();
services.AddScoped<ILoadBalancer, RoundRobinLoadBalancer>();

// 服务客户端工厂
services.AddHttpClient<IApiClient, ApiClient>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```

### 场景2：多租户系统设计

**业务需求**：支持多租户隔离，每个租户有独立的数据和配置

**🎯 技术方案**：
```
租户识别 → 上下文注入 → 数据隔离 → 配置隔离 → 权限控制
    ↓         ↓         ↓         ↓         ↓
  请求解析   依赖注入   数据过滤     配置切换    访问控制
```

**核心实现**：
1. **租户识别**：从请求头、子域名或路径中识别租户
2. **上下文注入**：将租户信息注入到依赖注入容器
3. **数据隔离**：在数据访问层实现租户过滤
4. **配置隔离**：为每个租户提供独立的配置

**依赖注入配置**：
```csharp
// 租户上下文服务
services.AddScoped<ITenantContext, TenantContext>();
services.AddScoped<ITenantResolver, HeaderTenantResolver>();

// 租户特定的服务注册
services.AddScoped<ITenantService>(sp =>
{
    var tenantContext = sp.GetRequiredService<ITenantContext>();
    var tenantId = tenantContext.CurrentTenantId;
    
    // 根据租户ID创建特定服务
    return new TenantService(tenantId, sp.GetRequiredService<IConfiguration>());
});
```

---

## 📊 性能优化深度指南

### 服务生命周期优化

**性能影响分析**：
| 生命周期 | 内存使用 | 创建开销 | GC压力 | 适用场景 |
|----------|----------|----------|--------|----------|
| **Singleton** | 最低 | 一次 | 最低 | 无状态服务 |
| **Scoped** | 中等 | 每请求 | 中等 | 有状态服务 |
| **Transient** | 最高 | 每次 | 最高 | 轻量服务 |

**优化策略**：
```csharp
// 使用对象池减少Transient服务的创建开销
services.AddSingleton<ObjectPool<ExpensiveObject>>(sp =>
{
    var policy = new ExpensiveObjectPooledObjectPolicy();
    return new DefaultObjectPool<ExpensiveObject>(policy, 10);
});

services.AddTransient<IExpensiveService>(sp =>
{
    var pool = sp.GetRequiredService<ObjectPool<ExpensiveObject>>();
    return new ExpensiveService(pool);
});

// 使用工厂模式延迟创建
services.AddSingleton<IFactory<ILazyService>, LazyServiceFactory>();
services.AddTransient<ILazyService>(sp =>
{
    var factory = sp.GetRequiredService<IFactory<ILazyService>>();
    return factory.Create();
});
```

### 依赖注入性能优化

**服务解析优化**：
```csharp
// 使用ServiceProvider优化服务解析
public class OptimizedService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly Dictionary<Type, object> _serviceCache = new();

    public OptimizedService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public T GetService<T>() where T : class
    {
        var type = typeof(T);
        if (_serviceCache.TryGetValue(type, out var cached))
        {
            return (T)cached;
        }

        var service = _serviceProvider.GetRequiredService<T>();
        _serviceCache[type] = service;
        return service;
    }
}

// 注册优化服务
services.AddScoped<IOptimizedService, OptimizedService>();
```

---

## 🎯 面试重点总结

### 高频技术问题

**Q1: 在Scoped服务中注入Singleton服务会有什么问题？**

**🎯 标准答案**：
- **生命周期不匹配**：Singleton服务在整个应用生命周期中存在，而Scoped服务只在请求期间存在
- **状态污染**：如果Singleton服务有状态，可能被多个请求共享，导致数据混乱
- **资源泄漏**：Scoped服务可能持有Singleton服务的引用，导致资源无法及时释放

**💡 面试加分点**：提到"我会使用依赖注入验证工具检查生命周期配置，确保服务注册的正确性"

**Q2: 如何实现配置热重载？**

**🎯 标准答案**：
- 使用IOptionsMonitor<T>监听配置变化
- 实现配置变更通知机制
- 使用配置源支持热重载（如文件配置、环境变量）

**💡 面试加分点**：提到"我会实现配置验证和变更日志，确保配置变更的安全性和可追溯性"

### 实战经验展示

**项目案例**：高并发电商系统架构优化

**技术挑战**：支持10万+并发用户，要求响应时间<200ms

**解决方案**：
1. 使用Scoped生命周期管理用户会话和业务上下文
2. 实现服务工厂模式，支持动态服务创建
3. 使用对象池优化频繁创建的对象
4. 实现配置热重载，支持动态配置调整

**性能提升**：系统响应时间从500ms降低到150ms，并发处理能力提升5倍

---

## 总结

.NET Core 依赖注入是构建可维护、可测试应用的核心技术，要真正掌握这些技术，需要：

1. **深入理解生命周期**：掌握Singleton、Scoped、Transient的区别和适用场景
2. **掌握服务注册模式**：理解接口注册、工厂模式、选项模式等最佳实践
3. **理解性能优化**：掌握对象池、延迟创建、服务缓存等优化策略
4. **掌握实战应用**：能够将依赖注入应用到实际项目架构中
5. **理解设计原则**：掌握控制反转、依赖倒置等设计原则

只有深入理解这些核心技术，才能在面试中展现出真正的技术深度，也才能在项目中构建出高质量、高性能的应用架构。
