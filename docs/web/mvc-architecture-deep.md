# MVC 架构深度原理与设计思维

## 1. MVC 架构哲学深度思考

### 1.1 MVC 模式的本质理解

**MVC 不仅仅是代码组织，更是一种关注点分离的哲学思考**：

**MVC 的深层含义**：
- **关注点分离**：
  - **模型（Model）**：专注于数据和业务逻辑
  - **视图（View）**：专注于用户界面和展示
  - **控制器（Controller）**：专注于用户交互和流程控制
  - **职责明确**：每个组件都有明确的职责边界

- **松耦合设计**：
  - **组件解耦**：模型、视图、控制器相互独立
  - **接口解耦**：通过接口定义组件间的交互
  - **依赖倒置**：依赖抽象而不是具体实现
  - **可测试性**：每个组件都可以独立测试

**MVC 的认知层次**：
1. **技术层面**：具体的代码实现和框架使用
2. **设计层面**：解决软件设计问题的通用方案
3. **思维层面**：培养良好的软件架构思维

### 1.2 MVC 架构的历史演进

**架构模式的演进历程**：
- **传统 MVC**：Smalltalk 时代的经典实现
- **Web MVC**：Web 应用中的 MVC 应用
- **现代 MVC**：框架化的 MVC 实现
- **MVC 变体**：MVP、MVVM 等衍生模式

## 2. 模型（Model）层深度设计

### 2.1 模型层的核心职责

**模型层的设计哲学**：
- **数据封装**：封装业务数据和状态
- **业务逻辑**：实现核心业务规则
- **数据验证**：确保数据的有效性和一致性
- **持久化**：管理数据的存储和检索

**模型层的实现策略**：
```csharp
// 领域模型示例
public class Order : Entity<Guid>
{
    private readonly List<OrderItem> _items = new();
    
    public string OrderNumber { get; private set; }
    public Customer Customer { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public Money TotalAmount { get; private set; }
    
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    // 私有构造函数，强制使用工厂方法
    private Order() { }
    
    // 工厂方法创建订单
    public static Order Create(string orderNumber, Customer customer)
    {
        if (string.IsNullOrWhiteSpace(orderNumber))
            throw new ArgumentException("Order number cannot be empty", nameof(orderNumber));
        
        if (customer == null)
            throw new ArgumentException("Customer cannot be null", nameof(customer));
        
        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = orderNumber,
            Customer = customer,
            Status = OrderStatus.Created,
            CreatedAt = DateTime.UtcNow
        };
        
        return order;
    }
    
    // 业务方法：添加订单项
    public void AddItem(Product product, int quantity)
    {
        if (product == null)
            throw new ArgumentException("Product cannot be null", nameof(product));
        
        if (quantity <= 0)
            throw new ArgumentException("Quantity must be positive", nameof(quantity));
        
        if (Status != OrderStatus.Created)
            throw new InvalidOperationException("Cannot add items to non-created order");
        
        var existingItem = _items.FirstOrDefault(i => i.Product.Id == product.Id);
        if (existingItem != null)
        {
            existingItem.UpdateQuantity(existingItem.Quantity + quantity);
        }
        else
        {
            var orderItem = OrderItem.Create(this, product, quantity);
            _items.Add(orderItem);
        }
        
        RecalculateTotal();
    }
    
    // 业务方法：确认订单
    public void Confirm()
    {
        if (Status != OrderStatus.Created)
            throw new InvalidOperationException("Order cannot be confirmed");
        
        if (!_items.Any())
            throw new InvalidOperationException("Cannot confirm empty order");
        
        Status = OrderStatus.Confirmed;
    }
    
    // 业务方法：取消订单
    public void Cancel()
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new InvalidOperationException("Cannot cancel shipped or delivered order");
        
        Status = OrderStatus.Cancelled;
    }
    
    private void RecalculateTotal()
    {
        TotalAmount = _items.Sum(item => item.TotalPrice);
    }
}

// 值对象示例
public class Money : ValueObject<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));
        
        if (string.IsNullOrWhiteSpace(currency))
            throw new ArgumentException("Currency cannot be empty", nameof(currency));
        
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }
    
    public static Money Zero(string currency) => new Money(0, currency);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add money with different currencies");
        
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Multiply(decimal factor)
    {
        if (factor < 0)
            throw new ArgumentException("Factor cannot be negative", nameof(factor));
        
        return new Money(Amount * factor, Currency);
    }
    
    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
    
    public override string ToString() => $"{Amount:F2} {Currency}";
}

// 实体基类
public abstract class Entity<TId> : IEquatable<Entity<TId>>
{
    public TId Id { get; protected set; }
    
    protected Entity() { }
    
    protected Entity(TId id)
    {
        Id = id;
    }
    
    public override bool Equals(object obj)
    {
        return Equals(obj as Entity<TId>);
    }
    
    public bool Equals(Entity<TId> other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return EqualityComparer<TId>.Default.Equals(Id, other.Id);
    }
    
    public override int GetHashCode()
    {
        return Id.GetHashCode();
    }
    
    public static bool operator ==(Entity<TId> left, Entity<TId> right)
    {
        return EqualityComparer<Entity<TId>>.Default.Equals(left, right);
    }
    
    public static bool operator !=(Entity<TId> left, Entity<TId> right)
    {
        return !(left == right);
    }
}

// 值对象基类
public abstract class ValueObject<T> : IEquatable<T> where T : ValueObject<T>
{
    public override bool Equals(object obj)
    {
        return Equals(obj as T);
    }
    
    public bool Equals(T other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }
    
    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Select(x => x != null ? x.GetHashCode() : 0)
            .Aggregate((x, y) => x ^ y);
    }
    
    protected abstract IEnumerable<object> GetEqualityComponents();
    
    public static bool operator ==(ValueObject<T> left, ValueObject<T> right)
    {
        return EqualityComparer<T>.Default.Equals(left, right);
    }
    
    public static bool operator !=(ValueObject<T> left, ValueObject<T> right)
    {
        return !(left == right);
    }
}
```

### 2.2 模型验证与业务规则

**模型验证策略**：
```csharp
// 模型验证器示例
public class OrderValidator : AbstractValidator<Order>
{
    public OrderValidator()
    {
        RuleFor(x => x.OrderNumber)
            .NotEmpty()
            .Matches(@"^ORD-\d{8}-\d{4}$")
            .WithMessage("Order number must match pattern ORD-YYYYMMDD-XXXX");
        
        RuleFor(x => x.Customer)
            .NotNull()
            .WithMessage("Customer is required");
        
        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item");
        
        RuleFor(x => x.TotalAmount)
            .GreaterThan(Money.Zero("USD"))
            .WithMessage("Order total must be greater than zero");
        
        // 自定义验证规则
        RuleFor(x => x)
            .Must(HaveValidCustomerStatus)
            .WithMessage("Customer must be active to place orders");
        
        RuleFor(x => x)
            .Must(HaveValidProductAvailability)
            .WithMessage("All products must be available");
    }
    
    private bool HaveValidCustomerStatus(Order order)
    {
        return order.Customer.Status == CustomerStatus.Active;
    }
    
    private bool HaveValidProductAvailability(Order order)
    {
        return order.Items.All(item => item.Product.IsAvailable);
    }
}

// 业务规则引擎示例
public class BusinessRuleEngine
{
    private readonly List<IBusinessRule> _rules = new();
    
    public void AddRule(IBusinessRule rule)
    {
        _rules.Add(rule);
    }
    
    public BusinessRuleResult Validate(object entity)
    {
        var violations = new List<BusinessRuleViolation>();
        
        foreach (var rule in _rules)
        {
            if (!rule.IsSatisfiedBy(entity))
            {
                violations.Add(new BusinessRuleViolation(rule.GetType().Name, rule.GetMessage()));
            }
        }
        
        return new BusinessRuleResult(violations);
    }
}

public interface IBusinessRule
{
    bool IsSatisfiedBy(object entity);
    string GetMessage();
}

public class BusinessRuleResult
{
    public bool IsValid => !Violations.Any();
    public IReadOnlyList<BusinessRuleViolation> Violations { get; }
    
    public BusinessRuleResult(IEnumerable<BusinessRuleViolation> violations)
    {
        Violations = violations.ToList().AsReadOnly();
    }
}

public class BusinessRuleViolation
{
    public string RuleName { get; }
    public string Message { get; }
    
    public BusinessRuleViolation(string ruleName, string message)
    {
        RuleName = ruleName;
        Message = message;
    }
}
```

## 3. 视图（View）层深度设计

### 3.1 视图层的设计原则

**视图层的核心职责**：
- **数据展示**：将模型数据以用户友好的方式展示
- **用户交互**：处理用户的输入和操作
- **界面布局**：管理页面的结构和样式
- **响应式设计**：适配不同的设备和屏幕尺寸

**视图层的实现策略**：
```csharp
// 强类型视图模型示例
public class OrderViewModel
{
    public Guid Id { get; set; }
    public string OrderNumber { get; set; }
    public string CustomerName { get; set; }
    public string CustomerEmail { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public decimal TotalAmount { get; set; }
    public string Currency { get; set; }
    public List<OrderItemViewModel> Items { get; set; } = new();
    public string StatusDisplay => Status.GetDisplayName();
    public string FormattedTotal => $"{TotalAmount:F2} {Currency}";
    public bool CanCancel => Status == OrderStatus.Created || Status == OrderStatus.Confirmed;
    public bool CanConfirm => Status == OrderStatus.Created;
    
    // 视图逻辑方法
    public string GetStatusCssClass()
    {
        return Status switch
        {
            OrderStatus.Created => "badge badge-primary",
            OrderStatus.Confirmed => "badge badge-info",
            OrderStatus.Shipped => "badge badge-warning",
            OrderStatus.Delivered => "badge badge-success",
            OrderStatus.Cancelled => "badge badge-danger",
            _ => "badge badge-secondary"
        };
    }
    
    public bool HasItems => Items.Any();
    public int ItemCount => Items.Count;
}

public class OrderItemViewModel
{
    public Guid ProductId { get; set; }
    public string ProductName { get; set; }
    public string ProductImage { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal TotalPrice { get; set; }
    public string FormattedUnitPrice => $"{UnitPrice:F2}";
    public string FormattedTotalPrice => $"{TotalPrice:F2}";
}

// 视图组件示例
public class OrderSummaryViewComponent : ViewComponent
{
    private readonly IOrderService _orderService;
    
    public OrderSummaryViewComponent(IOrderService orderService)
    {
        _orderService = orderService;
    }
    
    public async Task<IViewComponentResult> InvokeAsync(string customerId)
    {
        var orders = await _orderService.GetCustomerOrdersAsync(customerId);
        
        var viewModel = new OrderSummaryViewModel
        {
            TotalOrders = orders.Count,
            PendingOrders = orders.Count(o => o.Status == OrderStatus.Created),
            ConfirmedOrders = orders.Count(o => o.Status == OrderStatus.Confirmed),
            ShippedOrders = orders.Count(o => o.Status == OrderStatus.Shipped),
            DeliveredOrders = orders.Count(o => o.Status == OrderStatus.Delivered),
            CancelledOrders = orders.Count(o => o.Status == OrderStatus.Cancelled),
            TotalValue = orders.Sum(o => o.TotalAmount.Amount)
        };
        
        return View(viewModel);
    }
}

// 部分视图示例
public class OrderDetailsPartialViewModel
{
    public OrderViewModel Order { get; set; }
    public bool ShowActions { get; set; } = true;
    public bool ShowCustomerInfo { get; set; } = true;
    public bool ShowOrderHistory { get; set; } = false;
}

// 视图数据提供者
public interface IViewDataProvider<T>
{
    Task<T> GetViewDataAsync(object parameters);
}

public class OrderViewDataProvider : IViewDataProvider<OrderViewModel>
{
    private readonly IOrderService _orderService;
    private readonly IMapper _mapper;
    
    public OrderViewDataProvider(IOrderService orderService, IMapper mapper)
    {
        _orderService = orderService;
        _mapper = mapper;
    }
    
    public async Task<OrderViewModel> GetViewDataAsync(object parameters)
    {
        if (parameters is not Guid orderId)
            throw new ArgumentException("Parameters must be order ID", nameof(parameters));
        
        var order = await _orderService.GetOrderByIdAsync(orderId);
        if (order == null)
            throw new InvalidOperationException($"Order with ID {orderId} not found");
        
        return _mapper.Map<OrderViewModel>(order);
    }
}
```

### 3.2 视图引擎与模板系统

**视图引擎深度实现**：
```csharp
// 自定义视图引擎示例
public class CustomViewEngine : IViewEngine
{
    private readonly string[] _viewLocationFormats = {
        "~/Views/{1}/{0}.cshtml",
        "~/Views/Shared/{0}.cshtml",
        "~/Views/{1}/{0}.html",
        "~/Views/Shared/{0}.html"
    };
    
    public ViewEngineResult FindView(ActionContext context, string viewName, bool isMainPage)
    {
        var searchedLocations = new List<string>();
        
        foreach (var format in _viewLocationFormats)
        {
            var viewPath = string.Format(format, viewName, context.RouteData.Values["controller"]);
            
            if (File.Exists(Path.Combine(context.HttpContext.Request.PathBase, viewPath.TrimStart('~', '/')))
            {
                return ViewEngineResult.Found(viewName, new CustomView(viewPath));
            }
            
            searchedLocations.Add(viewPath);
        }
        
        return ViewEngineResult.NotFound(viewName, searchedLocations);
    }
    
    public ViewEngineResult FindView(ActionContext context, string viewName, bool isMainPage)
    {
        return FindView(context, viewName, isMainPage);
    }
}

public class CustomView : IView
{
    private readonly string _viewPath;
    
    public CustomView(string viewPath)
    {
        _viewPath = viewPath;
    }
    
    public string Path => _viewPath;
    
    public async Task RenderAsync(ViewContext context)
    {
        var viewContent = await File.ReadAllTextAsync(_viewPath);
        
        // 简单的模板引擎实现
        var processedContent = ProcessTemplate(viewContent, context.ViewData);
        
        await context.Writer.WriteAsync(processedContent);
    }
    
    private string ProcessTemplate(string template, ViewDataDictionary viewData)
    {
        // 简单的变量替换
        foreach (var item in viewData)
        {
            template = template.Replace($"{{{{{item.Key}}}}}", item.Value?.ToString() ?? "");
        }
        
        return template;
    }
}

// 视图结果包装器
public class ViewResultWrapper : IActionResult
{
    private readonly object _model;
    private readonly string _viewName;
    private readonly ViewDataDictionary _viewData;
    
    public ViewResultWrapper(object model, string viewName = null)
    {
        _model = model;
        _viewName = viewName;
        _viewData = new ViewDataDictionary(new EmptyModelMetadataProvider(), new ModelStateDictionary())
        {
            Model = model
        };
    }
    
    public async Task ExecuteResultAsync(ActionContext context)
    {
        var viewEngine = context.HttpContext.RequestServices.GetRequiredService<ICompositeViewEngine>();
        var viewEngineResult = viewEngine.FindView(context, _viewName ?? GetDefaultViewName(context), true);
        
        if (!viewEngineResult.Success)
        {
            throw new InvalidOperationException($"View '{_viewName}' not found");
        }
        
        var view = viewEngineResult.View;
        var viewContext = new ViewContext(context, view, _viewData, new TempDataDictionary(context.HttpContext, context.HttpContext.RequestServices.GetRequiredService<ITempDataProvider>()), context.Writer, new HtmlHelperOptions());
        
        await view.RenderAsync(viewContext);
    }
    
    private string GetDefaultViewName(ActionContext context)
    {
        var actionName = context.RouteData.Values["action"]?.ToString();
        return actionName ?? "Index";
    }
}
```

## 4. 控制器（Controller）层深度设计

### 4.1 控制器的设计原则

**控制器的核心职责**：
- **请求处理**：接收和处理 HTTP 请求
- **业务协调**：协调模型和视图的交互
- **流程控制**：控制应用的执行流程
- **响应生成**：生成适当的 HTTP 响应

**控制器的实现策略**：
```csharp
// 基础控制器示例
[ApiController]
[Route("api/[controller]")]
public abstract class BaseController : ControllerBase
{
    protected readonly ILogger _logger;
    protected readonly IMediator _mediator;
    
    protected BaseController(ILogger logger, IMediator mediator)
    {
        _logger = logger;
        _mediator = mediator;
    }
    
    // 统一的成功响应
    protected ActionResult<T> Success<T>(T data, string message = "Success")
    {
        return Ok(new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Message = message
        });
    }
    
    // 统一的错误响应
    protected ActionResult<T> Error<T>(string message, int statusCode = 400)
    {
        return StatusCode(statusCode, new ApiResponse<T>
        {
            Success = false,
            Message = message
        });
    }
    
    // 异常处理
    protected ActionResult<T> HandleException<T>(Exception ex)
    {
        _logger.LogError(ex, "An error occurred while processing the request");
        
        return ex switch
        {
            ValidationException validationEx => BadRequest(new ApiResponse<T>
            {
                Success = false,
                Message = "Validation failed",
                Errors = validationEx.Errors.Select(e => e.ErrorMessage).ToList()
            }),
            
            NotFoundException notFoundEx => NotFound(new ApiResponse<T>
            {
                Success = false,
                Message = notFoundEx.Message
            }),
            
            UnauthorizedAccessException => Unauthorized(new ApiResponse<T>
            {
                Success = false,
                Message = "Access denied"
            }),
            
            _ => StatusCode(500, new ApiResponse<T>
            {
                Success = false,
                Message = "An internal error occurred"
            })
        };
    }
}

// 订单控制器示例
[ApiController]
[Route("api/[controller]")]
public class OrdersController : BaseController
{
    private readonly IOrderService _orderService;
    private readonly IOrderValidator _orderValidator;
    
    public OrdersController(
        ILogger<OrdersController> logger,
        IMediator mediator,
        IOrderService orderService,
        IOrderValidator orderValidator) : base(logger, mediator)
    {
        _orderService = orderService;
        _orderValidator = orderValidator;
    }
    
    // GET: api/orders
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<List<OrderViewModel>>), 200)]
    public async Task<ActionResult<ApiResponse<List<OrderViewModel>>>> GetOrders(
        [FromQuery] OrderQueryParameters parameters)
    {
        try
        {
            var orders = await _orderService.GetOrdersAsync(parameters);
            var viewModels = _mapper.Map<List<OrderViewModel>>(orders);
            
            return Success(viewModels);
        }
        catch (Exception ex)
        {
            return HandleException<List<OrderViewModel>>(ex);
        }
    }
    
    // GET: api/orders/{id}
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<OrderViewModel>), 200)]
    [ProducesResponseType(typeof(ApiResponse<object>), 404)]
    public async Task<ActionResult<ApiResponse<OrderViewModel>>> GetOrder(Guid id)
    {
        try
        {
            var order = await _orderService.GetOrderByIdAsync(id);
            if (order == null)
                return Error<OrderViewModel>("Order not found", 404);
            
            var viewModel = _mapper.Map<OrderViewModel>(order);
            return Success(viewModel);
        }
        catch (Exception ex)
        {
            return HandleException<OrderViewModel>(ex);
        }
    }
    
    // POST: api/orders
    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<OrderViewModel>), 201)]
    [ProducesResponseType(typeof(ApiResponse<object>), 400)]
    public async Task<ActionResult<ApiResponse<OrderViewModel>>> CreateOrder(
        [FromBody] CreateOrderRequest request)
    {
        try
        {
            // 验证请求
            var validationResult = await _orderValidator.ValidateAsync(request);
            if (!validationResult.IsValid)
            {
                return Error<OrderViewModel>("Validation failed", 400);
            }
            
            // 创建订单命令
            var command = new CreateOrderCommand
            {
                CustomerId = request.CustomerId,
                Items = request.Items.Select(i => new OrderItemDto
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity
                }).ToList()
            };
            
            // 发送命令
            var orderId = await _mediator.Send(command);
            
            // 获取创建的订单
            var order = await _orderService.GetOrderByIdAsync(orderId);
            var viewModel = _mapper.Map<OrderViewModel>(order);
            
            return CreatedAtAction(nameof(GetOrder), new { id = orderId }, 
                Success(viewModel, "Order created successfully"));
        }
        catch (Exception ex)
        {
            return HandleException<OrderViewModel>(ex);
        }
    }
    
    // PUT: api/orders/{id}/confirm
    [HttpPut("{id:guid}/confirm")]
    [ProducesResponseType(typeof(ApiResponse<object>), 200)]
    [ProducesResponseType(typeof(ApiResponse<object>), 400)]
    public async Task<ActionResult<ApiResponse<object>>> ConfirmOrder(Guid id)
    {
        try
        {
            var command = new ConfirmOrderCommand { OrderId = id };
            await _mediator.Send(command);
            
            return Success<object>(null, "Order confirmed successfully");
        }
        catch (Exception ex)
        {
            return HandleException<object>(ex);
        }
    }
    
    // DELETE: api/orders/{id}
    [HttpDelete("{id:guid}")]
    [ProducesResponseType(typeof(ApiResponse<object>), 200)]
    [ProducesResponseType(typeof(ApiResponse<object>), 400)]
    public async Task<ActionResult<ApiResponse<object>>> CancelOrder(Guid id)
    {
        try
        {
            var command = new CancelOrderCommand { OrderId = id };
            await _mediator.Send(command);
            
            return Success<object>(null, "Order cancelled successfully");
        }
        catch (Exception ex)
        {
            return HandleException<object>(ex);
        }
    }
}

// 查询参数模型
public class OrderQueryParameters
{
    [Range(1, 100)]
    public int Page { get; set; } = 1;
    
    [Range(1, 1000)]
    public int PageSize { get; set; } = 20;
    
    public string? CustomerName { get; set; }
    public OrderStatus? Status { get; set; }
    public DateTime? CreatedFrom { get; set; }
    public DateTime? CreatedTo { get; set; }
    public string? SortBy { get; set; }
    public bool SortDescending { get; set; } = false;
}

// API 响应模型
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public string Message { get; set; } = string.Empty;
    public List<string>? Errors { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}
```

### 4.2 控制器性能优化

**性能优化策略**：
```csharp
// 异步控制器示例
[ApiController]
[Route("api/[controller]")]
public class AsyncOrdersController : BaseController
{
    private readonly IOrderService _orderService;
    private readonly IMemoryCache _cache;
    private readonly IBackgroundJobClient _backgroundJobs;
    
    public AsyncOrdersController(
        ILogger<AsyncOrdersController> logger,
        IMediator mediator,
        IOrderService orderService,
        IMemoryCache cache,
        IBackgroundJobClient backgroundJobs) : base(logger, mediator)
    {
        _orderService = orderService;
        _cache = cache;
        _backgroundJobs = backgroundJobs;
    }
    
    // 缓存优化的查询
    [HttpGet("cached")]
    [ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
    public async Task<ActionResult<ApiResponse<List<OrderViewModel>>>> GetCachedOrders(
        [FromQuery] OrderQueryParameters parameters)
    {
        var cacheKey = $"orders_{JsonSerializer.Serialize(parameters)}";
        
        if (_cache.TryGetValue(cacheKey, out List<OrderViewModel> cachedOrders))
        {
            return Success(cachedOrders, "Retrieved from cache");
        }
        
        var orders = await _orderService.GetOrdersAsync(parameters);
        var viewModels = _mapper.Map<List<OrderViewModel>>(orders);
        
        // 设置缓存，包含过期策略
        var cacheOptions = new MemoryCacheEntryOptions()
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(5))
            .SetSlidingExpiration(TimeSpan.FromMinutes(2))
            .RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogInformation("Cache entry {Key} was evicted due to {Reason}", key, reason);
            });
        
        _cache.Set(cacheKey, viewModels, cacheOptions);
        
        return Success(viewModels);
    }
    
    // 后台任务处理
    [HttpPost("bulk")]
    public async Task<ActionResult<ApiResponse<string>>> CreateBulkOrders(
        [FromBody] List<CreateOrderRequest> requests)
    {
        if (requests.Count > 1000)
        {
            return Error<string>("Bulk order limit is 1000 orders", 400);
        }
        
        // 启动后台任务
        var jobId = _backgroundJobs.Enqueue<IBulkOrderProcessor>(
            processor => processor.ProcessBulkOrders(requests));
        
        return Success(jobId, "Bulk order processing started");
    }
    
    // 流式响应
    [HttpGet("stream")]
    public async IAsyncEnumerable<OrderViewModel> GetOrdersStream(
        [FromQuery] OrderQueryParameters parameters)
    {
        await foreach (var order in _orderService.GetOrdersStreamAsync(parameters))
        {
            var viewModel = _mapper.Map<OrderViewModel>(order);
            yield return viewModel;
        }
    }
}

// 后台任务处理器
public interface IBulkOrderProcessor
{
    Task ProcessBulkOrders(List<CreateOrderRequest> requests);
}

public class BulkOrderProcessor : IBulkOrderProcessor
{
    private readonly IOrderService _orderService;
    private readonly ILogger<BulkOrderProcessor> _logger;
    
    public BulkOrderProcessor(IOrderService orderService, ILogger<BulkOrderProcessor> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    public async Task ProcessBulkOrders(List<CreateOrderRequest> requests)
    {
        _logger.LogInformation("Starting bulk order processing for {Count} orders", requests.Count);
        
        var batchSize = 100;
        var batches = requests.Chunk(batchSize);
        
        foreach (var batch in batches)
        {
            try
            {
                await ProcessBatch(batch);
                _logger.LogInformation("Processed batch of {Count} orders", batch.Length);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing batch of orders");
            }
        }
        
        _logger.LogInformation("Completed bulk order processing");
    }
    
    private async Task ProcessBatch(CreateOrderRequest[] batch)
    {
        var tasks = batch.Select(request => _orderService.CreateOrderAsync(request));
        await Task.WhenAll(tasks);
    }
}
```

## 5. MVC 架构优化策略

### 5.1 架构层面的优化

**架构优化策略**：
- **分层优化**：明确各层的职责边界
- **接口设计**：设计稳定的接口契约
- **依赖管理**：合理管理组件间的依赖关系
- **性能优化**：在各个层面实施性能优化

### 5.2 设计模式应用

**设计模式在 MVC 中的应用**：
- **工厂模式**：创建复杂的对象实例
- **策略模式**：实现不同的业务策略
- **观察者模式**：实现事件驱动的架构
- **命令模式**：处理用户操作和业务命令

## 6. 面试重点深度解析

### 6.1 高频技术问题

**MVC 架构深度理解**
- **架构设计原则**：深入理解 MVC 的设计思想和原则
- **组件职责分离**：掌握模型、视图、控制器的职责边界
- **数据流管理**：理解数据在 MVC 组件间的流转
- **性能优化策略**：掌握 MVC 架构的性能优化方法

### 6.2 架构设计问题

**企业级 MVC 应用设计**
- **大型应用架构**：如何设计大型 MVC 应用
- **性能优化**：如何优化 MVC 应用的性能
- **可维护性**：如何提高 MVC 应用的可维护性
- **可扩展性**：如何设计可扩展的 MVC 架构

### 6.3 实战案例分析

**电商系统 MVC 架构案例**
- **商品管理**：如何设计商品管理的 MVC 架构
- **订单处理**：如何设计订单处理的 MVC 架构
- **用户管理**：如何设计用户管理的 MVC 架构
- **支付系统**：如何设计支付系统的 MVC 架构

## 总结

MVC 架构是软件设计的重要模式，要真正掌握 MVC 架构，需要：

1. **深入理解架构原理**：理解 MVC 的核心思想和设计原则
2. **掌握实现技术**：掌握各种 MVC 框架的实现技术
3. **应用设计模式**：在实际项目中应用 MVC 架构
4. **持续优化改进**：不断优化和改进 MVC 架构
5. **性能调优**：在实际项目中验证和优化性能

只有深入理解这些原理，才能在面试中展现出真正的技术深度，也才能在项目中做出正确的架构设计决策。
