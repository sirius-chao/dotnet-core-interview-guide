# MVC 架构

## 📚 目录导航

- [MVC 架构设计哲学](#mvc-design-philosophy)
- [控制器](#controller-deep-analysis)
- [模型绑定机制](#model-binding-mechanism)
- [模型验证](#model-validation)
- [过滤器系统](#filter-system)
- [面试常见问题](#interview-questions)

## 🎯 学习目标

深入理解 ASP.NET Core MVC 架构的核心概念和实现原理，掌握面试中的高频考点：

- **MVC 模式**: 理解 Model-View-Controller 的设计哲学和职责分离
- **控制器**: 掌握控制器的生命周期、最佳实践和性能优化
- **模型绑定**: 了解模型绑定的工作流程和自定义绑定器
- **模型验证**: 掌握数据验证的机制和自定义验证器
- **过滤器**: 理解过滤器的执行顺序和自定义过滤器

## 🏗️ MVC 架构设计哲学 {#mvc-design-philosophy}

### MVC 模式的核心理念

MVC（Model-View-Controller）是一种软件架构模式，其核心思想是**职责分离**：

**Model（模型）**：
- 代表数据和业务逻辑
- 不直接与用户交互
- 包含数据验证规则和业务规则
- 独立于用户界面

**View（视图）**：
- 负责数据的展示和用户交互
- 不包含业务逻辑
- 通过模型获取数据
- 可以包含简单的展示逻辑

**Controller（控制器）**：
- 处理用户请求
- 协调模型和视图
- 不直接操作数据
- 决定使用哪个视图

## 🎮 控制器 {#controller-deep-analysis}

### 控制器的生命周期

**控制器的创建和销毁**：
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
        _logger.LogInformation("ProductsController created");
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _logger.LogInformation("ProductsController disposed");
        }
        base.Dispose(disposing);
    }
}
```

### 控制器最佳实践

**异步控制器方法**：
```csharp
[HttpGet]
public async Task<ActionResult<IEnumerable<Product>>> GetProductsAsync(
    [FromQuery] int page = 1, 
    [FromQuery] int pageSize = 10)
{
    try
    {
        var products = await _productService.GetProductsAsync(page, pageSize);
        return Ok(products);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error getting products");
        return StatusCode(500, "Internal server error");
    }
}
```

## 🔗 模型绑定机制 {#model-binding-mechanism}

### 模型绑定的工作流程

模型绑定是将 HTTP 请求数据转换为 .NET 对象的过程，其工作流程包括：

1. **值提供者（Value Provider）**：从不同来源获取原始值
2. **模型绑定器（Model Binder）**：将原始值转换为目标类型
3. **模型验证器（Model Validator）**：验证绑定后的模型
4. **模型状态（ModelState）**：存储绑定和验证结果

**模型绑定的数据源优先级**：
```csharp
[HttpGet("binding")]
public IActionResult TestBinding(
    int id,                    // 路由数据
    [FromQuery] string name,   // 查询字符串
    [FromBody] Product product) // 请求体
{
    return Ok(new { id, name, product });
}
```

## ✅ 模型验证 {#model-validation}

### 数据注解验证

**内置验证特性**：
```csharp
public class Product
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 10000)]
    public decimal Price { get; set; }

    [EmailAddress]
    public string ContactEmail { get; set; }
}
```

**自定义验证特性**：
```csharp
public class ValidProductCodeAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value == null)
        {
            return ValidationResult.Success;
        }

        var productCode = value.ToString();
        
        if (!Regex.IsMatch(productCode, @"^[A-Z]{2}\d{4}$"))
        {
            return new ValidationResult("Invalid product code format");
        }

        return ValidationResult.Success;
    }
}
```

## 🔍 过滤器系统 {#filter-system}

### 过滤器的执行顺序

**过滤器的执行顺序**：
1. **Authorization Filters**：授权过滤器
2. **Resource Filters**：资源过滤器
3. **Action Filters**：Action 过滤器
4. **Exception Filters**：异常过滤器
5. **Result Filters**：结果过滤器

**自定义过滤器示例**：
```csharp
public class LoggingActionFilter : IActionFilter
{
    private readonly ILogger<LoggingActionFilter> _logger;

    public LoggingActionFilter(ILogger<LoggingActionFilter> logger)
    {
        _logger = logger;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        var actionName = context.ActionDescriptor.DisplayName;
        _logger.LogInformation("Action {Action} is executing", actionName);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        var actionName = context.ActionDescriptor.DisplayName;
        _logger.LogInformation("Action {Action} completed", actionName);
    }
}
```

## ❓ 面试常见问题 {#interview-questions}

### Q1: MVC 和 MVVM 有什么区别？

**A**: 主要区别在于：

**1. MVC（Model-View-Controller）**

**Model**：数据和业务逻辑
**View**：界面展示  
**Controller**：处理用户输入（如点击、提交），协调 Model 和 View 的交互

**特点**：
- Controller 负责调度，View 较"被动"
- Model 和 View 通过 Controller 来解耦
- 传统 Web 框架（ASP.NET MVC、Spring MVC）常用这种模式

**⚠️ 注意**：在现代 Web 前端里，很多时候 View 也会直接绑定 Model（例如 React 的 state 就有点 Model 的影子），所以"严格 MVC"不常见。

**2. MVVM（Model-View-ViewModel）**

**Model**：数据和业务逻辑
**View**：界面（XAML、HTML+框架）
**ViewModel**：View 的抽象，提供属性和命令，供 View 绑定

**特点**：
- 双向数据绑定：View ↔ ViewModel ↔ Model
- ViewModel 通过属性暴露数据，View 自动更新；用户操作通过命令绑定回传到 ViewModel
- 减少了 Controller 层的存在感，ViewModel 起到了"Controller+部分Model"的作用

**⚠️ 注意**：MVVM 并不仅限于桌面应用（WPF、UWP、Xamarin），在现代 Web 框架（如 Angular、Vue，甚至 React Hooks+state）也有 MVVM 的思想。

**3. 关键区别总结**

| 特性 | MVC | MVVM |
|------|-----|------|
| 交互方式 | Controller 调度 Model & View | 数据绑定 + 命令绑定 |
| 角色职责 | Controller 负责输入逻辑 | ViewModel 负责状态管理和交互 |
| View 与逻辑的关系 | View 通过 Controller 与 Model 通信 | View 与 ViewModel 双向绑定 |
| 常见应用场景 | Web 后端框架（ASP.NET MVC、Spring MVC） | 桌面/移动端 UI（WPF、Xamarin）、现代前端框架 |

### Q2: 如何优化控制器的性能？

**A**: 通过以下方式优化：

**1. 异步编程优化**
- **使用 async/await**：避免阻塞线程，提高并发性能
- **异步数据库操作**：使用 `ToListAsync()`, `FirstOrDefaultAsync()` 等异步方法
- **并行处理**：使用 `Task.WhenAll()` 并行执行多个独立操作
- **取消令牌支持**：实现 `CancellationToken` 支持，提高响应性

**2. 缓存策略优化**
- **内存缓存**：使用 `IMemoryCache` 缓存频繁访问的数据
- **分布式缓存**：使用 Redis 等分布式缓存，支持多实例共享
- **缓存键设计**：设计合理的缓存键，避免缓存冲突
- **缓存失效策略**：设置合理的 TTL，及时更新过期数据

**3. 数据查询优化**
- **分页处理**：使用 `Skip().Take()` 或游标分页，减少数据传输量
- **延迟加载**：使用 `Include()` 预加载关联数据，避免 N+1 查询
- **投影查询**：使用 `Select()` 只返回需要的字段，减少数据传输
- **查询优化**：使用 `AsNoTracking()` 避免 EF 变更跟踪开销

**4. 响应优化**
- **压缩响应**：启用 Gzip 压缩，减少网络传输
- **ETag 支持**：实现 ETag 缓存，减少重复请求
- **响应缓存**：使用 `[ResponseCache]` 特性缓存响应
- **流式响应**：对于大文件，使用流式响应避免内存占用

**代码示例**：
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly IMemoryCache _cache;
    
    public ProductsController(IProductService productService, IMemoryCache cache)
    {
        _productService = productService;
        _cache = cache;
    }
    
    [HttpGet]
    [ResponseCache(Duration = 300)] // 缓存5分钟
    public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
        [FromQuery] ProductQueryParams queryParams,
        CancellationToken cancellationToken)
    {
        var cacheKey = $"products:{queryParams.GetHashCode()}";
        
        if (_cache.TryGetValue(cacheKey, out PagedResult<ProductDto> cachedResult))
        {
            return Ok(cachedResult);
        }
        
        var result = await _productService.GetProductsAsync(queryParams, cancellationToken);
        
        var cacheOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(10))
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(30));
            
        _cache.Set(cacheKey, result, cacheOptions);
        
        return Ok(result);
    }
}
```

### Q3: 模型绑定的数据源优先级是什么？

**A**: 从高到低的优先级：

**1. 路由数据**：`[FromRoute]` - 最高优先级
- 从 URL 路径中提取参数
- 例如：`/api/products/{id}` 中的 `{id}` 参数
- 适用于资源标识符等关键参数

**2. 查询字符串**：`[FromQuery]` - 第二优先级
- 从 URL 查询参数中提取
- 例如：`/api/products?category=electronics&page=1`
- 适用于过滤、分页、排序等可选参数

**3. 表单数据**：`[FromForm]` - 第三优先级
- 从 HTTP 表单中提取数据
- 适用于 `application/x-www-form-urlencoded` 和 `multipart/form-data`
- 常用于文件上传和表单提交

**4. 请求体**：`[FromBody]` - 第四优先级
- 从 HTTP 请求体中提取 JSON 或 XML 数据
- 适用于复杂对象和大量数据
- 常用于 POST、PUT 等操作

**5. 请求头**：`[FromHeader]` - 最低优先级
- 从 HTTP 请求头中提取数据
- 适用于认证令牌、语言设置等元数据

**自定义绑定示例**：
```csharp
public class ProductQueryModel
{
    [FromRoute]
    public int Id { get; set; }
    
    [FromQuery]
    public string Category { get; set; }
    
    [FromQuery]
    public int Page { get; set; } = 1;
    
    [FromQuery]
    public int PageSize { get; set; } = 10;
    
    [FromHeader(Name = "Accept-Language")]
    public string Language { get; set; }
}

[HttpGet("{id}")]
public async Task<ActionResult<Product>> GetProduct([FromRoute] int id, 
    [FromQuery] string category,
    [FromHeader(Name = "User-Agent")] string userAgent)
{
    // 实现逻辑
}
```

**注意事项**：
- 优先级可以通过显式指定 `[FromXxx]` 特性来覆盖
- 如果没有指定特性，框架会按照默认优先级进行绑定
- 绑定失败会抛出 `ModelBindingException` 或返回验证错误

## 📚 总结

MVC 架构是 ASP.NET Core 的核心，面试中需要深入理解：

1. **架构设计**：理解 MVC 模式的职责分离和优势
2. **控制器优化**：掌握异步编程、缓存、分页等性能优化技巧
3. **模型绑定**：了解数据源优先级和自定义绑定器
4. **模型验证**：掌握数据注解和自定义验证
5. **过滤器系统**：理解执行顺序和自定义过滤器实现

这些知识点不仅要在理论上理解，更要能够用代码示例来说明，在面试中展现出真正的技术深度。
