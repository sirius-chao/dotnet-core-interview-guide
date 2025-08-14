# 安全与最佳实践

## 1. OWASP 安全指南

### 1.1 注入攻击防护
```csharp
// SQL注入防护
public class SecureUserRepository : IUserRepository
{
    private readonly IDbConnection _connection;
    private readonly ILogger<SecureUserRepository> _logger;
    
    public SecureUserRepository(IDbConnection connection, ILogger<SecureUserRepository> logger)
    {
        _connection = connection;
        _logger = logger;
    }
    
    // 错误示例：容易受到SQL注入攻击
    public async Task<User> GetUserByEmailAsync_Unsafe(string email)
    {
        var sql = $"SELECT * FROM Users WHERE Email = '{email}'";
        return await _connection.QueryFirstOrDefaultAsync<User>(sql);
    }
    
    // 正确示例：使用参数化查询
    public async Task<User> GetUserByEmailAsync_Safe(string email)
    {
        var sql = "SELECT * FROM Users WHERE Email = @Email";
        var parameters = new { Email = email };
        return await _connection.QueryFirstOrDefaultAsync<User>(sql, parameters);
    }
    
    // 使用存储过程
    public async Task<User> GetUserByEmailAsync_StoredProcedure(string email)
    {
        var sql = "EXEC GetUserByEmail @Email";
        var parameters = new { Email = email };
        return await _connection.QueryFirstOrDefaultAsync<User>(sql, parameters);
    }
    
    // 输入验证
    public async Task<User> GetUserByEmailAsync_Validated(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            throw new ArgumentException("Email cannot be empty", nameof(email));
        
        if (!IsValidEmail(email))
            throw new ArgumentException("Invalid email format", nameof(email));
        
        var sql = "SELECT * FROM Users WHERE Email = @Email";
        var parameters = new { Email = email };
        return await _connection.QueryFirstOrDefaultAsync<User>(sql, parameters);
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

// XSS防护
public class XssProtectionService
{
    // HTML编码
    public static string EncodeHtml(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        return HttpUtility.HtmlEncode(input);
    }
    
    // JavaScript编码
    public static string EncodeJavaScript(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        return HttpUtility.JavaScriptStringEncode(input);
    }
    
    // URL编码
    public static string EncodeUrl(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        return HttpUtility.UrlEncode(input);
    }
    
    // 内容安全策略
    public static string GetCspHeader()
    {
        return "default-src 'self'; " +
               "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
               "style-src 'self' 'unsafe-inline'; " +
               "img-src 'self' data: https:; " +
               "font-src 'self' https:; " +
               "connect-src 'self' https:; " +
               "frame-ancestors 'none';";
    }
}

// 命令注入防护
public class SecureCommandExecutor
{
    private readonly ILogger<SecureCommandExecutor> _logger;
    
    public SecureCommandExecutor(ILogger<SecureCommandExecutor> logger)
    {
        _logger = logger;
    }
    
    // 错误示例：容易受到命令注入攻击
    public string ExecuteCommand_Unsafe(string command)
    {
        var process = new Process
        {
            StartInfo = new ProcessStartInfo
            {
                FileName = "cmd.exe",
                Arguments = $"/c {command}",
                RedirectStandardOutput = true,
                UseShellExecute = false,
                CreateNoWindow = true
            }
        };
        
        process.Start();
        var output = process.StandardOutput.ReadToEnd();
        process.WaitForExit();
        
        return output;
    }
    
    // 正确示例：白名单验证
    public string ExecuteCommand_Safe(string command)
    {
        var allowedCommands = new[] { "dir", "echo", "type" };
        
        if (!allowedCommands.Contains(command.ToLower()))
        {
            _logger.LogWarning("Blocked potentially dangerous command: {Command}", command);
            throw new SecurityException($"Command '{command}' is not allowed");
        }
        
        var process = new Process
        {
            StartInfo = new ProcessStartInfo
            {
                FileName = "cmd.exe",
                Arguments = $"/c {command}",
                RedirectStandardOutput = true,
                UseShellExecute = false,
                CreateNoWindow = true
            }
        };
        
        process.Start();
        var output = process.StandardOutput.ReadToEnd();
        process.WaitForExit();
        
        return output;
    }
}
```

### 1.2 认证和授权
```csharp
// JWT认证服务
public class JwtAuthenticationService
{
    private readonly IConfiguration _configuration;
    private readonly IUserService _userService;
    private readonly ILogger<JwtAuthenticationService> _logger;
    
    public JwtAuthenticationService(
        IConfiguration configuration,
        IUserService userService,
        ILogger<JwtAuthenticationService> logger)
    {
        _configuration = configuration;
        _userService = userService;
        _logger = logger;
    }
    
    public async Task<string> GenerateTokenAsync(User user)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Email),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim("UserId", user.Id.ToString())
        };
        
        // 添加角色声明
        if (user.Roles != null)
        {
            foreach (var role in user.Roles)
            {
                claims.Add(new Claim(ClaimTypes.Role, role.Name));
            }
        }
        
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public ClaimsPrincipal ValidateToken(string token)
    {
        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var key = Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]);
            
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _configuration["Jwt:Issuer"],
                ValidateAudience = true,
                ValidAudience = _configuration["Jwt:Audience"],
                ValidateLifetime = true,
                ClockSkew = TimeSpan.Zero
            };
            
            var principal = tokenHandler.ValidateToken(token, validationParameters, out var validatedToken);
            return principal;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Token validation failed");
            return null;
        }
    }
}

// 基于角色的授权
public class RoleBasedAuthorizationHandler : AuthorizationHandler<RoleRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        RoleRequirement requirement)
    {
        if (context.User == null)
            return Task.CompletedTask;
        
        var userRoles = context.User.Claims
            .Where(c => c.Type == ClaimTypes.Role)
            .Select(c => c.Value);
        
        if (userRoles.Any(role => requirement.Roles.Contains(role)))
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

public class RoleRequirement : IAuthorizationRequirement
{
    public string[] Roles { get; }
    
    public RoleRequirement(params string[] roles)
    {
        Roles = roles;
    }
}

// 基于策略的授权
public class PolicyBasedAuthorizationHandler : AuthorizationHandler<PolicyRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PolicyRequirement requirement)
    {
        if (context.User == null)
            return Task.CompletedTask;
        
        var userId = context.User.FindFirst("UserId")?.Value;
        if (string.IsNullOrEmpty(userId))
            return Task.CompletedTask;
        
        // 检查用户是否有特定权限
        if (HasPermission(context.User, requirement.Permission))
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
    
    private bool HasPermission(ClaimsPrincipal user, string permission)
    {
        // 实现权限检查逻辑
        return user.HasClaim("Permission", permission);
    }
}

public class PolicyRequirement : IAuthorizationRequirement
{
    public string Permission { get; }
    
    public PolicyRequirement(string permission)
    {
        Permission = permission;
    }
}

// 授权特性
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class RequirePermissionAttribute : Attribute, IAuthorizationFilter
{
    private readonly string _permission;
    
    public RequirePermissionAttribute(string permission)
    {
        _permission = permission;
    }
    
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var user = context.HttpContext.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }
        
        if (!user.HasClaim("Permission", _permission))
        {
            context.Result = new ForbidResult();
        }
    }
}
```

### 1.3 数据验证和清理
```csharp
// 输入验证服务
public class InputValidationService
{
    // 字符串验证
    public static ValidationResult ValidateString(string input, string fieldName, int maxLength = 100)
    {
        var result = new ValidationResult();
        
        if (string.IsNullOrWhiteSpace(input))
        {
            result.AddError(fieldName, $"{fieldName} is required");
            return result;
        }
        
        if (input.Length > maxLength)
        {
            result.AddError(fieldName, $"{fieldName} cannot exceed {maxLength} characters");
        }
        
        // 检查特殊字符
        if (ContainsDangerousCharacters(input))
        {
            result.AddError(fieldName, $"{fieldName} contains invalid characters");
        }
        
        return result;
    }
    
    // 数字验证
    public static ValidationResult ValidateNumber(int value, string fieldName, int minValue = int.MinValue, int maxValue = int.MaxValue)
    {
        var result = new ValidationResult();
        
        if (value < minValue || value > maxValue)
        {
            result.AddError(fieldName, $"{fieldName} must be between {minValue} and {maxValue}");
        }
        
        return result;
    }
    
    // 邮箱验证
    public static ValidationResult ValidateEmail(string email)
    {
        var result = new ValidationResult();
        
        if (string.IsNullOrWhiteSpace(email))
        {
            result.AddError("Email", "Email is required");
            return result;
        }
        
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            if (addr.Address != email)
            {
                result.AddError("Email", "Invalid email format");
            }
        }
        catch
        {
            result.AddError("Email", "Invalid email format");
        }
        
        return result;
    }
    
    // 文件上传验证
    public static ValidationResult ValidateFileUpload(IFormFile file, string[] allowedExtensions, long maxSizeInBytes)
    {
        var result = new ValidationResult();
        
        if (file == null || file.Length == 0)
        {
            result.AddError("File", "File is required");
            return result;
        }
        
        if (file.Length > maxSizeInBytes)
        {
            result.AddError("File", $"File size cannot exceed {maxSizeInBytes / 1024 / 1024} MB");
        }
        
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!allowedExtensions.Contains(extension))
        {
            result.AddError("File", $"File type {extension} is not allowed");
        }
        
        return result;
    }
    
    private static bool ContainsDangerousCharacters(string input)
    {
        var dangerousPatterns = new[]
        {
            "<script>", "javascript:", "vbscript:", "onload=", "onerror="
        };
        
        return dangerousPatterns.Any(pattern => 
            input.ToLowerInvariant().Contains(pattern));
    }
}

public class ValidationResult
{
    private readonly List<ValidationError> _errors = new List<ValidationError>();
    
    public bool IsValid => !_errors.Any();
    public IReadOnlyList<ValidationError> Errors => _errors.AsReadOnly();
    
    public void AddError(string field, string message)
    {
        _errors.Add(new ValidationError(field, message));
    }
    
    public void AddErrors(IEnumerable<ValidationError> errors)
    {
        _errors.AddRange(errors);
    }
}

public class ValidationError
{
    public string Field { get; }
    public string Message { get; }
    
    public ValidationError(string field, string message)
    {
        Field = field;
        Message = message;
    }
}

// 数据清理服务
public class DataSanitizationService
{
    // HTML清理
    public static string SanitizeHtml(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        // 使用HtmlSanitizer库
        var sanitizer = new HtmlSanitizer();
        sanitizer.AllowedTags.Add("div");
        sanitizer.AllowedTags.Add("span");
        sanitizer.AllowedTags.Add("p");
        sanitizer.AllowedTags.Add("br");
        sanitizer.AllowedTags.Add("strong");
        sanitizer.AllowedTags.Add("em");
        sanitizer.AllowedTags.Add("ul");
        sanitizer.AllowedTags.Add("ol");
        sanitizer.AllowedTags.Add("li");
        
        return sanitizer.Sanitize(input);
    }
    
    // SQL注入防护
    public static string SanitizeSql(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        // 移除危险字符
        var dangerousChars = new[] { "'", "\"", ";", "--", "/*", "*/", "xp_", "sp_" };
        var sanitized = input;
        
        foreach (var dangerousChar in dangerousChars)
        {
            sanitized = sanitized.Replace(dangerousChar, "");
        }
        
        return sanitized;
    }
    
    // XSS防护
    public static string SanitizeXss(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        // 编码HTML特殊字符
        return HttpUtility.HtmlEncode(input);
    }
}
```

## 2. 数据加密

### 2.1 对称加密
```csharp
public class SymmetricEncryptionService
{
    private readonly byte[] _key;
    private readonly byte[] _iv;
    
    public SymmetricEncryptionService(IConfiguration configuration)
    {
        var keyString = configuration["Encryption:Key"];
        var ivString = configuration["Encryption:IV"];
        
        if (string.IsNullOrEmpty(keyString) || string.IsNullOrEmpty(ivString))
            throw new InvalidOperationException("Encryption key and IV must be configured");
        
        _key = Convert.FromBase64String(keyString);
        _iv = Convert.FromBase64String(ivString);
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
        
        try
        {
            using var aes = Aes.Create();
            aes.Key = _key;
            aes.IV = _iv;
            
            using var decryptor = aes.CreateDecryptor();
            using var msDecrypt = new MemoryStream(Convert.FromBase64String(cipherText));
            using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
            using var srDecrypt = new StreamReader(csDecrypt);
            
            return srDecrypt.ReadToEnd();
        }
        catch (Exception ex)
        {
            throw new CryptographicException("Failed to decrypt data", ex);
        }
    }
}
```

### 2.2 非对称加密
```csharp
public class AsymmetricEncryptionService
{
    private readonly RSA _rsa;
    
    public AsymmetricEncryptionService(IConfiguration configuration)
    {
        var privateKeyPem = configuration["Encryption:PrivateKey"];
        var publicKeyPem = configuration["Encryption:PublicKey"];
        
        if (string.IsNullOrEmpty(privateKeyPem) || string.IsNullOrEmpty(publicKeyPem))
            throw new InvalidOperationException("RSA keys must be configured");
        
        _rsa = RSA.Create();
        _rsa.ImportFromPem(privateKeyPem);
    }
    
    public string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText))
            return plainText;
        
        var data = Encoding.UTF8.GetBytes(plainText);
        var encryptedData = _rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);
        
        return Convert.ToBase64String(encryptedData);
    }
    
    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText))
            return cipherText;
        
        try
        {
            var encryptedData = Convert.FromBase64String(cipherText);
            var decryptedData = _rsa.Decrypt(encryptedData, RSAEncryptionPadding.OaepSHA256);
            
            return Encoding.UTF8.GetString(decryptedData);
        }
        catch (Exception ex)
        {
            throw new CryptographicException("Failed to decrypt data", ex);
        }
    }
    
    public string Sign(string data)
    {
        if (string.IsNullOrEmpty(data))
            return data;
        
        var dataBytes = Encoding.UTF8.GetBytes(data);
        var signature = _rsa.SignData(dataBytes, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
        
        return Convert.ToBase64String(signature);
    }
    
    public bool Verify(string data, string signature)
    {
        if (string.IsNullOrEmpty(data) || string.IsNullOrEmpty(signature))
            return false;
        
        try
        {
            var dataBytes = Encoding.UTF8.GetBytes(data);
            var signatureBytes = Convert.FromBase64String(signature);
            
            return _rsa.VerifyData(dataBytes, signatureBytes, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
        }
        catch
        {
            return false;
        }
    }
}
```

### 2.3 哈希和盐值
```csharp
public class PasswordHashingService
{
    private readonly int _iterations;
    private readonly int _keySize;
    
    public PasswordHashingService(IConfiguration configuration)
    {
        _iterations = int.Parse(configuration["Password:Iterations"] ?? "10000");
        _keySize = int.Parse(configuration["Password:KeySize"] ?? "32");
    }
    
    public string HashPassword(string password)
    {
        if (string.IsNullOrEmpty(password))
            throw new ArgumentException("Password cannot be empty", nameof(password));
        
        // 生成随机盐值
        var salt = new byte[16];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(salt);
        }
        
        // 使用PBKDF2进行哈希
        using var pbkdf2 = new Rfc2898DeriveBytes(password, salt, _iterations, HashAlgorithmName.SHA256);
        var hash = pbkdf2.GetBytes(_keySize);
        
        // 组合盐值和哈希
        var hashBytes = new byte[32];
        Array.Copy(salt, 0, hashBytes, 0, 16);
        Array.Copy(hash, 0, hashBytes, 16, 16);
        
        return Convert.ToBase64String(hashBytes);
    }
    
    public bool VerifyPassword(string password, string hashedPassword)
    {
        if (string.IsNullOrEmpty(password) || string.IsNullOrEmpty(hashedPassword))
            return false;
        
        try
        {
            var hashBytes = Convert.FromBase64String(hashedPassword);
            
            // 提取盐值
            var salt = new byte[16];
            Array.Copy(hashBytes, 0, salt, 0, 16);
            
            // 提取哈希
            var hash = new byte[16];
            Array.Copy(hashBytes, 16, hash, 0, 16);
            
            // 验证密码
            using var pbkdf2 = new Rfc2898DeriveBytes(password, salt, _iterations, HashAlgorithmName.SHA256);
            var testHash = pbkdf2.GetBytes(_keySize);
            
            return hash.SequenceEqual(testHash);
        }
        catch
        {
            return false;
        }
    }
}
```

## 3. 代码审查

### 3.1 代码审查清单
```csharp
// 代码审查服务
public class CodeReviewService
{
    private readonly ILogger<CodeReviewService> _logger;
    
    public CodeReviewService(ILogger<CodeReviewService> logger)
    {
        _logger = logger;
    }
    
    public CodeReviewResult ReviewCode(string code, string language)
    {
        var result = new CodeReviewResult();
        
        // 安全性检查
        CheckSecurityIssues(code, language, result);
        
        // 性能检查
        CheckPerformanceIssues(code, language, result);
        
        // 可维护性检查
        CheckMaintainabilityIssues(code, language, result);
        
        // 代码质量检查
        CheckCodeQualityIssues(code, language, result);
        
        return result;
    }
    
    private void CheckSecurityIssues(string code, string language, CodeReviewResult result)
    {
        // 检查SQL注入
        if (ContainsSqlInjection(code))
        {
            result.AddIssue(CodeReviewIssueType.Security, "Potential SQL injection vulnerability");
        }
        
        // 检查XSS
        if (ContainsXssVulnerability(code))
        {
            result.AddIssue(CodeReviewIssueType.Security, "Potential XSS vulnerability");
        }
        
        // 检查硬编码凭据
        if (ContainsHardcodedCredentials(code))
        {
            result.AddIssue(CodeReviewIssueType.Security, "Hardcoded credentials detected");
        }
        
        // 检查不安全的反序列化
        if (ContainsUnsafeDeserialization(code))
        {
            result.AddIssue(CodeReviewIssueType.Security, "Unsafe deserialization detected");
        }
    }
    
    private void CheckPerformanceIssues(string code, string language, CodeReviewResult result)
    {
        // 检查N+1查询
        if (ContainsNPlusOneQuery(code))
        {
            result.AddIssue(CodeReviewIssueType.Performance, "Potential N+1 query issue");
        }
        
        // 检查内存泄漏
        if (ContainsMemoryLeak(code))
        {
            result.AddIssue(CodeReviewIssueType.Performance, "Potential memory leak");
        }
        
        // 检查同步等待
        if (ContainsSyncWait(code))
        {
            result.AddIssue(CodeReviewIssueType.Performance, "Synchronous wait in async method");
        }
    }
    
    private void CheckMaintainabilityIssues(string code, string language, CodeReviewResult result)
    {
        // 检查代码重复
        if (ContainsCodeDuplication(code))
        {
            result.AddIssue(CodeReviewIssueType.Maintainability, "Code duplication detected");
        }
        
        // 检查长方法
        if (ContainsLongMethods(code))
        {
            result.AddIssue(CodeReviewIssueType.Maintainability, "Long methods detected");
        }
        
        // 检查复杂条件
        if (ContainsComplexConditions(code))
        {
            result.AddIssue(CodeReviewIssueType.Maintainability, "Complex conditions detected");
        }
    }
    
    private void CheckCodeQualityIssues(string code, string language, CodeReviewResult result)
    {
        // 检查命名约定
        if (!FollowsNamingConventions(code, language))
        {
            result.AddIssue(CodeReviewIssueType.CodeQuality, "Naming conventions not followed");
        }
        
        // 检查注释质量
        if (!HasQualityComments(code))
        {
            result.AddIssue(CodeReviewIssueType.CodeQuality, "Insufficient or poor quality comments");
        }
        
        // 检查异常处理
        if (!HasProperExceptionHandling(code))
        {
            result.AddIssue(CodeReviewIssueType.CodeQuality, "Improper exception handling");
        }
    }
    
    // 具体的检查方法实现
    private bool ContainsSqlInjection(string code)
    {
        var patterns = new[]
        {
            @"string\.Format.*SELECT",
            @"\$""SELECT.*\{.*\}",
            @"\+.*SELECT.*\+"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsXssVulnerability(string code)
    {
        var patterns = new[]
        {
            @"Response\.Write.*Request",
            @"ViewBag\..*=.*Request",
            @"ViewData\..*=.*Request"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsHardcodedCredentials(string code)
    {
        var patterns = new[]
        {
            @"password.*=.*""[^""]+""",
            @"connectionString.*=.*""[^""]+""",
            @"apiKey.*=.*""[^""]+"""
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsUnsafeDeserialization(string code)
    {
        var patterns = new[]
        {
            @"JavaScriptSerializer",
            @"BinaryFormatter",
            @"XmlSerializer.*ReadObject"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsNPlusOneQuery(string code)
    {
        var patterns = new[]
        {
            @"foreach.*\.Where",
            @"foreach.*\.FirstOrDefault",
            @"foreach.*\.SingleOrDefault"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsMemoryLeak(string code)
    {
        var patterns = new[]
        {
            @"new Timer.*Start",
            @"new EventHandler",
            @"\.Subscribe"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsSyncWait(string code)
    {
        var patterns = new[]
        {
            @"async.*Task.*\.Result",
            @"async.*Task.*\.Wait",
            @"async.*Task.*\.GetAwaiter"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern, RegexOptions.IgnoreCase));
    }
    
    private bool ContainsCodeDuplication(string code)
    {
        // 简化的重复代码检测
        var lines = code.Split('\n');
        var lineGroups = lines.GroupBy(line => line.Trim())
            .Where(g => g.Count() > 3 && g.Key.Length > 20);
        
        return lineGroups.Any();
    }
    
    private bool ContainsLongMethods(string code)
    {
        var lines = code.Split('\n');
        var methodLines = 0;
        
        foreach (var line in lines)
        {
            if (line.Contains("public") || line.Contains("private") || line.Contains("protected"))
            {
                if (methodLines > 50) return true;
                methodLines = 0;
            }
            methodLines++;
        }
        
        return methodLines > 50;
    }
    
    private bool ContainsComplexConditions(string code)
    {
        var patterns = new[]
        {
            @".*&&.*&&.*&&",
            @".*\|\|.*\|\|.*\|\|",
            @".*\(.*\(.*\(.*\).*\).*\)"
        };
        
        return patterns.Any(pattern => Regex.IsMatch(code, pattern));
    }
    
    private bool FollowsNamingConventions(string code, string language)
    {
        if (language.ToLower() == "csharp")
        {
            // 检查C#命名约定
            var patterns = new[]
            {
                @"public.*[A-Z][a-zA-Z]*\s*\(",
                @"private.*[a-z][a-zA-Z]*\s*\(",
                @"class\s+[A-Z][a-zA-Z]*"
            };
            
            return patterns.All(pattern => Regex.IsMatch(code, pattern));
        }
        
        return true;
    }
    
    private bool HasQualityComments(string code)
    {
        var commentLines = code.Split('\n')
            .Count(line => line.Trim().StartsWith("//") || line.Trim().StartsWith("/*"));
        
        var totalLines = code.Split('\n').Length;
        var commentRatio = (double)commentLines / totalLines;
        
        return commentRatio > 0.1 && commentRatio < 0.5;
    }
    
    private bool HasProperExceptionHandling(string code)
    {
        var patterns = new[]
        {
            @"try\s*\{",
            @"catch\s*\{",
            @"finally\s*\{"
        };
        
        return patterns.All(pattern => Regex.IsMatch(code, pattern));
    }
}

public class CodeReviewResult
{
    private readonly List<CodeReviewIssue> _issues = new List<CodeReviewIssue>();
    
    public bool HasIssues => _issues.Any();
    public IReadOnlyList<CodeReviewIssue> Issues => _issues.AsReadOnly();
    
    public void AddIssue(CodeReviewIssueType type, string description)
    {
        _issues.Add(new CodeReviewIssue(type, description));
    }
    
    public IEnumerable<CodeReviewIssue> GetIssuesByType(CodeReviewIssueType type)
    {
        return _issues.Where(issue => issue.Type == type);
    }
}

public class CodeReviewIssue
{
    public CodeReviewIssueType Type { get; }
    public string Description { get; }
    public DateTime Timestamp { get; }
    
    public CodeReviewIssue(CodeReviewIssueType type, string description)
    {
        Type = type;
        Description = description;
        Timestamp = DateTime.UtcNow;
    }
}

public enum CodeReviewIssueType
{
    Security,
    Performance,
    Maintainability,
    CodeQuality
}
```

## 4. 面试重点

### 4.1 高频问题
1. **OWASP Top 10**：注入攻击、XSS、认证授权、数据泄露
2. **数据加密**：对称加密、非对称加密、哈希算法、盐值
3. **代码审查**：安全漏洞、性能问题、代码质量
4. **安全最佳实践**：输入验证、输出编码、最小权限原则
5. **合规性**：GDPR、SOX、PCI DSS等法规要求

### 4.2 代码示例准备
- 安全编码实践
- 加密解密实现
- 输入验证和清理
- 代码审查工具
- 安全测试方法

### 4.3 安全要点
- 深度防御策略
- 最小权限原则
- 安全开发生命周期
- 威胁建模
- 安全测试和渗透测试
