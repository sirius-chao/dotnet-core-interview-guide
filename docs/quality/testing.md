# 测试与质量保证面试指南 🚀

## 📚 快速导航
- [面试高频问题](#面试高频问题)
- [单元测试](#1-单元测试)
- [集成测试](#2-集成测试)
- [性能测试](#3-性能测试)
- [自动化测试](#4-自动化测试)
- [面试重点](#5-面试重点)

## ❓ 面试高频问题

### Q1: 如何设计高质量的单元测试？

**面试官想了解什么**：你的测试设计能力和质量意识。

**🎯 标准答案**：

**单元测试设计原则**：
1. **AAA模式**：Arrange（准备）、Act（执行）、Assert（断言）
2. **单一职责**：每个测试只测试一个功能点
3. **独立性**：测试之间相互独立，不依赖执行顺序
4. **可读性**：测试名称清晰表达测试意图

**具体实现**：
```csharp
[Fact]
public async Task GetUserByIdAsync_WithValidId_ReturnsUser()
{
    // Arrange - 准备测试数据
    var userId = 1;
    var expectedUser = new User { Id = userId, Name = "John" };
    _mockRepository.Setup(r => r.GetByIdAsync(userId))
                  .ReturnsAsync(expectedUser);
    
    // Act - 执行被测试方法
    var result = await _service.GetUserByIdAsync(userId);
    
    // Assert - 验证结果
    Assert.NotNull(result);
    Assert.Equal(expectedUser.Id, result.Id);
    Assert.Equal(expectedUser.Name, result.Name);
}
```

**💡 面试加分点**：提到"我会使用测试覆盖率工具确保关键代码路径被测试覆盖"

---

### Q2: 如何选择测试框架和工具？

**面试官想了解什么**：你对测试生态的了解。

**🎯 标准答案**：

**测试框架选择**：
| 框架 | 优势 | 劣势 | 适用场景 | 推荐指数 |
|------|------|------|----------|----------|
| **xUnit** | 简洁、现代、社区活跃 | 学习曲线 | .NET Core项目 | ⭐⭐⭐⭐⭐ |
| **NUnit** | 功能丰富、成熟稳定 | 配置复杂 | 传统.NET项目 | ⭐⭐⭐⭐ |
| **MSTest** | 集成度高、官方支持 | 功能相对简单 | Visual Studio项目 | ⭐⭐⭐ |

**测试工具选择**：
- **Mock框架**：Moq（易用）、NSubstitute（语法友好）
- **断言库**：FluentAssertions（可读性强）、Shouldly（语法简洁）
- **测试数据**：AutoFixture（自动生成测试数据）、Bogus（假数据生成）

**💡 面试加分点**：提到"我会根据项目特点选择合适工具，新项目用xUnit+Moq，老项目用NUnit"

---

### Q3: 如何实现持续集成和自动化测试？

**面试官想了解什么**：你的DevOps和自动化经验。

**🎯 标准答案**：

**CI/CD流水线设计**：
1. **代码提交**：触发自动化测试
2. **代码质量检查**：静态代码分析、代码覆盖率
3. **自动化测试**：单元测试、集成测试、端到端测试
4. **质量门禁**：测试通过率、覆盖率要求

**具体实现**：
```yaml
# Azure DevOps Pipeline
trigger:
- main

stages:
- stage: Test
  jobs:
  - job: UnitTests
    steps:
    - task: DotNetCoreCLI@2
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--collect:"XPlat Code Coverage"'
    
    - task: PublishTestResults@2
      inputs:
        testResultsFormat: 'VSTest'
        testResultsFiles: '**/TestResults/*.trx'
```

**💡 面试加分点**：提到"我会使用SonarQube进行代码质量分析，设置质量门禁确保代码质量"

---

## 🏗️ 实战场景分析

### 场景1：微服务测试策略

**业务需求**：为包含20+微服务的系统设计测试策略

**🎯 技术方案**：

```
代码提交 → 单元测试 → 集成测试 → 契约测试 → 端到端测试 → 部署验证
   ↓         ↓          ↓          ↓          ↓          ↓
  触发CI     快速反馈    服务集成    接口兼容    业务流程    生产验证
```

**核心实现**：
1. **单元测试**：每个服务独立的单元测试，覆盖率>80%
2. **集成测试**：服务间集成测试，使用TestContainers
3. **契约测试**：使用Pact进行服务契约测试
4. **端到端测试**：关键业务流程的端到端测试

**🔑 关键决策**：使用测试金字塔策略，单元测试占70%，集成测试占20%，端到端测试占10%

---

### 场景2：性能测试体系

**业务需求**：构建支持10000+并发用户的性能测试体系

**🎯 技术方案**：

```
性能需求 → 测试设计 → 测试执行 → 性能分析 → 瓶颈识别 → 优化验证
   ↓         ↓          ↓          ↓          ↓          ↓
  性能目标   测试场景    负载测试    数据收集    问题定位    效果验证
```

**核心实现**：
1. **性能测试工具**：使用NBomber、JMeter、K6
2. **测试场景**：登录、搜索、下单等关键场景
3. **监控体系**：应用性能监控、基础设施监控
4. **自动化**：性能测试集成到CI/CD流水线

---

## 📊 测试策略对比图表

### 测试类型对比

```
测试类型对比：
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   单元测试      │    │   集成测试      │    │   端到端测试    │
│                │    │                │    │                │
│ 快速执行       │    │ 中等速度        │    │ 慢速执行        │
│ 低成本         │    │ 中等成本        │    │ 高成本          │
│ 高覆盖率       │    │ 中等覆盖率      │    │ 低覆盖率        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 测试工具对比

| 工具 | 类型 | 优势 | 劣势 | 适用场景 | 推荐指数 |
|------|------|------|------|----------|----------|
| **xUnit** | 单元测试 | 现代、简洁、社区活跃 | 学习曲线 | .NET Core | ⭐⭐⭐⭐⭐ |
| **Moq** | Mock框架 | 易用、功能强大 | 性能开销 | 单元测试 | ⭐⭐⭐⭐⭐ |
| **TestContainers** | 集成测试 | 真实环境、易用 | 启动时间 | 集成测试 | ⭐⭐⭐⭐⭐ |
| **Selenium** | 端到端测试 | 浏览器自动化、成熟 | 不稳定、慢 | Web应用 | ⭐⭐⭐⭐ |

---

## 1. 单元测试

### 1.1 xUnit 测试框架
```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly Mock<IUserValidator> _mockValidator;
    private readonly Mock<ILogger<UserService>> _mockLogger;
    private readonly UserService _service;
    
    public UserServiceTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _mockValidator = new Mock<IUserValidator>();
        _mockLogger = new Mock<ILogger<UserService>>();
        _service = new UserService(_mockRepository.Object, _mockValidator.Object, _mockLogger.Object);
    }
    
    [Fact]
    public async Task GetUserByIdAsync_WithValidId_ReturnsUser()
    {
        // Arrange
        var userId = 1;
        var expectedUser = new User { Id = userId, FirstName = "John", LastName = "Doe" };
        _mockRepository.Setup(r => r.GetByIdAsync(userId)).ReturnsAsync(expectedUser);
        
        // Act
        var result = await _service.GetUserByIdAsync(userId);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedUser.Id, result.Id);
        Assert.Equal(expectedUser.FirstName, result.FirstName);
        Assert.Equal(expectedUser.LastName, result.LastName);
        
        _mockRepository.Verify(r => r.GetByIdAsync(userId), Times.Once);
    }
    
    [Fact]
    public async Task GetUserByIdAsync_WithInvalidId_ReturnsNull()
    {
        // Arrange
        var userId = 999;
        _mockRepository.Setup(r => r.GetByIdAsync(userId)).ReturnsAsync((User)null);
        
        // Act
        var result = await _service.GetUserByIdAsync(userId);
        
        // Assert
        Assert.Null(result);
        _mockRepository.Verify(r => r.GetByIdAsync(userId), Times.Once);
    }
    
    [Fact]
    public async Task CreateUserAsync_WithValidRequest_CreatesUser()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "John",
            LastName = "Doe",
            Email = "john.doe@example.com"
        };
        
        var validationResult = new ValidationResult();
        _mockValidator.Setup(v => v.ValidateAsync(request)).ReturnsAsync(validationResult);
        
        var createdUser = new User
        {
            Id = 1,
            FirstName = request.FirstName,
            LastName = request.LastName,
            Email = request.Email,
            CreatedAt = DateTime.UtcNow
        };
        
        _mockRepository.Setup(r => r.AddAsync(It.IsAny<User>())).ReturnsAsync(createdUser);
        
        // Act
        var result = await _service.CreateUserAsync(request);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(createdUser.Id, result.Id);
        Assert.Equal(createdUser.FirstName, result.FirstName);
        Assert.Equal(createdUser.LastName, result.LastName);
        Assert.Equal(createdUser.Email, result.Email);
        
        _mockValidator.Verify(v => v.ValidateAsync(request), Times.Once);
        _mockRepository.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Once);
    }
    
    [Fact]
    public async Task CreateUserAsync_WithInvalidRequest_ThrowsValidationException()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "",
            LastName = "",
            Email = "invalid-email"
        };
        
        var validationErrors = new List<ValidationFailure>
        {
            new ValidationFailure("FirstName", "First name is required"),
            new ValidationFailure("Email", "Invalid email format")
        };
        
        var validationResult = new ValidationResult(validationErrors);
        _mockValidator.Setup(v => v.ValidateAsync(request)).ReturnsAsync(validationResult);
        
        // Act & Assert
        var exception = await Assert.ThrowsAsync<ValidationException>(
            () => _service.CreateUserAsync(request));
        
        Assert.Equal("Validation failed", exception.Message);
        _mockRepository.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Never);
    }
    
    [Theory]
    [InlineData("john.doe@example.com", true)]
    [InlineData("invalid-email", false)]
    [InlineData("", false)]
    [InlineData(null, false)]
    public async Task CreateUserAsync_WithDifferentEmails_ValidatesCorrectly(string email, bool isValid)
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "John",
            LastName = "Doe",
            Email = email
        };
        
        if (isValid)
        {
            var validationResult = new ValidationResult();
            _mockValidator.Setup(v => v.ValidateAsync(request)).ReturnsAsync(validationResult);
            
            var createdUser = new User { Id = 1, Email = email };
            _mockRepository.Setup(r => r.AddAsync(It.IsAny<User>())).ReturnsAsync(createdUser);
            
            // Act
            var result = await _service.CreateUserAsync(request);
            
            // Assert
            Assert.NotNull(result);
            Assert.Equal(email, result.Email);
        }
        else
        {
            var validationErrors = new List<ValidationFailure>
            {
                new ValidationFailure("Email", "Invalid email format")
            };
            
            var validationResult = new ValidationResult(validationErrors);
            _mockValidator.Setup(v => v.ValidateAsync(request)).ReturnsAsync(validationResult);
            
            // Act & Assert
            await Assert.ThrowsAsync<ValidationException>(
                () => _service.CreateUserAsync(request));
        }
    }
}
```

### 1.2 测试数据构建器
```csharp
public class UserBuilder
{
    private int _id = 1;
    private string _firstName = "John";
    private string _lastName = "Doe";
    private string _email = "john.doe@example.com";
    private DateTime _createdAt = DateTime.UtcNow;
    private UserStatus _status = UserStatus.Active;
    
    public UserBuilder WithId(int id)
    {
        _id = id;
        return this;
    }
    
    public UserBuilder WithFirstName(string firstName)
    {
        _firstName = firstName;
        return this;
    }
    
    public UserBuilder WithLastName(string lastName)
    {
        _lastName = lastName;
        return this;
    }
    
    public UserBuilder WithEmail(string email)
    {
        _email = email;
        return this;
    }
    
    public UserBuilder WithCreatedAt(DateTime createdAt)
    {
        _createdAt = createdAt;
        return this;
    }
    
    public UserBuilder WithStatus(UserStatus status)
    {
        _status = status;
        return this;
    }
    
    public User Build()
    {
        return new User
        {
            Id = _id,
            FirstName = _firstName,
            LastName = _lastName,
            Email = _email,
            CreatedAt = _createdAt,
            Status = _status
        };
    }
    
    public static UserBuilder Default => new UserBuilder();
}

// 使用构建器
[Fact]
public async Task GetUserByIdAsync_WithActiveUser_ReturnsUser()
{
    // Arrange
    var user = UserBuilder.Default
        .WithId(1)
        .WithFirstName("Jane")
        .WithLastName("Smith")
        .WithEmail("jane.smith@example.com")
        .WithStatus(UserStatus.Active)
        .Build();
    
    _mockRepository.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(user);
    
    // Act
    var result = await _service.GetUserByIdAsync(1);
    
    // Assert
    Assert.Equal(user.Id, result.Id);
    Assert.Equal(user.FirstName, result.FirstName);
    Assert.Equal(user.LastName, result.LastName);
    Assert.Equal(user.Email, result.Email);
    Assert.Equal(user.Status, result.Status);
}
```

### 1.3 测试夹具 (Test Fixtures)
```csharp
public class UserServiceTestFixture : IDisposable
{
    public Mock<IUserRepository> MockRepository { get; }
    public Mock<IUserValidator> MockValidator { get; }
    public Mock<ILogger<UserService>> MockLogger { get; }
    public UserService Service { get; }
    
    public UserServiceTestFixture()
    {
        MockRepository = new Mock<IUserRepository>();
        MockValidator = new Mock<IUserValidator>();
        MockLogger = new Mock<ILogger<UserService>>();
        Service = new UserService(MockRepository.Object, MockValidator.Object, MockLogger.Object);
    }
    
    public void Dispose()
    {
        // 清理资源
    }
}

public class UserServiceTests : IClassFixture<UserServiceTestFixture>
{
    private readonly UserServiceTestFixture _fixture;
    
    public UserServiceTests(UserServiceTestFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public async Task GetUserByIdAsync_WithValidId_ReturnsUser()
    {
        // Arrange
        var userId = 1;
        var expectedUser = new User { Id = userId, FirstName = "John", LastName = "Doe" };
        _fixture.MockRepository.Setup(r => r.GetByIdAsync(userId)).ReturnsAsync(expectedUser);
        
        // Act
        var result = await _fixture.Service.GetUserByIdAsync(userId);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedUser.Id, result.Id);
    }
}
```

## 2. 集成测试

### 2.1 数据库集成测试
```csharp
public class UserServiceIntegrationTests : IClassFixture<TestDatabaseFixture>
{
    private readonly TestDatabaseFixture _fixture;
    private readonly UserService _service;
    private readonly ApplicationDbContext _context;
    
    public UserServiceIntegrationTests(TestDatabaseFixture fixture)
    {
        _fixture = fixture;
        _context = fixture.CreateContext();
        
        var validator = new UserValidator();
        var logger = LoggerFactory.Create(builder => builder.AddConsole()).CreateLogger<UserService>();
        
        _service = new UserService(new UserRepository(_context), validator, logger);
    }
    
    [Fact]
    public async Task CreateUserAsync_WithValidData_CreatesUserInDatabase()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "Integration",
            LastName = "Test",
            Email = "integration.test@example.com"
        };
        
        // Act
        var result = await _service.CreateUserAsync(request);
        
        // Assert
        Assert.NotNull(result);
        Assert.NotEqual(0, result.Id);
        
        // 验证数据确实保存在数据库中
        var savedUser = await _context.Users.FindAsync(result.Id);
        Assert.NotNull(savedUser);
        Assert.Equal(request.FirstName, savedUser.FirstName);
        Assert.Equal(request.LastName, savedUser.LastName);
        Assert.Equal(request.Email, savedUser.Email);
    }
    
    [Fact]
    public async Task GetUserByIdAsync_WithExistingUser_ReturnsUser()
    {
        // Arrange
        var user = new User
        {
            FirstName = "Existing",
            LastName = "User",
            Email = "existing.user@example.com",
            CreatedAt = DateTime.UtcNow
        };
        
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        
        // Act
        var result = await _service.GetUserByIdAsync(user.Id);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(user.Id, result.Id);
        Assert.Equal(user.FirstName, result.FirstName);
        Assert.Equal(user.LastName, result.LastName);
        Assert.Equal(user.Email, result.Email);
    }
    
    public void Dispose()
    {
        _context.Dispose();
    }
}

public class TestDatabaseFixture
{
    private readonly DbContextOptions<ApplicationDbContext> _options;
    
    public TestDatabaseFixture()
    {
        _options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
    }
    
    public ApplicationDbContext CreateContext()
    {
        var context = new ApplicationDbContext(_options);
        context.Database.EnsureCreated();
        return context;
    }
}
```

### 2.2 API 集成测试
```csharp
public class UserControllerIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public UserControllerIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // 替换真实服务为测试服务
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(IUserRepository));
                
                if (descriptor != null)
                {
                    services.Remove(descriptor);
                    services.AddScoped<IUserRepository, TestUserRepository>();
                }
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task GetUser_WithValidId_ReturnsUser()
    {
        // Arrange
        var userId = 1;
        
        // Act
        var response = await _client.GetAsync($"/api/users/{userId}");
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        var user = JsonSerializer.Deserialize<UserDto>(content, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });
        
        Assert.NotNull(user);
        Assert.Equal(userId, user.Id);
    }
    
    [Fact]
    public async Task CreateUser_WithValidData_ReturnsCreatedUser()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "API",
            LastName = "Test",
            Email = "api.test@example.com"
        };
        
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/users", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdUser = JsonSerializer.Deserialize<UserDto>(responseContent, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });
        
        Assert.NotNull(createdUser);
        Assert.NotEqual(0, createdUser.Id);
        Assert.Equal(request.FirstName, createdUser.FirstName);
        Assert.Equal(request.LastName, createdUser.LastName);
        Assert.Equal(request.Email, createdUser.Email);
    }
    
    [Fact]
    public async Task CreateUser_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            FirstName = "",
            LastName = "",
            Email = "invalid-email"
        };
        
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/users", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        Assert.Contains("validation", responseContent.ToLower());
    }
}

// 测试仓储实现
public class TestUserRepository : IUserRepository
{
    private readonly List<User> _users = new List<User>();
    private int _nextId = 1;
    
    public async Task<User> GetByIdAsync(int id)
    {
        return await Task.FromResult(_users.FirstOrDefault(u => u.Id == id));
    }
    
    public async Task<User> AddAsync(User user)
    {
        user.Id = _nextId++;
        user.CreatedAt = DateTime.UtcNow;
        _users.Add(user);
        return await Task.FromResult(user);
    }
    
    public async Task<bool> UpdateAsync(User user)
    {
        var existingUser = _users.FirstOrDefault(u => u.Id == user.Id);
        if (existingUser == null) return false;
        
        existingUser.FirstName = user.FirstName;
        existingUser.LastName = user.LastName;
        existingUser.Email = user.Email;
        
        return await Task.FromResult(true);
    }
    
    public async Task<bool> DeleteAsync(int id)
    {
        var user = _users.FirstOrDefault(u => u.Id == id);
        if (user == null) return false;
        
        _users.Remove(user);
        return await Task.FromResult(true);
    }
    
    public async Task<IEnumerable<User>> GetAllAsync()
    {
        return await Task.FromResult(_users.AsEnumerable());
    }
}
```

## 3. 性能测试

### 3.1 基准测试
```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class UserServiceBenchmarks
{
    private readonly UserService _service;
    private readonly List<CreateUserRequest> _requests;
    
    public UserServiceBenchmarks()
    {
        var mockRepository = new Mock<IUserRepository>();
        var mockValidator = new Mock<IUserValidator>();
        var mockLogger = new Mock<ILogger<UserService>>();
        
        _service = new UserService(mockRepository.Object, mockValidator.Object, mockLogger.Object);
        
        _requests = Enumerable.Range(0, 1000)
            .Select(i => new CreateUserRequest
            {
                FirstName = $"User{i}",
                LastName = $"Test{i}",
                Email = $"user{i}@test.com"
            })
            .ToList();
    }
    
    [Benchmark]
    public async Task CreateUserAsync_Single()
    {
        var request = _requests[0];
        await _service.CreateUserAsync(request);
    }
    
    [Benchmark]
    public async Task CreateUserAsync_Multiple()
    {
        foreach (var request in _requests.Take(100))
        {
            await _service.CreateUserAsync(request);
        }
    }
    
    [Benchmark]
    public async Task CreateUserAsync_Parallel()
    {
        var tasks = _requests.Take(100).Select(request => _service.CreateUserAsync(request));
        await Task.WhenAll(tasks);
    }
}

// 运行基准测试
public class BenchmarkRunner
{
    public static void RunBenchmarks()
    {
        var summary = BenchmarkRunner.Run<UserServiceBenchmarks>();
        
        Console.WriteLine("Benchmark Results:");
        Console.WriteLine(summary);
    }
}
```

### 3.2 负载测试
```csharp
public class LoadTest
{
    private readonly HttpClient _client;
    private readonly int _concurrentUsers;
    private readonly int _requestsPerUser;
    
    public LoadTest(string baseUrl, int concurrentUsers, int requestsPerUser)
    {
        _client = new HttpClient { BaseAddress = new Uri(baseUrl) };
        _concurrentUsers = concurrentUsers;
        _requestsPerUser = requestsPerUser;
    }
    
    public async Task<LoadTestResult> RunAsync()
    {
        var stopwatch = Stopwatch.StartNew();
        var tasks = new List<Task<List<RequestResult>>>();
        
        // 创建并发用户任务
        for (int i = 0; i < _concurrentUsers; i++)
        {
            tasks.Add(SimulateUserAsync(i));
        }
        
        // 等待所有用户完成
        var results = await Task.WhenAll(tasks);
        
        stopwatch.Stop();
        
        // 合并所有结果
        var allResults = results.SelectMany(r => r).ToList();
        
        return new LoadTestResult
        {
            TotalRequests = allResults.Count,
            TotalTime = stopwatch.Elapsed,
            ConcurrentUsers = _concurrentUsers,
            RequestsPerUser = _requestsPerUser,
            Results = allResults
        };
    }
    
    private async Task<List<RequestResult>> SimulateUserAsync(int userId)
    {
        var results = new List<RequestResult>();
        
        for (int i = 0; i < _requestsPerUser; i++)
        {
            var stopwatch = Stopwatch.StartNew();
            
            try
            {
                var response = await _client.GetAsync($"/api/users/{userId}");
                stopwatch.Stop();
                
                results.Add(new RequestResult
                {
                    UserId = userId,
                    RequestId = i,
                    StatusCode = (int)response.StatusCode,
                    ResponseTime = stopwatch.Elapsed,
                    Success = response.IsSuccessStatusCode
                });
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                results.Add(new RequestResult
                {
                    UserId = userId,
                    RequestId = i,
                    StatusCode = 0,
                    ResponseTime = stopwatch.Elapsed,
                    Success = false,
                    Error = ex.Message
                });
            }
            
            // 模拟用户思考时间
            await Task.Delay(Random.Shared.Next(100, 500));
        }
        
        return results;
    }
}

public class LoadTestResult
{
    public int TotalRequests { get; set; }
    public TimeSpan TotalTime { get; set; }
    public int ConcurrentUsers { get; set; }
    public int RequestsPerUser { get; set; }
    public List<RequestResult> Results { get; set; } = new();
    
    public double RequestsPerSecond => TotalRequests / TotalTime.TotalSeconds;
    public double AverageResponseTime => Results.Average(r => r.ResponseTime.TotalMilliseconds);
    public double SuccessRate => Results.Count(r => r.Success) * 100.0 / TotalRequests;
    
    public void PrintSummary()
    {
        Console.WriteLine($"Load Test Results:");
        Console.WriteLine($"Total Requests: {TotalRequests}");
        Console.WriteLine($"Total Time: {TotalTime}");
        Console.WriteLine($"Concurrent Users: {ConcurrentUsers}");
        Console.WriteLine($"Requests per User: {RequestsPerUser}");
        Console.WriteLine($"Requests per Second: {RequestsPerSecond:F2}");
        Console.WriteLine($"Average Response Time: {AverageResponseTime:F2}ms");
        Console.WriteLine($"Success Rate: {SuccessRate:F2}%");
        
        var responseTimePercentiles = Results
            .Where(r => r.Success)
            .Select(r => r.ResponseTime.TotalMilliseconds)
            .OrderBy(t => t)
            .ToList();
        
        if (responseTimePercentiles.Any())
        {
            var p50 = responseTimePercentiles[responseTimePercentiles.Count / 2];
            var p95 = responseTimePercentiles[(int)(responseTimePercentiles.Count * 0.95)];
            var p99 = responseTimePercentiles[(int)(responseTimePercentiles.Count * 0.99)];
            
            Console.WriteLine($"Response Time Percentiles:");
            Console.WriteLine($"  P50: {p50:F2}ms");
            Console.WriteLine($"  P95: {p95:F2}ms");
            Console.WriteLine($"  P99: {p99:F2}ms");
        }
    }
}

public class RequestResult
{
    public int UserId { get; set; }
    public int RequestId { get; set; }
    public int StatusCode { get; set; }
    public TimeSpan ResponseTime { get; set; }
    public bool Success { get; set; }
    public string Error { get; set; }
}
```

## 4. 测试驱动开发 (TDD)

### 4.1 TDD 循环示例
```csharp
// 第一步：编写失败的测试
[Fact]
public void CalculateDiscount_WithValidOrder_ReturnsCorrectDiscount()
{
    // Arrange
    var order = new Order
    {
        TotalAmount = 100.00m,
        CustomerType = CustomerType.Premium,
        OrderDate = DateTime.Today
    };
    
    var calculator = new DiscountCalculator();
    
    // Act
    var discount = calculator.CalculateDiscount(order);
    
    // Assert
    Assert.Equal(15.00m, discount); // 期望15%的折扣
}

// 第二步：编写最简单的实现使测试通过
public class DiscountCalculator
{
    public decimal CalculateDiscount(Order order)
    {
        if (order.CustomerType == CustomerType.Premium)
        {
            return order.TotalAmount * 0.15m;
        }
        
        return 0;
    }
}

// 第三步：重构代码
public class DiscountCalculator
{
    private readonly Dictionary<CustomerType, decimal> _discountRates = new()
    {
        { CustomerType.Regular, 0.00m },
        { CustomerType.Premium, 0.15m },
        { CustomerType.VIP, 0.25m }
    };
    
    public decimal CalculateDiscount(Order order)
    {
        if (order == null)
            throw new ArgumentNullException(nameof(order));
        
        if (!_discountRates.TryGetValue(order.CustomerType, out var rate))
            return 0;
        
        return order.TotalAmount * rate;
    }
}

// 添加更多测试用例
[Theory]
[InlineData(CustomerType.Regular, 100.00, 0.00)]
[InlineData(CustomerType.Premium, 100.00, 15.00)]
[InlineData(CustomerType.VIP, 100.00, 25.00)]
[InlineData(CustomerType.Premium, 0.00, 0.00)]
[InlineData(CustomerType.Premium, 200.00, 30.00)]
public void CalculateDiscount_WithDifferentCustomerTypes_ReturnsCorrectDiscount(
    CustomerType customerType, decimal totalAmount, decimal expectedDiscount)
{
    // Arrange
    var order = new Order
    {
        TotalAmount = totalAmount,
        CustomerType = customerType,
        OrderDate = DateTime.Today
    };
    
    var calculator = new DiscountCalculator();
    
    // Act
    var discount = calculator.CalculateDiscount(order);
    
    // Assert
    Assert.Equal(expectedDiscount, discount);
}

[Fact]
public void CalculateDiscount_WithNullOrder_ThrowsArgumentNullException()
{
    // Arrange
    var calculator = new DiscountCalculator();
    
    // Act & Assert
    Assert.Throws<ArgumentNullException>(() => calculator.CalculateDiscount(null));
}
```

### 4.2 行为驱动开发 (BDD)
```csharp
[Feature("User Management")]
public class UserManagementFeature
{
    [Scenario("Create a new user")]
    public void CreateNewUser()
    {
        var userService = default(IUserService);
        var createRequest = default(CreateUserRequest);
        var result = default(UserDto);
        var exception = default(Exception);
        
        "Given a user service" = () =>
        {
            userService = new UserService(
                new Mock<IUserRepository>().Object,
                new Mock<IUserValidator>().Object,
                new Mock<ILogger<UserService>>().Object);
        };
        
        "And a valid user creation request" = () =>
        {
            createRequest = new CreateUserRequest
            {
                FirstName = "John",
                LastName = "Doe",
                Email = "john.doe@example.com"
            };
        };
        
        "When I create a new user" = async () =>
        {
            try
            {
                result = await userService.CreateUserAsync(createRequest);
            }
            catch (Exception ex)
            {
                exception = ex;
            }
        };
        
        "Then the user should be created successfully" = () =>
        {
            Assert.Null(exception);
            Assert.NotNull(result);
            Assert.NotEqual(0, result.Id);
            Assert.Equal(createRequest.FirstName, result.FirstName);
            Assert.Equal(createRequest.LastName, result.LastName);
            Assert.Equal(createRequest.Email, result.Email);
        };
        
        "And the user should have a creation timestamp" = () =>
        {
            Assert.True(result.CreatedAt > DateTime.UtcNow.AddMinutes(-1));
        };
    }
    
    [Scenario("Create user with invalid email")]
    public void CreateUserWithInvalidEmail()
    {
        var userService = default(IUserService);
        var createRequest = default(CreateUserRequest);
        var exception = default(Exception);
        
        "Given a user service" = () =>
        {
            userService = new UserService(
                new Mock<IUserRepository>().Object,
                new Mock<IUserValidator>().Object,
                new Mock<ILogger<UserService>>().Object);
        };
        
        "And a user creation request with invalid email" = () =>
        {
            createRequest = new CreateUserRequest
            {
                FirstName = "John",
                LastName = "Doe",
                Email = "invalid-email"
            };
        };
        
        "When I create a new user" = async () =>
        {
            try
            {
                await userService.CreateUserAsync(createRequest);
            }
            catch (Exception ex)
            {
                exception = ex;
            }
        };
        
        "Then a validation exception should be thrown" = () =>
        {
            Assert.NotNull(exception);
            Assert.IsType<ValidationException>(exception);
        };
        
        "And the exception should contain email validation error" = () =>
        {
            Assert.Contains("email", exception.Message.ToLower());
        };
    }
}
```

## 5. 测试覆盖率

### 5.1 代码覆盖率分析
```csharp
// 使用 Coverlet 进行代码覆盖率分析
public class UserServiceCoverageTests
{
    [Fact]
    public void UserService_ShouldHaveFullCoverage()
    {
        // 这个测试确保所有代码路径都被覆盖
        var userService = new UserService(
            new Mock<IUserRepository>().Object,
            new Mock<IUserValidator>().Object,
            new Mock<ILogger<UserService>>().Object);
        
        // 测试所有公共方法
        Assert.NotNull(userService);
    }
}

// 在项目文件中配置覆盖率
/*
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
  
  <ItemGroup>
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
*/
```

## 6. 面试重点

### 6.1 高频问题
1. **单元测试**：测试框架选择、Mock使用、测试数据构建
2. **集成测试**：数据库测试、API测试、测试夹具
3. **性能测试**：基准测试、负载测试、性能指标
4. **TDD/BDD**：测试驱动开发、行为驱动开发
5. **测试覆盖率**：覆盖率分析、质量指标

### 6.2 代码示例准备
- 单元测试的完整示例
- 集成测试的数据库和API测试
- 性能测试的基准和负载测试
- TDD循环的实现
- 测试覆盖率配置

### 6.3 测试质量要点
- 测试的可读性和可维护性
- 测试的完整性和覆盖率
- 测试的性能和稳定性
- 测试的自动化程度
- 测试的持续集成
