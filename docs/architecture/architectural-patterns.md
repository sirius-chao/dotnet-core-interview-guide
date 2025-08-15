# 架构模式

## 1. 分层架构 (Layered Architecture)

### 1.1 经典三层架构

**分层架构的设计哲学**
分层架构是一种将应用程序按照职责和关注点进行垂直分割的架构模式。每一层都有明确的职责，层与层之间通过定义良好的接口进行通信，实现了关注点分离和模块化设计。

**分层架构的核心原则**：
1. **单一职责**：每一层只负责特定的功能领域
2. **依赖方向**：依赖关系只能从上到下，上层依赖下层
3. **接口隔离**：层与层之间通过接口进行交互，降低耦合度
4. **可替换性**：每一层都可以独立替换，不影响其他层
5. **可测试性**：每一层都可以独立进行单元测试

**经典三层架构的组成**：
- **表现层（Presentation Layer）**：负责用户界面和用户交互，处理HTTP请求和响应
- **业务逻辑层（Business Logic Layer）**：包含业务规则、验证逻辑和业务流程控制
- **数据访问层（Data Access Layer）**：负责数据持久化，与数据库进行交互

**分层架构的优势**：
- **可维护性**：代码结构清晰，便于理解和维护
- **可扩展性**：可以独立扩展某一层的功能
- **可重用性**：业务逻辑层和数据访问层可以被多个表现层使用
- **可测试性**：每一层都可以独立测试，提高测试覆盖率
- **团队协作**：不同团队可以并行开发不同的层

**分层架构的挑战**：
- **性能开销**：层与层之间的调用可能带来性能开销
- **复杂性增加**：对于简单应用，分层可能增加不必要的复杂性
- **过度设计**：需要根据项目规模合理选择分层粒度
- **依赖管理**：需要仔细管理层与层之间的依赖关系
```csharp
// 表现层 (Presentation Layer)
[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    private readonly IUserService _userService;
    
    public UserController(IUserService userService)
    {
        _userService = userService;
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        if (user == null)
            return NotFound();
            
        return Ok(user);
    }
}

// 业务逻辑层 (Business Logic Layer)
public interface IUserService
{
    Task<UserDto> GetUserByIdAsync(int id);
    Task<UserDto> CreateUserAsync(CreateUserRequest request);
    Task<bool> UpdateUserAsync(int id, UpdateUserRequest request);
    Task<bool> DeleteUserAsync(int id);
}

public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly IUserValidator _userValidator;
    private readonly ILogger<UserService> _logger;
    
    public UserService(
        IUserRepository userRepository,
        IUserValidator userValidator,
        ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _userValidator = userValidator;
        _logger = logger;
    }
    
    public async Task<UserDto> GetUserByIdAsync(int id)
    {
        try
        {
            var user = await _userRepository.GetByIdAsync(id);
            return user?.ToDto();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting user {UserId}", id);
            throw;
        }
    }
    
    public async Task<UserDto> CreateUserAsync(CreateUserRequest request)
    {
        var validationResult = await _userValidator.ValidateAsync(request);
        if (!validationResult.IsValid)
        {
            throw new ValidationException(validationResult.Errors);
        }
        
        var user = new User
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email,
            CreatedAt = DateTime.UtcNow
        };
        
        var createdUser = await _userRepository.AddAsync(user);
        return createdUser.ToDto();
    }
}

// 数据访问层 (Data Access Layer)
public interface IUserRepository
{
    Task<User> GetByIdAsync(int id);
    Task<User> AddAsync(User user);
    Task<bool> UpdateAsync(User user);
    Task<bool> DeleteAsync(int id);
    Task<IEnumerable<User>> GetAllAsync();
}

public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;
    
    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<User> GetByIdAsync(int id)
    {
        return await _context.Users
            .Include(u => u.Profile)
            .FirstOrDefaultAsync(u => u.Id == id);
    }
    
    public async Task<User> AddAsync(User user)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        return user;
    }
}
```

### 1.2 依赖注入配置
```csharp
// 在 Program.cs 中配置依赖注入
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserValidator, UserValidator>();

// 使用工厂模式注册
builder.Services.AddScoped<IUserService>(provider =>
{
    var repository = provider.GetRequiredService<IUserRepository>();
    var validator = provider.GetRequiredService<IUserValidator>();
    var logger = provider.GetRequiredService<ILogger<UserService>>();
    
    return new UserService(repository, validator, logger);
});
```

## 2. 微服务架构 (Microservices)

### 2.1 服务定义
```csharp
// 用户服务
public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly IEventBus _eventBus;
    
    public UserService(IUserRepository userRepository, IEventBus eventBus)
    {
        _userRepository = userRepository;
        _eventBus = eventBus;
    }
    
    public async Task<UserDto> CreateUserAsync(CreateUserRequest request)
    {
        var user = await _userRepository.AddAsync(new User
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email
        });
        
        // 发布用户创建事件
        await _eventBus.PublishAsync(new UserCreatedEvent
        {
            UserId = user.Id,
            Email = user.Email,
            Timestamp = DateTime.UtcNow
        });
        
        return user.ToDto();
    }
}

// 订单服务
public class OrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IUserServiceClient _userServiceClient;
    
    public OrderService(IOrderRepository orderRepository, IUserServiceClient userServiceClient)
    {
        _orderRepository = orderRepository;
        _userServiceClient = userServiceClient;
    }
    
    public async Task<OrderDto> CreateOrderAsync(CreateOrderRequest request)
    {
        // 调用用户服务验证用户
        var user = await _userServiceClient.GetUserAsync(request.UserId);
        if (user == null)
            throw new ValidationException("User not found");
        
        var order = new Order
        {
            UserId = request.UserId,
            TotalAmount = request.TotalAmount,
            Status = OrderStatus.Pending
        };
        
        var createdOrder = await _orderRepository.AddAsync(order);
        return createdOrder.ToDto();
    }
}
```

### 2.2 服务间通信
```csharp
// HTTP客户端
public interface IUserServiceClient
{
    Task<UserDto> GetUserAsync(int userId);
    Task<bool> ValidateUserAsync(int userId);
}

public class UserServiceClient : IUserServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<UserServiceClient> _logger;
    
    public UserServiceClient(HttpClient httpClient, ILogger<UserServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<UserDto> GetUserAsync(int userId)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/users/{userId}");
            response.EnsureSuccessStatusCode();
            
            var content = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<UserDto>(content);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error calling user service for user {UserId}", userId);
            throw;
        }
    }
}

// 事件总线
public interface IEventBus
{
    Task PublishAsync<T>(T @event) where T : class;
    Task SubscribeAsync<T>(Func<T, Task> handler) where T : class;
}

public class RabbitMQEventBus : IEventBus
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public RabbitMQEventBus(IConnection connection)
    {
        _connection = connection;
        _channel = _connection.CreateModel();
    }
    
    public async Task PublishAsync<T>(T @event) where T : class
    {
        var eventName = typeof(T).Name;
        var message = JsonSerializer.Serialize(@event);
        var body = Encoding.UTF8.GetBytes(message);
        
        _channel.BasicPublish(
            exchange: "user_events",
            routingKey: eventName.ToLower(),
            basicProperties: null,
            body: body);
        
        await Task.CompletedTask;
    }
}
```

### 2.3 服务发现和配置
```csharp
// 服务注册
public class ServiceRegistry
{
    private readonly ILogger<ServiceRegistry> _logger;
    private readonly IConfiguration _configuration;
    
    public ServiceRegistry(ILogger<ServiceRegistry> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }
    
    public async Task RegisterServiceAsync()
    {
        var serviceInfo = new ServiceInfo
        {
            ServiceName = "UserService",
            ServiceId = Guid.NewGuid().ToString(),
            Address = _configuration["ServiceAddress"],
            Port = int.Parse(_configuration["ServicePort"]),
            HealthCheckUrl = "/health"
        };
        
        // 注册到服务发现中心
        await RegisterToConsulAsync(serviceInfo);
    }
    
    private async Task RegisterToConsulAsync(ServiceInfo serviceInfo)
    {
        // 实现Consul注册逻辑
        _logger.LogInformation("Service registered: {ServiceName}", serviceInfo.ServiceName);
    }
}

// 健康检查
public class HealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            // 检查数据库连接
            // 检查外部服务依赖
            return Task.FromResult(HealthCheckResult.Healthy());
        }
        catch (Exception ex)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(exception: ex));
        }
    }
}
```

## 3. CQRS 模式 (Command Query Responsibility Segregation)

### 3.1 命令和查询分离
```csharp
// 命令
public interface ICommand
{
    Guid Id { get; }
}

public class CreateUserCommand : ICommand
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
}

public class UpdateUserCommand : ICommand
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public int UserId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

// 查询
public interface IQuery<TResult>
{
}

public class GetUserByIdQuery : IQuery<UserDto>
{
    public int UserId { get; set; }
}

public class GetUsersQuery : IQuery<IEnumerable<UserDto>>
{
    public string SearchTerm { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
}

// 命令处理器
public interface ICommandHandler<in TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command);
}

public class CreateUserCommandHandler : ICommandHandler<CreateUserCommand>
{
    private readonly IUserRepository _userRepository;
    private readonly IEventBus _eventBus;
    
    public CreateUserCommandHandler(IUserRepository userRepository, IEventBus eventBus)
    {
        _userRepository = userRepository;
        _eventBus = eventBus;
    }
    
    public async Task HandleAsync(CreateUserCommand command)
    {
        var user = new User
        {
            FirstName = command.FirstName,
            LastName = command.LastName,
            Email = command.Email,
            CreatedAt = DateTime.UtcNow
        };
        
        await _userRepository.AddAsync(user);
        
        // 发布事件
        await _eventBus.PublishAsync(new UserCreatedEvent
        {
            UserId = user.Id,
            Email = user.Email,
            Timestamp = DateTime.UtcNow
        });
    }
}

// 查询处理器
public interface IQueryHandler<in TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query);
}

public class GetUserByIdQueryHandler : IQueryHandler<GetUserByIdQuery, UserDto>
{
    private readonly IUserReadRepository _userReadRepository;
    
    public GetUserByIdQueryHandler(IUserReadRepository userReadRepository)
    {
        _userReadRepository = userReadRepository;
    }
    
    public async Task<UserDto> HandleAsync(GetUserByIdQuery query)
    {
        var user = await _userReadRepository.GetByIdAsync(query.UserId);
        return user?.ToDto();
    }
}
```

### 3.2 命令和查询总线
```csharp
// 命令总线
public interface ICommandBus
{
    Task SendAsync<TCommand>(TCommand command) where TCommand : ICommand;
}

public class CommandBus : ICommandBus
{
    private readonly IServiceProvider _serviceProvider;
    
    public CommandBus(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task SendAsync<TCommand>(TCommand command) where TCommand : ICommand
    {
        var handlerType = typeof(ICommandHandler<>).MakeGenericType(command.GetType());
        var handler = _serviceProvider.GetService(handlerType);
        
        if (handler == null)
            throw new InvalidOperationException($"No handler found for command {command.GetType().Name}");
        
        var method = handlerType.GetMethod("HandleAsync");
        await (Task)method.Invoke(handler, new object[] { command });
    }
}

// 查询总线
public interface IQueryBus
{
    Task<TResult> SendAsync<TQuery, TResult>(TQuery query) where TQuery : IQuery<TResult>;
}

public class QueryBus : IQueryBus
{
    private readonly IServiceProvider _serviceProvider;
    
    public QueryBus(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task<TResult> SendAsync<TQuery, TResult>(TQuery query) where TQuery : IQuery<TResult>
    {
        var handlerType = typeof(IQueryHandler<,>).MakeGenericType(query.GetType(), typeof(TResult));
        var handler = _serviceProvider.GetService(handlerType);
        
        if (handler == null)
            throw new InvalidOperationException($"No handler found for query {query.GetType().Name}");
        
        var method = handlerType.GetMethod("HandleAsync");
        return await (Task<TResult>)method.Invoke(handler, new object[] { query });
    }
}
```

### 3.3 读写模型分离
```csharp
// 写模型
public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    
    public void Update(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
        UpdatedAt = DateTime.UtcNow;
    }
}

// 读模型
public class UserDto
{
    public int Id { get; set; }
    public string FullName { get; set; }
    public string Email { get; set; }
    public DateTime CreatedAt { get; set; }
    public int OrderCount { get; set; }
    public decimal TotalSpent { get; set; }
}

// 读模型仓储
public interface IUserReadRepository
{
    Task<UserDto> GetByIdAsync(int id);
    Task<IEnumerable<UserDto>> GetAllAsync();
    Task<PaginatedResult<UserDto>> GetPaginatedAsync(int pageNumber, int pageSize, string searchTerm);
}

public class UserReadRepository : IUserReadRepository
{
    private readonly IDbConnection _connection;
    
    public UserReadRepository(IDbConnection connection)
    {
        _connection = connection;
    }
    
    public async Task<UserDto> GetByIdAsync(int id)
    {
        var sql = @"
            SELECT u.Id, u.FirstName, u.LastName, u.Email, u.CreatedAt,
                   COUNT(o.Id) as OrderCount, COALESCE(SUM(o.TotalAmount), 0) as TotalSpent
            FROM Users u
            LEFT JOIN Orders o ON u.Id = o.UserId
            WHERE u.Id = @Id
            GROUP BY u.Id, u.FirstName, u.LastName, u.Email, u.CreatedAt";
        
        var user = await _connection.QueryFirstOrDefaultAsync<UserDto>(sql, new { Id = id });
        if (user != null)
        {
            user.FullName = $"{user.FirstName} {user.LastName}";
        }
        
        return user;
    }
}
```

## 4. 事件驱动架构 (Event-Driven Architecture)

### 4.1 事件定义
```csharp
// 领域事件
public abstract class DomainEvent
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime OccurredOn { get; set; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

public class UserCreatedEvent : DomainEvent
{
    public int UserId { get; set; }
    public string Email { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public class UserUpdatedEvent : DomainEvent
{
    public int UserId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public class UserDeletedEvent : DomainEvent
{
    public int UserId { get; set; }
}

// 集成事件
public abstract class IntegrationEvent : DomainEvent
{
    public string CorrelationId { get; set; }
}

public class UserCreatedIntegrationEvent : IntegrationEvent
{
    public int UserId { get; set; }
    public string Email { get; set; }
}
```

### 4.2 事件发布和订阅
```csharp
// 事件发布器
public interface IEventPublisher
{
    Task PublishAsync<T>(T @event) where T : DomainEvent;
}

public class EventPublisher : IEventPublisher
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<EventPublisher> _logger;
    
    public EventPublisher(IServiceProvider serviceProvider, ILogger<EventPublisher> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    public async Task PublishAsync<T>(T @event) where T : DomainEvent
    {
        var handlers = _serviceProvider.GetServices<IEventHandler<T>>();
        
        foreach (var handler in handlers)
        {
            try
            {
                await handler.HandleAsync(@event);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error handling event {EventType}", @event.EventType);
                // 考虑重试机制或死信队列
            }
        }
    }
}

// 事件处理器
public interface IEventHandler<in TEvent> where TEvent : DomainEvent
{
    Task HandleAsync(TEvent @event);
}

public class UserCreatedEventHandler : IEventHandler<UserCreatedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ILogger<UserCreatedEventHandler> _logger;
    
    public UserCreatedEventHandler(IEmailService emailService, ILogger<UserCreatedEventHandler> logger)
    {
        _emailService = emailService;
        _logger = logger;
    }
    
    public async Task HandleAsync(UserCreatedEvent @event)
    {
        _logger.LogInformation("Handling user created event for user {UserId}", @event.UserId);
        
        // 发送欢迎邮件
        await _emailService.SendWelcomeEmailAsync(@event.Email, @event.FirstName);
        
        // 其他业务逻辑...
    }
}

public class UserCreatedIntegrationEventHandler : IEventHandler<UserCreatedIntegrationEvent>
{
    private readonly IUserServiceClient _userServiceClient;
    
    public UserCreatedIntegrationEventHandler(IUserServiceClient userServiceClient)
    {
        _userServiceClient = userServiceClient;
    }
    
    public async Task HandleAsync(UserCreatedIntegrationEvent @event)
    {
        // 同步用户数据到其他服务
        await _userServiceClient.SyncUserAsync(@event.UserId);
    }
}
```

### 4.3 事件存储
```csharp
// 事件存储接口
public interface IEventStore
{
    Task SaveEventsAsync(int aggregateId, IEnumerable<DomainEvent> events, int expectedVersion);
    Task<IEnumerable<DomainEvent>> GetEventsAsync(int aggregateId);
}

public class EventStore : IEventStore
{
    private readonly IDbConnection _connection;
    private readonly IEventSerializer _serializer;
    
    public EventStore(IDbConnection connection, IEventSerializer serializer)
    {
        _connection = connection;
        _serializer = serializer;
    }
    
    public async Task SaveEventsAsync(int aggregateId, IEnumerable<DomainEvent> events, int expectedVersion)
    {
        var eventList = events.ToList();
        var version = expectedVersion;
        
        foreach (var @event in eventList)
        {
            version++;
            @event.Version = version;
            
            var eventData = new EventData
            {
                AggregateId = aggregateId,
                Version = version,
                EventType = @event.GetType().Name,
                EventData = _serializer.Serialize(@event),
                OccurredOn = @event.OccurredOn
            };
            
            await _connection.ExecuteAsync(@"
                INSERT INTO Events (AggregateId, Version, EventType, EventData, OccurredOn)
                VALUES (@AggregateId, @Version, @EventType, @EventData, @OccurredOn)",
                eventData);
        }
    }
    
    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(int aggregateId)
    {
        var events = await _connection.QueryAsync<EventData>(
            "SELECT * FROM Events WHERE AggregateId = @AggregateId ORDER BY Version",
            new { AggregateId = aggregateId });
        
        return events.Select(e => _serializer.Deserialize(e.EventData, e.EventType));
    }
}
```

## 5. 领域驱动设计 (DDD)

### 5.1 聚合根
```csharp
public class User : AggregateRoot
{
    private readonly List<Order> _orders = new List<Order>();
    
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string Email { get; private set; }
    public UserStatus Status { get; private set; }
    public IReadOnlyCollection<Order> Orders => _orders.AsReadOnly();
    
    private User() { } // EF Core需要
    
    public User(string firstName, string lastName, string email)
    {
        SetName(firstName, lastName);
        SetEmail(email);
        Status = UserStatus.Active;
        
        AddDomainEvent(new UserCreatedEvent
        {
            UserId = Id,
            FirstName = FirstName,
            LastName = LastName,
            Email = Email
        });
    }
    
    public void SetName(string firstName, string lastName)
    {
        if (string.IsNullOrWhiteSpace(firstName))
            throw new DomainException("First name cannot be empty");
            
        if (string.IsNullOrWhiteSpace(lastName))
            throw new DomainException("Last name cannot be empty");
        
        FirstName = firstName;
        LastName = lastName;
        
        AddDomainEvent(new UserUpdatedEvent
        {
            UserId = Id,
            FirstName = FirstName,
            LastName = LastName
        });
    }
    
    public void SetEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email) || !IsValidEmail(email))
            throw new DomainException("Invalid email address");
        
        Email = email;
    }
    
    public void Deactivate()
    {
        if (Status == UserStatus.Inactive)
            throw new DomainException("User is already inactive");
        
        if (_orders.Any(o => o.Status == OrderStatus.Pending))
            throw new DomainException("Cannot deactivate user with pending orders");
        
        Status = UserStatus.Inactive;
        
        AddDomainEvent(new UserDeactivatedEvent { UserId = Id });
    }
    
    public Order CreateOrder(decimal totalAmount)
    {
        if (Status != UserStatus.Active)
            throw new DomainException("Inactive users cannot create orders");
        
        var order = new Order(Id, totalAmount);
        _orders.Add(order);
        
        return order;
    }
    
    private bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}
```

### 5.2 值对象
```csharp
public class Email : ValueObject
{
    public string Value { get; }
    
    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email cannot be empty");
        
        if (!IsValidEmail(value))
            throw new DomainException("Invalid email format");
        
        Value = value.ToLowerInvariant();
    }
    
    public static implicit operator string(Email email) => email.Value;
    public static explicit operator Email(string value) => new Email(value);
    
    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Value;
    }
    
    private bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}

public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency = "USD")
    {
        if (amount < 0)
            throw new DomainException("Amount cannot be negative");
        
        if (string.IsNullOrWhiteSpace(currency))
            throw new DomainException("Currency cannot be empty");
        
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }
    
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new DomainException("Cannot add money with different currencies");
        
        return new Money(left.Amount + right.Amount, left.Currency);
    }
    
    public static Money operator -(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new DomainException("Cannot subtract money with different currencies");
        
        var result = left.Amount - right.Amount;
        if (result < 0)
            throw new DomainException("Result cannot be negative");
        
        return new Money(result, left.Currency);
    }
    
    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

### 5.3 领域服务
```csharp
public interface IUserDomainService
{
    Task<bool> IsEmailUniqueAsync(string email);
    Task<bool> CanUserCreateOrderAsync(int userId);
}

public class UserDomainService : IUserDomainService
{
    private readonly IUserRepository _userRepository;
    private readonly IOrderRepository _orderRepository;
    
    public UserDomainService(IUserRepository userRepository, IOrderRepository orderRepository)
    {
        _userRepository = userRepository;
        _orderRepository = orderRepository;
    }
    
    public async Task<bool> IsEmailUniqueAsync(string email)
    {
        var existingUser = await _userRepository.GetByEmailAsync(email);
        return existingUser == null;
    }
    
    public async Task<bool> CanUserCreateOrderAsync(int userId)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        if (user == null || user.Status != UserStatus.Active)
            return false;
        
        // 检查用户是否有未完成的订单
        var pendingOrders = await _orderRepository.GetPendingOrdersByUserIdAsync(userId);
        return !pendingOrders.Any();
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **分层架构**：职责分离、依赖方向、测试策略
2. **微服务**：服务拆分原则、通信方式、数据一致性
3. **CQRS**：读写分离、事件溯源、性能优化
4. **事件驱动**：松耦合、异步处理、事件存储
5. **DDD**：聚合设计、值对象、领域服务

### 6.2 代码示例准备
- 分层架构的依赖注入配置
- 微服务间的通信实现
- CQRS模式的命令查询分离
- 事件驱动的发布订阅
- DDD的聚合根设计

### 6.3 架构设计原则
- **单一职责**：每个组件只负责一个功能
- **开闭原则**：对扩展开放，对修改关闭
- **依赖倒置**：依赖抽象而非具体实现
- **接口隔离**：客户端不应该依赖它不需要的接口
- **里氏替换**：子类必须能够替换其基类
