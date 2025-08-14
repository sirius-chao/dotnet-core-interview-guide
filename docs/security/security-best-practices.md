# 安全与最佳实践

## 1. 身份认证与授权

### 1.1 ASP.NET Core Identity

#### 用户管理
```csharp
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
    public string Department { get; set; }
    
    // 自定义属性
    public virtual ICollection<UserClaim> UserClaims { get; set; }
    public virtual ICollection<UserRole> UserRoles { get; set; }
}

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // 配置用户表
        builder.Entity<ApplicationUser>(entity =>
        {
            entity.ToTable("Users");
            entity.Property(e => e.FirstName).HasMaxLength(100);
            entity.Property(e => e.LastName).HasMaxLength(100);
            entity.Property(e => e.Department).HasMaxLength(50);
        });
    }
}
```

#### 角色管理
```csharp
public class RoleService : IRoleService
{
    private readonly RoleManager<IdentityRole> _roleManager;
    private readonly UserManager<ApplicationUser> _userManager;
    
    public async Task<IdentityResult> CreateRoleAsync(string roleName, string description = null)
    {
        if (await _roleManager.RoleExistsAsync(roleName))
        {
            return IdentityResult.Failed(new IdentityError
            {
                Description = "角色已存在"
            });
        }
        
        var role = new IdentityRole(roleName);
        var result = await _roleManager.CreateAsync(role);
        
        if (result.Succeeded && !string.IsNullOrEmpty(description))
        {
            await _roleManager.AddClaimAsync(role, new Claim("Description", description));
        }
        
        return result;
    }
    
    public async Task<bool> AssignUserToRoleAsync(string userId, string roleName)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null)
            return false;
            
        var result = await _userManager.AddToRoleAsync(user, roleName);
        return result.Succeeded;
    }
    
    public async Task<IList<string>> GetUserRolesAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null)
            return new List<string>();
            
        return await _userManager.GetRolesAsync(user);
    }
}
```

### 1.2 JWT 认证

#### JWT 配置
```csharp
public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; }
}

public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;
    
    public JwtService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }
    
    public string GenerateToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id),
            new Claim(ClaimTypes.Name, user.UserName),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim("FirstName", user.FirstName),
            new Claim("LastName", user.LastName)
        };
        
        // 添加角色声明
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.SecretKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: creds
        );
        
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
```

#### 认证中间件配置
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // JWT 配置
        services.Configure<JwtSettings>(Configuration.GetSection("Jwt"));
        
        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(Configuration["Jwt:SecretKey"])),
                ValidateIssuer = true,
                ValidIssuer = Configuration["Jwt:Issuer"],
                ValidateAudience = true,
                ValidAudience = Configuration["Jwt:Audience"],
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            };
            
            options.Events = new JwtBearerEvents
            {
                OnAuthenticationFailed = context =>
                {
                    if (context.Exception.GetType() == typeof(SecurityTokenExpiredException))
                    {
                        context.Response.Headers.Add("Token-Expired", "true");
                    }
                    return Task.CompletedTask;
                }
            };
        });
        
        // 授权策略
        services.AddAuthorization(options =>
        {
            options.AddPolicy("AdminOnly", policy =>
                policy.RequireRole("Admin"));
                
            options.AddPolicy("UserManagement", policy =>
                policy.RequireRole("Admin", "UserManager"));
                
            options.AddPolicy("MinimumAge", policy =>
                policy.RequireAssertion(context =>
                    context.User.HasClaim(c => c.Type == "Age" && 
                        int.TryParse(c.Value, out var age) && age >= 18)));
        });
    }
}
```

### 1.3 授权策略

#### 基于策略的授权
```csharp
[Authorize(Policy = "AdminOnly")]
public class AdminController : ControllerBase
{
    [HttpGet("users")]
    public async Task<IActionResult> GetUsers()
    {
        // 只有管理员可以访问
        return Ok(await _userService.GetAllUsersAsync());
    }
}

[Authorize(Policy = "UserManagement")]
public class UserManagementController : ControllerBase
{
    [HttpPost("users")]
    public async Task<IActionResult> CreateUser([FromBody] CreateUserRequest request)
    {
        // 管理员和用户管理员都可以访问
        var result = await _userService.CreateUserAsync(request);
        return Ok(result);
    }
}

// 自定义授权特性
public class MinimumAgeAttribute : AuthorizeAttribute
{
    public MinimumAgeAttribute(int minimumAge)
    {
        Policy = $"MinimumAge{minimumAge}";
    }
}

[MinimumAge(21)]
public class AlcoholController : ControllerBase
{
    [HttpGet("products")]
    public IActionResult GetProducts()
    {
        // 需要21岁以上
        return Ok(_alcoholService.GetProducts());
    }
}
```

## 2. 数据安全

### 2.1 数据加密

#### 敏感数据加密
```csharp
public class EncryptionService : IEncryptionService
{
    private readonly byte[] _key;
    private readonly byte[] _iv;
    
    public EncryptionService(IConfiguration configuration)
    {
        var secretKey = configuration["Encryption:Key"];
        var secretIv = configuration["Encryption:IV"];
        
        _key = Convert.FromBase64String(secretKey);
        _iv = Convert.FromBase64String(secretIv);
    }
    
    public string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText))
            return plainText;
            
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;
        
        using var encryptor = aes.CreateEncryptor();
        using var msEncrypt = new MemoryStream();
        using var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write);
        using var swEncrypt = new StreamWriter(csEncrypt);
        
        swEncrypt.Write(plainText);
        swEncrypt.Flush();
        csEncrypt.FlushFinalBlock();
        
        return Convert.ToBase64String(msEncrypt.ToArray());
    }
    
    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText))
            return cipherText;
            
        using var aes = Aes.Create();
        aes.Key = _key;
        aes.IV = _iv;
        
        using var decryptor = aes.CreateDecryptor();
        using var msDecrypt = new MemoryStream(Convert.FromBase64String(cipherText));
        using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
        using var srDecrypt = new StreamReader(csDecrypt);
        
        return srDecrypt.ReadToEnd();
    }
}

// 在实体中使用
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    
    private string _encryptedEmail;
    public string Email
    {
        get => _encryptionService?.Decrypt(_encryptedEmail) ?? _encryptedEmail;
        set => _encryptedEmail = _encryptionService?.Encrypt(value) ?? value;
    }
    
    private readonly IEncryptionService _encryptionService;
    
    public User(IEncryptionService encryptionService)
    {
        _encryptionService = encryptionService;
    }
}
```

#### 密码哈希
```csharp
public class PasswordHasher : IPasswordHasher
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password, BCrypt.Net.BCrypt.GenerateSalt(12));
    }
    
    public bool VerifyPassword(string password, string hashedPassword)
    {
        return BCrypt.Net.BCrypt.Verify(password, hashedPassword);
    }
    
    public bool NeedsRehash(string hashedPassword)
    {
        return BCrypt.Net.BCrypt.PasswordNeedsRehash(hashedPassword, 12);
    }
}

public class UserService : IUserService
{
    private readonly IPasswordHasher _passwordHasher;
    
    public async Task<IdentityResult> CreateUserAsync(CreateUserRequest request)
    {
        var user = new ApplicationUser
        {
            UserName = request.Username,
            Email = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName
        };
        
        // 哈希密码
        var hashedPassword = _passwordHasher.HashPassword(request.Password);
        var result = await _userManager.CreateAsync(user, hashedPassword);
        
        return result;
    }
    
    public async Task<bool> ValidateCredentialsAsync(string username, string password)
    {
        var user = await _userManager.FindByNameAsync(username);
        if (user == null)
            return false;
            
        return _passwordHasher.VerifyPassword(password, user.PasswordHash);
    }
}
```

### 2.2 SQL 注入防护

#### 参数化查询
```csharp
public class UserRepository : IUserRepository
{
    private readonly DbContext _context;
    
    public async Task<User> GetUserByEmailAsync(string email)
    {
        // 使用参数化查询防止 SQL 注入
        return await _context.Users
            .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email)
            .FirstOrDefaultAsync();
    }
    
    public async Task<IEnumerable<User>> SearchUsersAsync(string searchTerm)
    {
        // 使用 EF Core 的 LINQ 查询，自动参数化
        return await _context.Users
            .Where(u => u.FirstName.Contains(searchTerm) || 
                       u.LastName.Contains(searchTerm) ||
                       u.Email.Contains(searchTerm))
            .ToListAsync();
    }
    
    // 动态查询构建
    public async Task<IEnumerable<User>> GetUsersWithFilterAsync(UserFilter filter)
    {
        var query = _context.Users.AsQueryable();
        
        if (!string.IsNullOrEmpty(filter.FirstName))
        {
            query = query.Where(u => u.FirstName.Contains(filter.FirstName));
        }
        
        if (!string.IsNullOrEmpty(filter.LastName))
        {
            query = query.Where(u => u.LastName.Contains(filter.LastName));
        }
        
        if (filter.MinAge.HasValue)
        {
            query = query.Where(u => u.Age >= filter.MinAge.Value);
        }
        
        return await query.ToListAsync();
    }
}
```

## 3. 网络安全

### 3.1 HTTPS 配置

#### SSL/TLS 配置
```csharp
public class Startup
{
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // 强制 HTTPS
        app.UseHttpsRedirection();
        
        // HSTS 配置
        app.UseHsts();
        
        // 安全头配置
        app.Use(async (context, next) =>
        {
            context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
            context.Response.Headers.Add("X-Frame-Options", "DENY");
            context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
            context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
            context.Response.Headers.Add("Content-Security-Policy", 
                "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';");
            
            await next();
        });
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

#### 证书管理
```csharp
public class CertificateService : ICertificateService
{
    public X509Certificate2 GetCertificate(string thumbprint)
    {
        using var store = new X509Store(StoreName.My, StoreLocation.CurrentUser);
        store.Open(OpenFlags.ReadOnly);
        
        var certificates = store.Certificates.Find(
            X509FindType.FindByThumbprint, 
            thumbprint, 
            false);
            
        return certificates.Count > 0 ? certificates[0] : null;
    }
    
    public bool ValidateCertificate(X509Certificate2 certificate)
    {
        try
        {
            // 检查证书是否过期
            if (DateTime.Now < certificate.NotBefore || DateTime.Now > certificate.NotAfter)
                return false;
                
            // 检查证书链
            var chain = new X509Chain();
            chain.Build(certificate);
            
            foreach (var element in chain.ChainElements)
            {
                if (element.Certificate.Verify())
                    continue;
                    
                // 检查吊销状态
                var revocationList = new X509RevocationList();
                if (revocationList.IsRevoked(element.Certificate))
                    return false;
            }
            
            return true;
        }
        catch
        {
            return false;
        }
    }
}
```

### 3.2 CORS 配置

#### 跨域策略
```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddCors(options =>
        {
            options.AddPolicy("AllowSpecificOrigin", policy =>
            {
                policy.WithOrigins("https://trusted-site.com")
                      .AllowAnyMethod()
                      .AllowAnyHeader()
                      .AllowCredentials();
            });
            
            options.AddPolicy("AllowAnyOrigin", policy =>
            {
                policy.AllowAnyOrigin()
                      .AllowAnyMethod()
                      .AllowAnyHeader();
            });
            
            options.AddPolicy("RestrictedPolicy", policy =>
            {
                policy.WithOrigins("https://api.example.com")
                      .WithMethods("GET", "POST")
                      .WithHeaders("Authorization", "Content-Type")
                      .SetPreflightMaxAge(TimeSpan.FromHours(1));
            });
        });
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // 全局 CORS 策略
        app.UseCors("AllowSpecificOrigin");
        
        // 或者针对特定控制器/操作
        app.UseRouting();
        app.UseCors();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}

// 在控制器中使用
[ApiController]
[Route("api/[controller]")]
[EnableCors("RestrictedPolicy")]
public class ApiController : ControllerBase
{
    [HttpGet]
    [EnableCors("AllowAnyOrigin")]
    public IActionResult Get()
    {
        return Ok("Public data");
    }
    
    [HttpPost]
    public IActionResult Post([FromBody] object data)
    {
        // 使用控制器级别的 CORS 策略
        return Ok("Data received");
    }
}
```

## 4. 输入验证与清理

### 4.1 模型验证

#### 数据注解验证
```csharp
public class CreateUserRequest
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
        ErrorMessage = "密码必须包含大小写字母、数字和特殊字符")]
    public string Password { get; set; }
    
    [Required(ErrorMessage = "确认密码是必填项")]
    [Compare("Password", ErrorMessage = "两次输入的密码不一致")]
    public string ConfirmPassword { get; set; }
    
    [Required(ErrorMessage = "姓名是必填项")]
    [StringLength(100, ErrorMessage = "姓名长度不能超过100个字符")]
    public string FirstName { get; set; }
    
    [Required(ErrorMessage = "姓氏是必填项")]
    [StringLength(100, ErrorMessage = "姓氏长度不能超过100个字符")]
    public string LastName { get; set; }
    
    [Range(18, 120, ErrorMessage = "年龄必须在18-120岁之间")]
    public int? Age { get; set; }
    
    [Phone(ErrorMessage = "电话号码格式不正确")]
    public string PhoneNumber { get; set; }
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
        if (userService == null)
            return ValidationResult.Success;
            
        var isEmailExists = userService.IsEmailExistsAsync(email).Result;
        if (isEmailExists)
        {
            return new ValidationResult("该邮箱已被注册");
        }
        
        return ValidationResult.Success;
    }
}
```

#### 自定义验证器
```csharp
public class UserValidator : AbstractValidator<CreateUserRequest>
{
    public UserValidator(IUserService userService)
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("用户名不能为空")
            .Length(3, 50).WithMessage("用户名长度必须在3-50个字符之间")
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("用户名只能包含字母、数字和下划线")
            .MustAsync(async (username, cancellation) =>
            {
                if (string.IsNullOrEmpty(username))
                    return true;
                return !await userService.IsUsernameExistsAsync(username);
            }).WithMessage("用户名已存在");
            
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("邮箱不能为空")
            .EmailAddress().WithMessage("邮箱格式不正确")
            .MustAsync(async (email, cancellation) =>
            {
                if (string.IsNullOrEmpty(email))
                    return true;
                return !await userService.IsEmailExistsAsync(email);
            }).WithMessage("该邮箱已被注册");
            
        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("密码不能为空")
            .MinimumLength(8).WithMessage("密码长度不能少于8个字符")
            .Matches(@"[A-Z]").WithMessage("密码必须包含至少一个大写字母")
            .Matches(@"[a-z]").WithMessage("密码必须包含至少一个小写字母")
            .Matches(@"[0-9]").WithMessage("密码必须包含至少一个数字")
            .Matches(@"[^a-zA-Z0-9]").WithMessage("密码必须包含至少一个特殊字符");
            
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("两次输入的密码不一致");
            
        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("年龄必须在18-120岁之间");
    }
}
```

### 4.2 输入清理

#### HTML 清理
```csharp
public class HtmlSanitizerService : IHtmlSanitizerService
{
    private readonly HtmlSanitizer _sanitizer;
    
    public HtmlSanitizerService()
    {
        _sanitizer = new HtmlSanitizer();
        
        // 允许的标签
        _sanitizer.AllowedTags.Add("div");
        _sanitizer.AllowedTags.Add("span");
        _sanitizer.AllowedTags.Add("p");
        _sanitizer.AllowedTags.Add("br");
        _sanitizer.AllowedTags.Add("strong");
        _sanitizer.AllowedTags.Add("em");
        _sanitizer.AllowedTags.Add("ul");
        _sanitizer.AllowedTags.Add("ol");
        _sanitizer.AllowedTags.Add("li");
        
        // 允许的属性
        _sanitizer.AllowedAttributes.Add("class");
        _sanitizer.AllowedAttributes.Add("id");
        _sanitizer.AllowedAttributes.Add("style");
        
        // 允许的 CSS 属性
        _sanitizer.AllowedCssProperties.Add("color");
        _sanitizer.AllowedCssProperties.Add("background-color");
        _sanitizer.AllowedCssProperties.Add("font-size");
        _sanitizer.AllowedCssProperties.Add("text-align");
    }
    
    public string SanitizeHtml(string html)
    {
        if (string.IsNullOrEmpty(html))
            return html;
            
        return _sanitizer.Sanitize(html);
    }
    
    public string SanitizeText(string text)
    {
        if (string.IsNullOrEmpty(text))
            return text;
            
        // 移除所有 HTML 标签
        return Regex.Replace(text, "<[^>]*>", string.Empty);
    }
}

// 在模型中使用
public class BlogPost
{
    public int Id { get; set; }
    
    [Required]
    [StringLength(200)]
    public string Title { get; set; }
    
    [Required]
    public string Content { get; set; }
    
    public string SanitizedContent
    {
        get => _htmlSanitizer.SanitizeHtml(Content);
    }
    
    private readonly IHtmlSanitizerService _htmlSanitizer;
    
    public BlogPost(IHtmlSanitizerService htmlSanitizer)
    {
        _htmlSanitizer = htmlSanitizer;
    }
}
```

## 5. 审计与日志

### 5.1 审计日志

#### 审计服务
```csharp
public class AuditLog
{
    public int Id { get; set; }
    public string UserId { get; set; }
    public string UserName { get; set; }
    public string Action { get; set; }
    public string EntityName { get; set; }
    public string EntityId { get; set; }
    public string OldValues { get; set; }
    public string NewValues { get; set; }
    public DateTime Timestamp { get; set; }
    public string IpAddress { get; set; }
    public string UserAgent { get; set; }
}

public class AuditService : IAuditService
{
    private readonly DbContext _context;
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public async Task LogAsync(string action, string entityName, string entityId, 
        object oldValues = null, object newValues = null)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        var user = httpContext?.User;
        
        var auditLog = new AuditLog
        {
            UserId = user?.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            UserName = user?.Identity?.Name,
            Action = action,
            EntityName = entityName,
            EntityId = entityId,
            OldValues = oldValues != null ? JsonSerializer.Serialize(oldValues) : null,
            NewValues = newValues != null ? JsonSerializer.Serialize(newValues) : null,
            Timestamp = DateTime.UtcNow,
            IpAddress = GetClientIpAddress(httpContext),
            UserAgent = httpContext?.Request.Headers["User-Agent"].ToString()
        };
        
        _context.AuditLogs.Add(auditLog);
        await _context.SaveChangesAsync();
    }
    
    private string GetClientIpAddress(HttpContext httpContext)
    {
        if (httpContext == null)
            return string.Empty;
            
        var forwardedHeader = httpContext.Request.Headers["X-Forwarded-For"].FirstOrDefault();
        if (!string.IsNullOrEmpty(forwardedHeader))
        {
            return forwardedHeader.Split(',')[0].Trim();
        }
        
        return httpContext.Connection.RemoteIpAddress?.ToString() ?? string.Empty;
    }
}

// 审计特性
public class AuditAttribute : ActionFilterAttribute
{
    private readonly string _action;
    private readonly string _entityName;
    
    public AuditAttribute(string action, string entityName)
    {
        _action = action;
        _entityName = entityName;
    }
    
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        var auditService = context.HttpContext.RequestServices.GetService<IAuditService>();
        if (auditService == null)
            return;
            
        var entityId = GetEntityId(context);
        var oldValues = GetOldValues(context);
        var newValues = GetNewValues(context);
        
        auditService.LogAsync(_action, _entityName, entityId, oldValues, newValues);
    }
    
    private string GetEntityId(ActionExecutedContext context)
    {
        // 从路由或参数中获取实体ID
        if (context.RouteData.Values.TryGetValue("id", out var id))
            return id?.ToString();
            
        return string.Empty;
    }
    
    private object GetOldValues(ActionExecutedContext context)
    {
        // 获取旧值（如更新操作）
        return null;
    }
    
    private object GetNewValues(ActionExecutedContext context)
    {
        // 获取新值
        return context.Result is ObjectResult objectResult ? objectResult.Value : null;
    }
}
```

### 5.2 安全日志

#### 安全事件日志
```csharp
public class SecurityEventLog
{
    public int Id { get; set; }
    public string EventType { get; set; }
    public string UserId { get; set; }
    public string UserName { get; set; }
    public string IpAddress { get; set; }
    public string UserAgent { get; set; }
    public string Details { get; set; }
    public DateTime Timestamp { get; set; }
    public bool IsSuccessful { get; set; }
}

public class SecurityLogger : ISecurityLogger
{
    private readonly ILogger<SecurityLogger> _logger;
    private readonly DbContext _context;
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public async Task LogSecurityEventAsync(string eventType, string details, bool isSuccessful = true)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        var user = httpContext?.User;
        
        var securityEvent = new SecurityEventLog
        {
            EventType = eventType,
            UserId = user?.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            UserName = user?.Identity?.Name,
            IpAddress = GetClientIpAddress(httpContext),
            UserAgent = httpContext?.Request.Headers["User-Agent"].ToString(),
            Details = details,
            Timestamp = DateTime.UtcNow,
            IsSuccessful = isSuccessful
        };
        
        _context.SecurityEventLogs.Add(securityEvent);
        await _context.SaveChangesAsync();
        
        // 同时记录到结构化日志
        _logger.LogInformation("Security event: {EventType} by {UserName} from {IpAddress} - {Details}", 
            eventType, user?.Identity?.Name, GetClientIpAddress(httpContext), details);
    }
    
    public async Task LogLoginAttemptAsync(string username, bool isSuccessful, string reason = null)
    {
        var details = isSuccessful ? "Login successful" : $"Login failed: {reason}";
        await LogSecurityEventAsync("Login", details, isSuccessful);
    }
    
    public async Task LogPasswordChangeAsync(string userId)
    {
        await LogSecurityEventAsync("PasswordChange", "Password changed successfully");
    }
    
    public async Task LogAccessDeniedAsync(string resource, string reason)
    {
        await LogSecurityEventAsync("AccessDenied", $"Access denied to {resource}: {reason}", false);
    }
}
```

## 6. 面试重点

### 6.1 身份认证
- **多因素认证**: TOTP、SMS、硬件令牌
- **OAuth 2.0/OpenID Connect**: 第三方认证、JWT 令牌
- **会话管理**: 会话超时、并发登录控制

### 6.2 数据安全
- **加密算法**: AES、RSA、哈希算法选择
- **密钥管理**: 密钥轮换、密钥存储安全
- **数据脱敏**: PII 数据保护、敏感信息处理

### 6.3 网络安全
- **HTTPS 配置**: 证书管理、TLS 版本选择
- **安全头配置**: CSP、HSTS、XSS 防护
- **CORS 策略**: 跨域安全、预检请求处理

### 6.4 安全最佳实践
- **输入验证**: 白名单验证、参数化查询
- **输出编码**: HTML 编码、JavaScript 编码
- **安全日志**: 审计追踪、安全事件监控
- **漏洞防护**: OWASP Top 10、安全编码规范
