# ASP.NET Core Web开发技术

## 1. ASP.NET Core MVC

### 1.1 控制器最佳实践
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(IUserService userService, ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers([FromQuery] UserQueryParameters parameters)
    {
        try
        {
            var users = await _userService.GetUsersAsync(parameters);
            return Ok(users);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting users");
            return StatusCode(500, "Internal server error");
        }
    }

    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

### 1.2 模型验证
```csharp
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string FirstName { get; set; }

    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string LastName { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [StringLength(20, MinimumLength = 6)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{6,}$", 
        ErrorMessage = "Password must contain at least one uppercase letter, one lowercase letter, and one number")]
    public string Password { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }
}

// 自定义验证特性
public class UniqueEmailAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        var email = value as string;
        if (string.IsNullOrEmpty(email))
            return ValidationResult.Success;

        var userService = validationContext.GetService<IUserService>();
        if (userService.IsEmailUnique(email))
            return ValidationResult.Success;

        return new ValidationResult("Email already exists");
    }
}
```

### 1.3 过滤器 (Filters)
```csharp
// 授权过滤器
[Authorize(Roles = "Admin")]
public class AdminController : ControllerBase
{
    [HttpGet("sensitive-data")]
    public IActionResult GetSensitiveData()
    {
        return Ok("Admin only data");
    }
}

// 自定义授权过滤器
public class CustomAuthorizationFilter : IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var user = context.HttpContext.User;
        if (!user.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }

        // 自定义授权逻辑
        if (!user.HasClaim("Permission", "ReadData"))
        {
            context.Result = new ForbidResult();
        }
    }
}

// 异常过滤器
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;

    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");

        var result = new ObjectResult(new
        {
            Message = "An error occurred while processing your request",
            ErrorId = Guid.NewGuid().ToString()
        })
        {
            StatusCode = 500
        };

        context.Result = result;
        context.ExceptionHandled = true;
    }
}
```

## 2. Web API 设计

### 2.1 RESTful API 设计原则
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class ProductsController : ControllerBase
{
    // GET api/v1/products
    [HttpGet]
    [ProducesResponseType(typeof(PaginatedResult<ProductDto>), 200)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<PaginatedResult<ProductDto>>> GetProducts(
        [FromQuery] ProductQueryParameters parameters)
    {
        var products = await _productService.GetProductsAsync(parameters);
        var paginatedResult = new PaginatedResult<ProductDto>
        {
            Data = products,
            PageNumber = parameters.PageNumber,
            PageSize = parameters.PageSize,
            TotalCount = await _productService.GetTotalCountAsync(parameters)
        };

        Response.Headers.Add("X-Pagination", JsonSerializer.Serialize(paginatedResult.GetMetadata()));
        return Ok(paginatedResult);
    }

    // GET api/v1/products/{id}
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        if (product == null)
            return NotFound();

        return Ok(product);
    }

    // POST api/v1/products
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), 201)]
    [ProducesResponseType(400)]
    public async Task<ActionResult<ProductDto>> CreateProduct([FromBody] CreateProductRequest request)
    {
        var product = await _productService.CreateProductAsync(request);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }

    // PUT api/v1/products/{id}
    [HttpPut("{id:int}")]
    [ProducesResponseType(204)]
    [ProducesResponseType(400)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> UpdateProduct(int id, [FromBody] UpdateProductRequest request)
    {
        var success = await _productService.UpdateProductAsync(id, request);
        if (!success)
            return NotFound();

        return NoContent();
    }

    // DELETE api/v1/products/{id}
    [HttpDelete("{id:int}")]
    [ProducesResponseType(204)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var success = await _productService.DeleteProductAsync(id);
        if (!success)
            return NotFound();

        return NoContent();
    }
}
```

### 2.2 API 版本控制
```csharp
// 在 Program.cs 中配置
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-API-Version"),
        new MediaTypeApiVersionReader("version")
    );
});

builder.Services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// 版本特定的控制器
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductV2Dto>>> GetProducts()
    {
        // V2 版本的实现
        return Ok();
    }
}
```

### 2.3 API 文档 (Swagger)
```csharp
// 在 Program.cs 中配置
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "A sample API",
        Contact = new OpenApiContact
        {
            Name = "Your Name",
            Email = "your.email@example.com"
        }
    });

    // 添加JWT认证
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new string[] {}
        }
    });

    // 添加XML注释
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});
```

## 3. SignalR 实时通信

### 3.1 Hub 实现
```csharp
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;
    private readonly IUserService _userService;

    public ChatHub(ILogger<ChatHub> logger, IUserService userService)
    {
        _logger = logger;
        _userService = userService;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (!string.IsNullOrEmpty(userId))
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"User_{userId}");
            await Groups.AddToGroupAsync(Context.ConnectionId, "AllUsers");
        }

        _logger.LogInformation("Client connected: {ConnectionId}", Context.ConnectionId);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception exception)
    {
        _logger.LogInformation("Client disconnected: {ConnectionId}", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }

    public async Task SendMessage(string message)
    {
        var userId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var userName = Context.User?.Identity?.Name ?? "Anonymous";

        var chatMessage = new ChatMessage
        {
            UserId = userId,
            UserName = userName,
            Message = message,
            Timestamp = DateTime.UtcNow
        };

        // 发送给所有用户
        await Clients.Group("AllUsers").SendAsync("ReceiveMessage", chatMessage);

        // 记录消息
        await _userService.LogChatMessageAsync(chatMessage);
    }

    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("UserJoined", Context.User?.Identity?.Name, roomName);
    }

    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("UserLeft", Context.User?.Identity?.Name, roomName);
    }

    public async Task SendToUser(string userId, string message)
    {
        var chatMessage = new ChatMessage
        {
            UserId = Context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            UserName = Context.User?.Identity?.Name ?? "Anonymous",
            Message = message,
            Timestamp = DateTime.UtcNow
        };

        await Clients.Group($"User_{userId}").SendAsync("ReceivePrivateMessage", chatMessage);
    }
}
```

### 3.2 客户端集成
```csharp
// JavaScript 客户端
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chathub")
    .withAutomaticReconnect()
    .build();

connection.on("ReceiveMessage", (message) => {
    console.log(`Message from ${message.userName}: ${message.message}`);
    // 更新UI
});

connection.on("ReceivePrivateMessage", (message) => {
    console.log(`Private message from ${message.userName}: ${message.message}`);
    // 更新UI
});

connection.start()
    .then(() => console.log("Connected to SignalR"))
    .catch(err => console.error(err));

// 发送消息
function sendMessage() {
    const message = document.getElementById("messageInput").value;
    connection.invoke("SendMessage", message);
}

// .NET 客户端
public class ChatClient
{
    private HubConnection _connection;

    public async Task ConnectAsync()
    {
        _connection = new HubConnectionBuilder()
            .WithUrl("https://localhost:5001/chathub")
            .WithAutomaticReconnection()
            .Build();

        _connection.On<ChatMessage>("ReceiveMessage", message =>
        {
            Console.WriteLine($"Message from {message.UserName}: {message.Message}");
        });

        await _connection.StartAsync();
    }

    public async Task SendMessageAsync(string message)
    {
        await _connection.InvokeAsync("SendMessage", message);
    }
}
```

## 4. Blazor 服务器端和客户端

### 4.1 Blazor Server 组件
```csharp
@page "/counter"
@page "/counter/{InitialCount:int}"
@inject ILogger<Counter> Logger
@inject IJSRuntime JSRuntime

<h3>Counter</h3>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    [Parameter]
    public int InitialCount { get; set; }

    private int currentCount = 0;

    protected override void OnInitialized()
    {
        currentCount = InitialCount;
        Logger.LogInformation("Counter initialized with value: {Count}", currentCount);
    }

    private async Task IncrementCount()
    {
        currentCount++;
        Logger.LogInformation("Counter incremented to: {Count}", currentCount);
        
        // 调用JavaScript
        await JSRuntime.InvokeVoidAsync("console.log", $"Count is now: {currentCount}");
        
        // 触发UI更新
        StateHasChanged();
    }
}
```

### 4.2 Blazor WebAssembly 组件
```csharp
@page "/fetchdata"
@inject HttpClient Http
@inject ILogger<FetchData> Logger

<h1>Weather forecast</h1>

<p>This component demonstrates fetching data from the server.</p>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Temp. (F)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.TemperatureF</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private WeatherForecast[] forecasts;

    protected override async Task OnInitializedAsync()
    {
        try
        {
            forecasts = await Http.GetFromJsonAsync<WeatherForecast[]>("WeatherForecast");
            Logger.LogInformation("Loaded {Count} weather forecasts", forecasts?.Length ?? 0);
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Error loading weather forecasts");
        }
    }
}
```

### 4.3 状态管理
```csharp
// 状态容器
public class AppState
{
    public event Action OnChange;
    private int _counter;

    public int Counter
    {
        get => _counter;
        set
        {
            _counter = value;
            NotifyStateChanged();
        }
    }

    private void NotifyStateChanged() => OnChange?.Invoke();
}

// 在 Program.cs 中注册
builder.Services.AddScoped<AppState>();

// 在组件中使用
@inject AppState AppState
@implements IDisposable

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }

    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }

    private void IncrementCounter()
    {
        AppState.Counter++;
    }
}
```

## 5. Razor Pages

### 5.1 基本页面
```csharp
@page
@model CreateUserModel
@{
    ViewData["Title"] = "Create User";
}

<h1>Create User</h1>

<div class="row">
    <div class="col-md-6">
        <form method="post">
            <div asp-validation-summary="ModelOnly" class="text-danger"></div>
            
            <div class="form-group">
                <label asp-for="User.FirstName"></label>
                <input asp-for="User.FirstName" class="form-control" />
                <span asp-validation-for="User.FirstName" class="text-danger"></span>
            </div>
            
            <div class="form-group">
                <label asp-for="User.LastName"></label>
                <input asp-for="User.LastName" class="form-control" />
                <span asp-validation-for="User.LastName" class="text-danger"></span>
            </div>
            
            <div class="form-group">
                <label asp-for="User.Email"></label>
                <input asp-for="User.Email" class="form-control" />
                <span asp-validation-for="User.Email" class="text-danger"></span>
            </div>
            
            <button type="submit" class="btn btn-primary">Create</button>
        </form>
    </div>
</div>

@section Scripts {
    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}
}
```

### 5.2 页面模型
```csharp
public class CreateUserModel : PageModel
{
    private readonly IUserService _userService;
    private readonly ILogger<CreateUserModel> _logger;

    public CreateUserModel(IUserService userService, ILogger<CreateUserModel> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [BindProperty]
    public CreateUserRequest User { get; set; }

    public void OnGet()
    {
        // 页面加载时的逻辑
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        try
        {
            var userId = await _userService.CreateUserAsync(User);
            _logger.LogInformation("User created with ID: {UserId}", userId);
            
            TempData["Message"] = "User created successfully!";
            return RedirectToPage("./Index");
        }
        catch (ValidationException ex)
        {
            ModelState.AddModelError("", ex.Message);
            return Page();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating user");
            ModelState.AddModelError("", "An error occurred while creating the user.");
            return Page();
        }
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **MVC架构**：Model、View、Controller的职责分离
2. **Web API设计**：RESTful原则、版本控制、文档
3. **SignalR应用**：实时通信、Hub、客户端集成
4. **Blazor特点**：Server vs WebAssembly、组件生命周期
5. **Razor Pages vs MVC**：选择场景、优缺点

### 6.2 代码示例准备
- 自定义过滤器的实现
- API版本控制的配置
- SignalR Hub的实现
- Blazor组件的状态管理
- Razor Pages的表单处理

### 6.3 性能优化要点
- 使用异步方法避免阻塞
- 合理使用缓存和压缩
- 优化数据库查询
- 使用CDN加速静态资源
- 实现适当的错误处理和日志记录
