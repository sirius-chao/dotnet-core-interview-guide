# 测试与质量保证

## 1. 单元测试

### 1.1 测试框架

#### xUnit 基础
```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly UserService _userService;
    
    public UserServiceTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _userService = new UserService(_mockRepository.Object);
    }
    
    [Fact]
    public async Task GetUserById_WithValidId_ReturnsUser()
    {
        // Arrange
        var userId = 1;
        var expectedUser = new User { Id = userId, Name = "Test User" };
        _mockRepository.Setup(r => r.GetByIdAsync(userId))
                      .ReturnsAsync(expectedUser);
        
        // Act
        var result = await _userService.GetUserByIdAsync(userId);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedUser.Name, result.Name);
        _mockRepository.Verify(r => r.GetByIdAsync(userId), Times.Once);
    }
    
    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public async Task GetUserById_WithInvalidId_ThrowsArgumentException(int invalidId)
    {
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(
            () => _userService.GetUserByIdAsync(invalidId));
    }
}
```

#### 测试数据生成
```csharp
public class UserServiceTests
{
    [Fact]
    public async Task CreateUser_WithValidData_CreatesUser()
    {
        // Arrange
        var userData = new CreateUserRequest
        {
            FirstName = "John",
            LastName = "Doe",
            Email = "john.doe@example.com"
        };
        
        _mockRepository.Setup(r => r.CreateAsync(It.IsAny<User>()))
                      .ReturnsAsync(1);
        
        // Act
        var result = await _userService.CreateUserAsync(userData);
        
        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(1, result.UserId);
    }
    
    [Fact]
    public async Task CreateUser_WithDuplicateEmail_ReturnsFailure()
    {
        // Arrange
        var userData = new CreateUserRequest
        {
            FirstName = "John",
            LastName = "Doe",
            Email = "existing@example.com"
        };
        
        _mockRepository.Setup(r => r.GetByEmailAsync(userData.Email))
                      .ReturnsAsync(new User());
        
        // Act
        var result = await _userService.CreateUserAsync(userData);
        
        // Assert
        Assert.False(result.IsSuccess);
        Assert.Contains("Email already exists", result.ErrorMessage);
    }
}
```

### 1.2 测试工具

#### Moq 高级用法
```csharp
public class AdvancedMockingTests
{
    [Fact]
    public void Mock_WithCallback_ExecutesCustomLogic()
    {
        // Arrange
        var mockService = new Mock<IMyService>();
        var callCount = 0;
        
        mockService.Setup(s => s.Process(It.IsAny<string>()))
                  .Callback<string>(input => callCount++)
                  .Returns("processed");
        
        // Act
        mockService.Object.Process("test1");
        mockService.Object.Process("test2");
        
        // Assert
        Assert.Equal(2, callCount);
    }
    
    [Fact]
    public void Mock_WithSequence_ReturnsDifferentValues()
    {
        // Arrange
        var mockService = new Mock<IMyService>();
        mockService.SetupSequence(s => s.GetNextId())
                  .Returns(1)
                  .Returns(2)
                  .Returns(3);
        
        // Act & Assert
        Assert.Equal(1, mockService.Object.GetNextId());
        Assert.Equal(2, mockService.Object.GetNextId());
        Assert.Equal(3, mockService.Object.GetNextId());
    }
}
```

#### AutoFixture 集成
```csharp
public class AutoFixtureTests
{
    private readonly IFixture _fixture;
    
    public AutoFixtureTests()
    {
        _fixture = new Fixture();
        _fixture.Behaviors.OfType<ThrowingRecursionBehavior>()
                .ToList()
                .ForEach(b => _fixture.Behaviors.Remove(b));
        _fixture.Behaviors.Add(new OmitOnRecursionBehavior());
    }
    
    [Fact]
    public void Create_WithAutoFixture_GeneratesValidData()
    {
        // Arrange & Act
        var user = _fixture.Create<User>();
        
        // Assert
        Assert.NotNull(user.FirstName);
        Assert.NotNull(user.LastName);
        Assert.NotNull(user.Email);
    }
    
    [Fact]
    public void Create_WithCustomization_AppliesRules()
    {
        // Arrange
        _fixture.Customize<User>(c => c
            .With(u => u.Email, "test@example.com")
            .With(u => u.IsActive, true));
        
        // Act
        var user = _fixture.Create<User>();
        
        // Assert
        Assert.Equal("test@example.com", user.Email);
        Assert.True(user.IsActive);
    }
}
```

## 2. 集成测试

### 2.1 Web 应用测试

#### TestServer 测试
```csharp
public class WebApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public WebApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // 替换真实服务为测试服务
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContext));
                if (descriptor != null)
                {
                    services.Remove(descriptor);
                    services.AddDbContext<DbContext>(options =>
                    {
                        options.UseInMemoryDatabase("TestDb");
                    });
                }
            });
        });
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task GetUsers_ReturnsUsersList()
    {
        // Act
        var response = await _client.GetAsync("/api/users");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        var users = JsonSerializer.Deserialize<List<User>>(content);
        Assert.NotNull(users);
    }
    
    [Fact]
    public async Task CreateUser_WithValidData_ReturnsCreatedUser()
    {
        // Arrange
        var userData = new CreateUserRequest
        {
            FirstName = "John",
            LastName = "Doe",
            Email = "john.doe@example.com"
        };
        var json = JsonSerializer.Serialize(userData);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/users", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var location = response.Headers.Location;
        Assert.NotNull(location);
    }
}
```

### 2.2 数据库测试

#### 内存数据库测试
```csharp
public class RepositoryIntegrationTests : IDisposable
{
    private readonly DbContext _context;
    private readonly UserRepository _repository;
    
    public RepositoryIntegrationTests()
    {
        var options = new DbContextOptionsBuilder<DbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        
        _context = new DbContext(options);
        _repository = new UserRepository(_context);
        
        // 种子数据
        SeedData();
    }
    
    private void SeedData()
    {
        var users = new List<User>
        {
            new User { Id = 1, FirstName = "John", LastName = "Doe", Email = "john@example.com" },
            new User { Id = 2, FirstName = "Jane", LastName = "Smith", Email = "jane@example.com" }
        };
        
        _context.Users.AddRange(users);
        _context.SaveChanges();
    }
    
    [Fact]
    public async Task GetById_WithExistingId_ReturnsUser()
    {
        // Act
        var user = await _repository.GetByIdAsync(1);
        
        // Assert
        Assert.NotNull(user);
        Assert.Equal("John", user.FirstName);
    }
    
    [Fact]
    public async Task Create_WithValidUser_AddsToDatabase()
    {
        // Arrange
        var newUser = new User
        {
            FirstName = "Bob",
            LastName = "Johnson",
            Email = "bob@example.com"
        };
        
        // Act
        var userId = await _repository.CreateAsync(newUser);
        
        // Assert
        Assert.True(userId > 0);
        var savedUser = await _repository.GetByIdAsync(userId);
        Assert.NotNull(savedUser);
        Assert.Equal("Bob", savedUser.FirstName);
    }
    
    public void Dispose()
    {
        _context?.Dispose();
    }
}
```

## 3. 性能测试

### 3.1 基准测试

#### BenchmarkDotNet
```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class StringConcatenationBenchmarks
{
    private readonly string[] _strings = { "Hello", "World", "Benchmark", "Test" };
    
    [Benchmark]
    public string StringConcatenation()
    {
        string result = "";
        foreach (var str in _strings)
        {
            result += str;
        }
        return result;
    }
    
    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var str in _strings)
        {
            sb.Append(str);
        }
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringJoin()
    {
        return string.Join("", _strings);
    }
    
    [Benchmark]
    public string StringConcat()
    {
        return string.Concat(_strings);
    }
}
```

### 3.2 负载测试

#### NBomber
```csharp
public class LoadTest
{
    [Fact]
    public void UserApi_LoadTest()
    {
        var httpClient = new HttpClient();
        
        var scenario = Scenario.Create("user_api_load_test", async context =>
        {
            try
            {
                var response = await httpClient.GetAsync("https://api.example.com/users");
                
                if (response.IsSuccessStatusCode)
                {
                    context.Ok();
                }
                else
                {
                    context.Fail();
                }
            }
            catch
            {
                context.Fail();
            }
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), copies: 10)
        );
        
        NBomberRunner
            .RegisterScenarios(scenario)
            .Run();
    }
}
```

## 4. 测试策略

### 4.1 测试金字塔

#### 测试层次结构
```
        /\
       /  \     E2E Tests (少量)
      /____\    
     /      \   
    /        \  Integration Tests (中等)
   /__________\ 
  /            \ 
 /              \ Unit Tests (大量)
/________________\
```

#### 测试覆盖率目标
```csharp
// 单元测试: 80-90%
// 集成测试: 60-70%
// E2E测试: 20-30%
```

### 4.2 测试数据管理

#### 测试数据工厂
```csharp
public static class TestDataFactory
{
    public static User CreateValidUser()
    {
        return new User
        {
            FirstName = "Test",
            LastName = "User",
            Email = $"test.{Guid.NewGuid()}@example.com",
            IsActive = true
        };
    }
    
    public static User CreateInvalidUser()
    {
        return new User
        {
            FirstName = "",
            LastName = "",
            Email = "invalid-email",
            IsActive = false
        };
    }
    
    public static List<User> CreateUserList(int count)
    {
        var users = new List<User>();
        for (int i = 0; i < count; i++)
        {
            users.Add(CreateValidUser());
        }
        return users;
    }
}
```

## 5. 质量保证工具

### 5.1 代码质量

#### SonarQube 规则
```csharp
// 避免魔法数字
public class Calculator
{
    private const int MaxRetryCount = 3; // 而不是直接使用 3
    
    public async Task<int> CalculateAsync(int input)
    {
        for (int i = 0; i < MaxRetryCount; i++)
        {
            try
            {
                return await PerformCalculationAsync(input);
            }
            catch (Exception) when (i < MaxRetryCount - 1)
            {
                await Task.Delay(1000); // 延迟1秒
                continue;
            }
        }
        throw new InvalidOperationException("Calculation failed after retries");
    }
}
```

#### StyleCop 配置
```xml
<!-- StyleCop.json -->
{
  "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
  "settings": {
    "documentationRules": {
      "companyName": "Your Company",
      "copyrightText": "Copyright (c) {companyName}. All rights reserved.",
      "variables": {
        "companyName": "Your Company"
      }
    },
    "layoutRules": {
      "newlineAtEndOfFile": "Require"
    },
    "namingRules": {
      "allowCommonHungarianPrefixes": false
    }
  }
}
```

### 5.2 持续集成

#### GitHub Actions 测试
```yaml
name: Test and Quality
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore
    
    - name: Run tests
      run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage"
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./TestResults/coverage.cobertura.xml
        flags: unittests
        name: codecov-umbrella
    
    - name: Run SonarQube
      run: |
        dotnet tool install --global dotnet-sonarscanner
        dotnet sonarscanner begin /k:"your-project-key" /d:sonar.host.url="https://sonarcloud.io" /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
        dotnet build
        dotnet sonarscanner end /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
```

## 6. 面试重点

### 6.1 测试设计
- **测试用例设计**: 边界值、等价类、错误场景
- **测试数据管理**: 工厂模式、种子数据、清理策略
- **测试隔离**: 每个测试独立、无副作用

### 6.2 测试工具
- **Mock 框架**: Moq 的高级用法、回调、序列
- **测试数据**: AutoFixture 自定义、数据驱动测试
- **性能测试**: BenchmarkDotNet、NBomber

### 6.3 测试策略
- **测试金字塔**: 各层测试的比例和重要性
- **测试覆盖率**: 目标设定、质量指标
- **持续测试**: CI/CD 集成、自动化测试

### 6.4 质量保证
- **代码质量**: SonarQube、StyleCop、代码审查
- **测试自动化**: 单元测试、集成测试、E2E测试
- **质量指标**: 缺陷密度、测试通过率、性能基准
