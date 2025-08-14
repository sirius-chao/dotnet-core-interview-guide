# 2. Web开发技术

## 2.1 ASP.NET Core

### MVC 架构模式

#### 模型绑定和验证
```csharp
public class UserModel
{
    [Required(ErrorMessage = "用户名是必填项")]
    [StringLength(50, MinimumLength = 3, ErrorMessage = "用户名长度必须在3-50个字符之间")]
    public string Username { get; set; }
    
    [Required(ErrorMessage = "邮箱是必填项")]
    [EmailAddress(ErrorMessage = "邮箱格式不正确")]
    public string Email { get; set; }
    
    [Range(18, 100, ErrorMessage = "年龄必须在18-100之间")]
    public int Age { get; set; }
}

public class UserController : Controller
{
    [HttpPost]
    public IActionResult Create(UserModel model)
    {
        if (!ModelState.IsValid)
        {
            return View(model);
        }
        
        // 处理有效数据
        return RedirectToAction("Index");
    }
}
```

#### 视图引擎和 Razor 语法
```csharp
@model List<UserModel>

@{
    ViewData["Title"] = "用户列表";
}

<h1>@ViewData["Title"]</h1>

@if (Model.Any())
{
    <table class="table">
        <thead>
            <tr>
                <th>用户名</th>
                <th>邮箱</th>
                <th>年龄</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var user in Model)
            {
                <tr>
                    <td>@user.Username</td>
                    <td>@user.Email</td>
                    <td>@user.Age</td>
                </tr>
            }
        </tbody>
    </table>
}
else
{
    <p>暂无用户数据</p>
}
```

#### 控制器职责分离
```csharp
// 控制器应该保持简洁，只负责处理HTTP请求
public class UserController : Controller
{
    private readonly IUserService _userService;
    private readonly ILogger<UserController> _logger;
    
    public UserController(IUserService userService, ILogger<UserController> logger)
    {
        _userService = userService;
        _logger = logger;
    }
    
    [HttpGet]
    public async Task<IActionResult> Index()
    {
        try
        {
            var users = await _userService.GetAllUsersAsync();
            return View(users);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "获取用户列表失败");
            return View("Error");
        }
    }
    
    [HttpPost]
    public async Task<IActionResult> Create(UserModel model)
    {
        if (!ModelState.IsValid)
        {
            return View(model);
        }
        
        try
        {
            await _userService.CreateUserAsync(model);
            TempData["Message"] = "用户创建成功";
            return RedirectToAction(nameof(Index));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "创建用户失败");
            ModelState.AddModelError("", "创建用户时发生错误");
            return View(model);
        }
    }
}
```

#### 过滤器 (Filters)
```csharp
// 自定义授权过滤器
public class CustomAuthorizeAttribute : AuthorizeAttribute, IAuthorizationFilter
{
    private readonly string _requiredRole;
    
    public CustomAuthorizeAttribute(string requiredRole)
    {
        _requiredRole = requiredRole;
    }
    
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var user = context.HttpContext.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }
        
        if (!user.IsInRole(_requiredRole))
        {
            context.Result = new ForbidResult();
            return;
        }
    }
}

// 自定义异常过滤器
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    
    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
    {
        _logger = logger;
    }
    
    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "未处理的异常");
        
        context.Result = new ViewResult
        {
            ViewName = "Error",
            ViewData = new ViewDataDictionary(new EmptyModelMetadataProvider(), new ModelStateDictionary())
            {
                Model = context.Exception
            }
        };
        
        context.ExceptionHandled = true;
    }
}

// 使用过滤器
[CustomAuthorize("Admin")]
public class AdminController : Controller
{
    // 只有Admin角色的用户才能访问
}
```

### Web API 设计

#### RESTful API 设计原则
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    
    public UsersController(IUserService userService)
    {
        _userService = userService;
    }
    
    // GET api/users - 获取所有用户
    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers([FromQuery] UserQueryParameters parameters)
    {
        var users = await _userService.GetUsersAsync(parameters);
        return Ok(users);
    }
    
    // GET api/users/{id} - 获取指定用户
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        if (user == null)
        {
            return NotFound();
        }
        return Ok(user);
    }
    
    // POST api/users - 创建新用户
    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser([FromBody] CreateUserRequest request)
    {
        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
    
    // PUT api/users/{id} - 更新用户
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(int id, [FromBody] UpdateUserRequest request)
    {
        var success = await _userService.UpdateUserAsync(id, request);
        if (!success)
        {
            return NotFound();
        }
        return NoContent();
    }
    
    // DELETE api/users/{id} - 删除用户
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(int id)
    {
        var success = await _userService.DeleteUserAsync(id);
        if (!success)
        {
            return NotFound();
        }
        return NoContent();
    }
}
```

#### 版本控制策略
```csharp
// 使用URL版本控制
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class UsersV1Controller : ControllerBase
{
    // V1版本的实现
}

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class UsersV2Controller : ControllerBase
{
    // V2版本的实现
}

// 在Program.cs中配置版本控制
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

builder.Services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});
```

#### 状态码使用
```csharp
// 正确的状态码使用
[HttpPost]
public async Task<IActionResult> CreateUser([FromBody] CreateUserRequest request)
{
    try
    {
        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
    catch (ValidationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "创建用户失败");
        return StatusCode(500, "内部服务器错误");
    }
}

[HttpPut("{id}")]
public async Task<IActionResult> UpdateUser(int id, [FromBody] UpdateUserRequest request)
{
    var success = await _userService.UpdateUserAsync(id, request);
    if (!success)
    {
        return NotFound($"用户ID {id} 不存在");
    }
    return NoContent(); // 204 No Content
}
```

#### API 文档生成
```csharp
// 在Program.cs中配置Swagger
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo 
    { 
        Title = "用户管理API", 
        Version = "v1",
        Description = "用户管理的RESTful API"
    });
    
    // 添加XML注释
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    c.IncludeXmlComments(xmlPath);
    
    // 添加认证
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });
});

// 在控制器中添加XML注释
/// <summary>
/// 获取所有用户
/// </summary>
/// <param name="parameters">查询参数</param>
/// <returns>用户列表</returns>
/// <response code="200">成功返回用户列表</response>
/// <response code="400">查询参数无效</response>
/// <response code="500">服务器内部错误</response>
[HttpGet]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[ProducesResponseType(StatusCodes.Status500InternalServerError)]
public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers([FromQuery] UserQueryParameters parameters)
{
    // 实现代码
}
```

### 中间件开发

#### 请求管道理解
```csharp
// 中间件的执行顺序很重要
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        var app = builder.Build();
        
        // 异常处理中间件应该最先执行
        app.UseExceptionHandler("/Error");
        
        // HTTPS重定向
        app.UseHttpsRedirection();
        
        // 静态文件服务
        app.UseStaticFiles();
        
        // 路由中间件
        app.UseRouting();
        
        // 认证中间件
        app.UseAuthentication();
        
        // 授权中间件
        app.UseAuthorization();
        
        // 端点映射
        app.MapControllers();
        
        app.Run();
    }
}
```

#### 自定义中间件开发
```csharp
// 性能监控中间件
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
        var sw = Stopwatch.StartNew();
        var originalBodyStream = context.Response.Body;
        
        try
        {
            using var memoryStream = new MemoryStream();
            context.Response.Body = memoryStream;
            
            await _next(context);
            
            sw.Stop();
            memoryStream.Position = 0;
            await memoryStream.CopyToAsync(originalBodyStream);
            
            _logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                sw.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
        finally
        {
            context.Response.Body = originalBodyStream;
        }
    }
}

// 扩展方法
public static class PerformanceMiddlewareExtensions
{
    public static IApplicationBuilder UsePerformanceMonitoring(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<PerformanceMiddleware>();
    }
}

// 使用中间件
app.UsePerformanceMonitoring();
```

#### 中间件短路
```csharp
// 健康检查中间件
public class HealthCheckMiddleware
{
    private readonly RequestDelegate _next;
    
    public HealthCheckMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            // 短路，直接返回健康状态
            context.Response.StatusCode = 200;
            await context.Response.WriteAsync("Healthy");
            return;
        }
        
        await _next(context);
    }
}

// 维护模式中间件
public class MaintenanceModeMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;
    
    public MaintenanceModeMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var isMaintenanceMode = _configuration.GetValue<bool>("MaintenanceMode:Enabled");
        
        if (isMaintenanceMode && !context.Request.Path.StartsWithSegments("/admin"))
        {
            context.Response.StatusCode = 503;
            context.Response.ContentType = "text/html";
            await context.Response.WriteAsync(@"
                <html>
                    <body>
                        <h1>系统维护中</h1>
                        <p>系统正在进行维护，请稍后再试。</p>
                    </body>
                </html>");
            return;
        }
        
        await _next(context);
    }
}
```

### 身份认证与授权

#### JWT 令牌认证
```csharp
// JWT配置
public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; }
}

// JWT服务
public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;
    
    public JwtService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }
    
    public string GenerateToken(User user)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Email, user.Email)
        };
        
        // 添加角色声明
        foreach (var role in user.Roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public ClaimsPrincipal ValidateToken(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_jwtSettings.SecretKey);
        
        var validationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = _jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = _jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
        
        try
        {
            var principal = tokenHandler.ValidateToken(token, validationParameters, out var validatedToken);
            return principal;
        }
        catch
        {
            return null;
        }
    }
}

// 在Program.cs中配置JWT认证
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("Jwt"));
builder.Services.AddScoped<IJwtService, JwtService>();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"])),
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });

// 使用认证
app.UseAuthentication();
app.UseAuthorization();
```

#### 基于角色的授权
```csharp
// 角色授权
[Authorize(Roles = "Admin")]
public class AdminController : Controller
{
    [HttpGet]
    public IActionResult Index()
    {
        return View();
    }
}

// 多个角色
[Authorize(Roles = "Admin,Manager")]
public class ManagementController : Controller
{
    // 只有Admin或Manager角色的用户才能访问
}

// 在控制器方法上使用
public class UserController : Controller
{
    [Authorize(Roles = "Admin")]
    public IActionResult Delete(int id)
    {
        // 只有Admin角色才能删除用户
    }
    
    [Authorize(Roles = "Admin,Manager")]
    public IActionResult Edit(int id)
    {
        // Admin和Manager角色都能编辑用户
    }
}
```

#### 基于策略的授权
```csharp
// 定义策略
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        
        builder.Services.AddAuthorization(options =>
        {
            // 年龄策略
            options.AddPolicy("MinimumAge", policy =>
                policy.RequireAssertion(context =>
                    context.User.HasClaim(c => c.Type == "Age" && 
                        int.TryParse(c.Value, out var age) && age >= 18)));
            
            // 部门策略
            options.AddPolicy("SameDepartment", policy =>
                policy.RequireAssertion(context =>
                {
                    var userDept = context.User.FindFirst("Department")?.Value;
                    var resourceDept = context.Resource as string;
                    return userDept == resourceDept;
                }));
            
            // 自定义策略
            options.AddPolicy("CanEditUser", policy =>
                policy.RequireAssertion(context =>
                {
                    var user = context.User;
                    var resourceId = context.Resource as int?;
                    
                    // 管理员可以编辑任何用户
                    if (user.IsInRole("Admin"))
                        return true;
                    
                    // 用户只能编辑自己的信息
                    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
                    return userId == resourceId?.ToString();
                }));
        });
        
        var app = builder.Build();
        app.UseAuthorization();
        app.MapControllers();
        app.Run();
    }
}

// 使用策略
public class UserController : Controller
{
    [Authorize(Policy = "MinimumAge")]
    public IActionResult AdultOnly()
    {
        return View();
    }
    
    [Authorize(Policy = "CanEditUser")]
    public IActionResult Edit(int id)
    {
        // 使用策略进行授权
        return View();
    }
}
```

#### 自定义授权处理器
```csharp
// 自定义授权要求
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    
    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// 自定义授权处理器
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        MinimumAgeRequirement requirement)
    {
        var ageClaim = context.User.FindFirst("Age");
        if (ageClaim == null)
        {
            return Task.CompletedTask;
        }
        
        if (int.TryParse(ageClaim.Value, out var age) && age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

// 注册处理器
builder.Services.AddScoped<IAuthorizationHandler, MinimumAgeHandler>();

// 使用自定义要求
[Authorize(Policy = "MinimumAge18")]
public class AdultController : Controller
{
    // 只有18岁以上的用户才能访问
}
```

### 路由配置

#### 属性路由
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public IActionResult GetProducts()
    {
        return Ok();
    }
    
    // GET api/products/5
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        return Ok();
    }
    
    // GET api/products/5/details
    [HttpGet("{id}/details")]
    public IActionResult GetProductDetails(int id)
    {
        return Ok();
    }
    
    // POST api/products
    [HttpPost]
    public IActionResult CreateProduct([FromBody] Product product)
    {
        return Ok();
    }
    
    // PUT api/products/5
    [HttpPut("{id}")]
    public IActionResult UpdateProduct(int id, [FromBody] Product product)
    {
        return Ok();
    }
    
    // DELETE api/products/5
    [HttpDelete("{id}")]
    public IActionResult DeleteProduct(int id)
    {
        return Ok();
    }
}
```

#### 约定路由
```csharp
// 在Program.cs中配置约定路由
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.MapControllerRoute(
    name: "blog",
    pattern: "blog/{*article}",
    defaults: new { controller = "Blog", action = "Article" });

app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

#### 路由约束
```csharp
[HttpGet("{id:int:min(1)}")]
public IActionResult GetProduct(int id)
{
    // id必须是大于等于1的整数
}

[HttpGet("{name:alpha:minlength(3)}")]
public IActionResult GetProductByName(string name)
{
    // name必须是至少3个字符的字母
}

[HttpGet("{category:regex(^[A-Za-z]+$)}")]
public IActionResult GetProductsByCategory(string category)
{
    // category必须只包含字母
}

// 自定义路由约束
public class ValidIdConstraint : IRouteConstraint
{
    public bool Match(HttpContext httpContext, IRouter router, string routeKey, 
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var stringValue = Convert.ToString(value);
            return !string.IsNullOrEmpty(stringValue) && 
                   stringValue.Length == 6 && 
                   stringValue.All(char.IsDigit);
        }
        return false;
    }
}

// 注册自定义约束
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap["validId"] = typeof(ValidIdConstraint);
});

// 使用自定义约束
[HttpGet("{id:validId}")]
public IActionResult GetProduct(string id)
{
    // id必须是6位数字
}
```

#### 区域路由
```csharp
// 创建区域
[Area("Admin")]
public class AdminController : Controller
{
    [Route("admin/dashboard")]
    public IActionResult Dashboard()
    {
        return View();
    }
    
    [Route("admin/users")]
    public IActionResult Users()
    {
        return View();
    }
}

// 区域路由配置
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

### 模型绑定与验证

#### 自定义模型绑定器
```csharp
// 自定义模型绑定器
public class CustomModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        if (bindingContext == null)
        {
            throw new ArgumentNullException(nameof(bindingContext));
        }
        
        var valueProviderResult = bindingContext.ValueProvider.GetValue(bindingContext.ModelName);
        if (valueProviderResult == ValueProviderResult.None)
        {
            return Task.CompletedTask;
        }
        
        bindingContext.ModelState.SetModelValue(bindingContext.ModelName, valueProviderResult);
        
        var value = valueProviderResult.FirstValue;
        if (string.IsNullOrEmpty(value))
        {
            return Task.CompletedTask;
        }
        
        // 自定义绑定逻辑
        if (int.TryParse(value, out var intValue))
        {
            bindingContext.Result = ModelBindingResult.Success(intValue);
        }
        else
        {
            bindingContext.ModelState.TryAddModelError(bindingContext.ModelName, 
                "值必须是有效的整数");
        }
        
        return Task.CompletedTask;
    }
}

// 使用自定义绑定器
public class Product
{
    [ModelBinder(typeof(CustomModelBinder))]
    public int Id { get; set; }
    
    public string Name { get; set; }
}
```

#### 数据注解验证
```csharp
public class UserModel
{
    [Required(ErrorMessage = "用户名是必填项")]
    [StringLength(50, MinimumLength = 3, ErrorMessage = "用户名长度必须在3-50个字符之间")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "用户名只能包含字母、数字和下划线")]
    public string Username { get; set; }
    
    [Required(ErrorMessage = "邮箱是必填项")]
    [EmailAddress(ErrorMessage = "邮箱格式不正确")]
    public string Email { get; set; }
    
    [Required(ErrorMessage = "密码是必填项")]
    [StringLength(100, MinimumLength = 8, ErrorMessage = "密码长度必须在8-100个字符之间")]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]", 
        ErrorMessage = "密码必须包含至少一个大写字母、一个小写字母、一个数字和一个特殊字符")]
    public string Password { get; set; }
    
    [Compare("Password", ErrorMessage = "确认密码与密码不匹配")]
    public string ConfirmPassword { get; set; }
    
    [Range(18, 100, ErrorMessage = "年龄必须在18-100之间")]
    public int Age { get; set; }
    
    [Phone(ErrorMessage = "电话号码格式不正确")]
    public string PhoneNumber { get; set; }
    
    [Url(ErrorMessage = "网站地址格式不正确")]
    public string Website { get; set; }
    
    [CreditCard(ErrorMessage = "信用卡号格式不正确")]
    public string CreditCardNumber { get; set; }
}
```

#### 自定义验证特性
```csharp
// 自定义验证特性
public class CustomValidationAttribute : ValidationAttribute
{
    private readonly string _otherProperty;
    
    public CustomValidationAttribute(string otherProperty)
    {
        _otherProperty = otherProperty;
    }
    
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        var otherPropertyInfo = validationContext.ObjectType.GetProperty(_otherProperty);
        if (otherPropertyInfo == null)
        {
            return new ValidationResult($"找不到属性 {_otherProperty}");
        }
        
        var otherValue = otherPropertyInfo.GetValue(validationContext.ObjectInstance);
        
        // 自定义验证逻辑
        if (value != null && otherValue != null)
        {
            if (value.ToString().Equals(otherValue.ToString(), StringComparison.OrdinalIgnoreCase))
            {
                return new ValidationResult("两个字段不能相同");
            }
        }
        
        return ValidationResult.Success;
    }
}

// 使用自定义验证特性
public class UserModel
{
    public string Username { get; set; }
    
    [CustomValidation("Username")]
    public string Nickname { get; set; }
}

// 自定义远程验证
public class RemoteValidationAttribute : ValidationAttribute
{
    private readonly string _action;
    private readonly string _controller;
    
    public RemoteValidationAttribute(string action, string controller)
    {
        _action = action;
        _controller = controller;
    }
    
    public override string FormatErrorMessage(string name)
    {
        return string.Format(ErrorMessageString, name);
    }
    
    public override bool IsValid(object value)
    {
        // 这里通常需要调用远程验证
        return true;
    }
}
```

## 2.2 前端集成

### SignalR 实时通信

#### Hub 类设计
```csharp
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;
    
    public ChatHub(ILogger<ChatHub> logger)
    {
        _logger = logger;
    }
    
    public override async Task OnConnectedAsync()
    {
        _logger.LogInformation($"用户 {Context.ConnectionId} 已连接");
        await Clients.All.SendAsync("UserConnected", Context.ConnectionId);
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        _logger.LogInformation($"用户 {Context.ConnectionId} 已断开连接");
        await Clients.All.SendAsync("UserDisconnected", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
    
    public async Task SendMessage(string user, string message)
    {
        _logger.LogInformation($"用户 {user} 发送消息: {message}");
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
    
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserJoinedGroup", Context.ConnectionId, groupName);
    }
    
    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).SendAsync("UserLeftGroup", Context.ConnectionId, groupName);
    }
    
    public async Task SendToGroup(string groupName, string user, string message)
    {
        await Clients.Group(groupName).SendAsync("ReceiveGroupMessage", user, message);
    }
    
    public async Task SendToUser(string userId, string user, string message)
    {
        await Clients.User(userId).SendAsync("ReceivePrivateMessage", user, message);
    }
}
```

#### 连接管理
```csharp
// 在Program.cs中配置SignalR
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = true;
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.MaximumReceiveMessageSize = 1024 * 1024; // 1MB
});

// 映射Hub
app.MapHub<ChatHub>("/chatHub");

// 客户端连接管理
public class ChatHub : Hub
{
    private static readonly Dictionary<string, string> _userConnections = new();
    
    public async Task RegisterUser(string username)
    {
        _userConnections[Context.ConnectionId] = username;
        await Clients.All.SendAsync("UserRegistered", username);
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        if (_userConnections.TryGetValue(Context.ConnectionId, out var username))
        {
            _userConnections.Remove(Context.ConnectionId);
            await Clients.All.SendAsync("UserDisconnected", username);
        }
        await base.OnDisconnectedAsync(exception);
    }
    
    public string GetCurrentUser()
    {
        return _userConnections.TryGetValue(Context.ConnectionId, out var username) ? username : null;
    }
}
```

#### 组管理
```csharp
public class ChatHub : Hub
{
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("UserJoinedRoom", Context.ConnectionId, roomName);
        
        // 发送房间历史消息
        var history = await GetRoomHistory(roomName);
        await Clients.Caller.SendAsync("RoomHistory", history);
    }
    
    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).SendAsync("UserLeftRoom", Context.ConnectionId, roomName);
    }
    
    public async Task SendToRoom(string roomName, string message)
    {
        var user = GetCurrentUser();
        await Clients.Group(roomName).SendAsync("RoomMessage", user, message);
        
        // 保存消息到数据库
        await SaveMessageToDatabase(roomName, user, message);
    }
    
    public async Task GetRoomUsers(string roomName)
    {
        var users = await GetUsersInRoom(roomName);
        await Clients.Caller.SendAsync("RoomUsers", users);
    }
}
```

#### 扩展性考虑
```csharp
// 使用Redis作为SignalR的后端存储
builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.Configuration.ConnectionString = "localhost:6379";
        options.Configuration.ChannelPrefix = "ChatHub";
    });

// 使用Azure SignalR Service
builder.Services.AddSignalR()
    .AddAzureSignalR("your-connection-string");

// 自定义Hub过滤器
public class LoggingHubFilter : IHubFilter
{
    private readonly ILogger<LoggingHubFilter> _logger;
    
    public LoggingHubFilter(ILogger<LoggingHubFilter> logger)
    {
        _logger = logger;
    }
    
    public async ValueTask<object> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object>> next)
    {
        _logger.LogInformation($"调用方法: {invocationContext.HubMethodName}");
        
        try
        {
            var result = await next(invocationContext);
            _logger.LogInformation($"方法 {invocationContext.HubMethodName} 执行成功");
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, $"方法 {invocationContext.HubMethodName} 执行失败");
            throw;
        }
    }
    
    public Task OnConnectedAsync(HubLifetimeContext context, Func<HubLifetimeContext, Task> next)
    {
        _logger.LogInformation($"用户 {context.Hub.Context.ConnectionId} 连接");
        return next(context);
    }
    
    public Task OnDisconnectedAsync(HubLifetimeContext context, Exception exception, Func<HubLifetimeContext, Exception, Task> next)
    {
        _logger.LogInformation($"用户 {context.Hub.Context.ConnectionId} 断开连接");
        return next(context, exception);
    }
}

// 注册过滤器
builder.Services.AddSignalR()
    .AddHubFilter<LoggingHubFilter>();
```

### Blazor 组件开发

#### 组件生命周期
```csharp
@page "/counter"
@inject ILogger<Counter> Logger

<h1>Counter</h1>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;
    
    protected override void OnInitialized()
    {
        Logger.LogInformation("Counter组件已初始化");
    }
    
    protected override async Task OnInitializedAsync()
    {
        // 异步初始化
        await Task.Delay(100);
        Logger.LogInformation("Counter组件异步初始化完成");
    }
    
    protected override void OnParametersSet()
    {
        Logger.LogInformation("Counter组件参数已设置");
    }
    
    protected override async Task OnParametersSetAsync()
    {
        // 异步参数设置
        await Task.Delay(100);
        Logger.LogInformation("Counter组件异步参数设置完成");
    }
    
    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            Logger.LogInformation("Counter组件首次渲染完成");
        }
        else
        {
            Logger.LogInformation("Counter组件重新渲染完成");
        }
    }
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await Task.Delay(100);
            Logger.LogInformation("Counter组件异步首次渲染完成");
        }
    }
    
    private void IncrementCount()
    {
        currentCount++;
        StateHasChanged(); // 手动触发重新渲染
    }
    
    public void Dispose()
    {
        Logger.LogInformation("Counter组件已销毁");
    }
}
```

#### 状态管理
```csharp
// 状态管理服务
public class AppState
{
    public event Action OnChange;
    
    private string _currentUser;
    public string CurrentUser
    {
        get => _currentUser;
        set
        {
            _currentUser = value;
            NotifyStateChanged();
        }
    }
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}

// 注册服务
builder.Services.AddScoped<AppState>();

// 在组件中使用
@inject AppState AppState
@implements IDisposable

<h3>欢迎, @AppState.CurrentUser!</h3>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }
    
    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
    
    private void UpdateUser()
    {
        AppState.CurrentUser = "新用户";
    }
}

// 使用CascadingValue进行状态传递
<CascadingValue Value="AppState">
    <UserProfile />
</CascadingValue>

// 子组件
@inject AppState AppState

<p>当前用户: @AppState.CurrentUser</p>
```

#### 组件间通信
```csharp
// 父组件
@page "/parent"

<h3>父组件</h3>

<ChildComponent @bind-Value="parentValue" OnValueChanged="HandleValueChanged" />

<p>父组件值: @parentValue</p>

@code {
    private string parentValue = "初始值";
    
    private void HandleValueChanged(string newValue)
    {
        parentValue = newValue;
        Console.WriteLine($"值已更改为: {newValue}");
    }
}

// 子组件
<h4>子组件</h4>

<input @bind="childValue" @bind:event="oninput" />

<button @onclick="UpdateParent">更新父组件</button>

@code {
    [Parameter]
    public string Value { get; set; }
    
    [Parameter]
    public EventCallback<string> ValueChanged { get; set; }
    
    [Parameter]
    public EventCallback<string> OnValueChanged { get; set; }
    
    private string childValue;
    
    protected override void OnParametersSet()
    {
        childValue = Value;
    }
    
    private async Task UpdateParent()
    {
        await ValueChanged.InvokeAsync(childValue);
        await OnValueChanged.InvokeAsync(childValue);
    }
}

// 使用事件回调
public class EventCallbackExample
{
    [Parameter]
    public EventCallback<string> OnSave { get; set; }
    
    private async Task SaveData()
    {
        var data = "保存的数据";
        await OnSave.InvokeAsync(data);
    }
}
```

#### 性能优化
```csharp
// 使用ShouldRender优化渲染
@code {
    protected override bool ShouldRender()
    {
        // 只有在特定条件下才重新渲染
        return shouldRender;
    }
}

// 使用@key优化列表渲染
@foreach (var item in items)
{
    <ListItem @key="item.Id" Item="item" />
}

// 使用Virtualize优化长列表
<Virtualize Items="items" Context="item">
    <ListItem Item="item" />
</Virtualize>

// 使用Lazy加载组件
@if (shouldLoadComponent)
{
    <LazyComponent />
}

// 使用CSS隔离
@page "/counter"
@attribute [CssIsolation]

<style>
    .counter {
        color: blue;
    }
</style>

<h1 class="counter">Counter</h1>
```

### Razor 页面

#### 页面模型
```csharp
// 页面模型
public class ContactModel : PageModel
{
    [BindProperty]
    public ContactForm Contact { get; set; }
    
    [BindProperty(SupportsGet = true)]
    public string Message { get; set; }
    
    public void OnGet()
    {
        // GET请求处理
    }
    
    public IActionResult OnPost()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }
        
        // 处理表单提交
        return RedirectToPage("./ThankYou", new { message = "表单提交成功" });
    }
    
    public IActionResult OnPostDelete(int id)
    {
        // 处理删除操作
        return RedirectToPage();
    }
}

// 页面文件
@page
@model ContactModel
@{
    ViewData["Title"] = "联系我们";
}

<h1>联系我们</h1>

@if (!string.IsNullOrEmpty(Model.Message))
{
    <div class="alert alert-success">@Model.Message</div>
}

<form method="post">
    <div class="form-group">
        <label asp-for="Contact.Name">姓名</label>
        <input asp-for="Contact.Name" class="form-control" />
        <span asp-validation-for="Contact.Name" class="text-danger"></span>
    </div>
    
    <div class="form-group">
        <label asp-for="Contact.Email">邮箱</label>
        <input asp-for="Contact.Email" class="form-control" />
        <span asp-validation-for="Contact.Email" class="text-danger"></span>
    </div>
    
    <button type="submit" class="btn btn-primary">提交</button>
</form>

@section Scripts {
    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}
}
```

#### 页面路由
```csharp
// 自定义路由
@page "/contact-us"
@model ContactModel

// 带参数的页面
@page "/user/{id:int}"
@model UserProfileModel

@code {
    [BindProperty(SupportsGet = true)]
    public int Id { get; set; }
}

// 可选参数
@page "/blog/{slug?}"
@model BlogPostModel

@code {
    [BindProperty(SupportsGet = true)]
    public string Slug { get; set; } = "default";
}

// 多个参数
@page "/product/{category}/{id:int}"
@model ProductModel

@code {
    [BindProperty(SupportsGet = true)]
    public string Category { get; set; }
    
    [BindProperty(SupportsGet = true)]
    public int Id { get; set; }
}
```

#### 页面过滤器
```csharp
// 页面过滤器
public class LoggingPageFilter : IPageFilter
{
    private readonly ILogger<LoggingPageFilter> _logger;
    
    public LoggingPageFilter(ILogger<LoggingPageFilter> logger)
    {
        _logger = logger;
    }
    
    public void OnPageHandlerExecuting(PageHandlerExecutingContext context)
    {
        _logger.LogInformation($"页面 {context.ActionDescriptor.RelativePath} 开始执行");
    }
    
    public void OnPageHandlerExecuted(PageHandlerExecutedContext context)
    {
        _logger.LogInformation($"页面 {context.ActionDescriptor.RelativePath} 执行完成");
    }
    
    public void OnPageHandlerSelected(PageHandlerSelectedContext context)
    {
        _logger.LogInformation($"页面处理器已选择: {context.HandlerMethod?.Name}");
    }
}

// 注册过滤器
builder.Services.AddMvc(options =>
{
    options.Filters.Add<LoggingPageFilter>();
});

// 在页面模型上使用过滤器
[ServiceFilter(typeof(LoggingPageFilter))]
public class ContactModel : PageModel
{
    // 页面实现
}
```

## 面试重点总结

### 高频面试题
1. **MVC模式中各个组件的职责是什么？**
2. **如何实现自定义中间件？**
3. **JWT认证的流程是什么？**
4. **如何实现基于策略的授权？**
5. **SignalR的实时通信原理是什么？**

### 代码示例准备
- 自定义中间件的实现
- JWT认证的完整流程
- 基于策略的授权实现
- SignalR Hub的开发
- Blazor组件的生命周期管理

### 性能优化要点
- 合理使用缓存
- 避免N+1查询问题
- 使用异步编程
- 合理配置中间件顺序
- 优化Blazor组件渲染
