# Entity Framework Core

## 1. Code First 开发

### 1.1 实体模型设计

**Code First开发模式的核心思想**
Code First是Entity Framework的一种开发模式，它允许开发者通过编写C#代码来定义数据模型，然后让EF自动生成数据库结构。这种方式实现了"代码即模型"的理念。

**Code First的优势**：
1. **版本控制友好**：实体模型作为代码文件，可以纳入版本控制系统
2. **快速迭代**：修改模型后可以快速更新数据库结构
3. **类型安全**：编译时检查，避免运行时类型错误
4. **测试友好**：可以使用内存数据库进行单元测试
5. **跨平台**：不依赖特定的数据库管理工具

**实体设计原则**：
- **单一职责**：每个实体类代表一个业务概念
- **关系清晰**：明确定义实体间的关系（一对一、一对多、多对多）
- **数据完整性**：使用约束确保数据的有效性
- **性能考虑**：合理设计索引，避免N+1查询问题
- **扩展性**：预留扩展字段，支持未来功能需求

**实体属性的设计策略**：
- **主键设计**：使用自增整数或GUID作为主键
- **审计字段**：包含创建时间、修改时间、创建人等审计信息
- **软删除**：使用IsDeleted标志而不是物理删除
- **并发控制**：使用RowVersion或时间戳防止并发冲突
- **计算属性**：使用表达式属性提高性能

**导航属性的设计考虑**：
- **延迟加载**：使用virtual关键字支持延迟加载
- **预加载**：通过Include方法预加载相关数据
- **显式加载**：使用Load方法按需加载数据
- **关系配置**：明确配置级联删除、外键约束等
```csharp
public class User
{
    public int Id { get; set; }
    
    [Required]
    [StringLength(100)]
    public string FirstName { get; set; }
    
    [Required]
    [StringLength(100)]
    public string LastName { get; set; }
    
    [Required]
    [EmailAddress]
    [Index(nameof(Email), IsUnique = true)]
    public string Email { get; set; }
    
    public DateTime CreatedAt { get; set; }
    
    public DateTime? LastLoginAt { get; set; }
    
    // 导航属性
    public virtual ICollection<Order> Orders { get; set; }
    public virtual UserProfile Profile { get; set; }
    
    // 计算属性
    public string FullName => $"{FirstName} {LastName}";
    
    // 阴影属性
    public string CreatedBy { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public int UserId { get; set; }
    
    [StringLength(500)]
    public string Bio { get; set; }
    
    public string AvatarUrl { get; set; }
    
    // 外键关系
    public virtual User User { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime OrderDate { get; set; }
    
    // 导航属性
    public virtual User User { get; set; }
    public virtual ICollection<OrderItem> Items { get; set; }
    
    // 并发控制
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

### 1.2 DbContext 配置

**DbContext的核心作用**
DbContext是Entity Framework的核心类，它代表与数据库的会话，负责管理实体对象、跟踪变更、执行查询和保存数据。它是连接代码模型和数据库的桥梁。

**DbContext的生命周期管理**：
1. **创建阶段**：通过依赖注入或手动创建DbContext实例
2. **配置阶段**：在OnModelCreating方法中配置实体映射
3. **使用阶段**：执行查询、添加、修改、删除操作
4. **变更跟踪**：自动跟踪实体的状态变化
5. **保存阶段**：调用SaveChanges将变更保存到数据库
6. **释放阶段**：使用using语句或依赖注入容器管理生命周期

**配置策略的选择**：
- **Fluent API配置**：在OnModelCreating方法中使用ModelBuilder进行配置
- **数据注解配置**：使用特性（Attributes）直接在实体类上配置
- **约定配置**：依赖EF的默认约定，减少配置代码
- **混合配置**：结合多种配置方式，灵活处理复杂场景

**性能优化配置**：
- **查询优化**：配置适当的索引，优化查询性能
- **关系配置**：合理设置级联删除和延迟加载
- **批量操作**：配置批量插入、更新、删除的阈值
- **连接池**：合理配置数据库连接池大小
- **查询缓存**：启用查询结果缓存，减少重复查询

**安全配置考虑**：
- **参数化查询**：使用参数化查询防止SQL注入
- **权限控制**：配置适当的数据库用户权限
- **敏感数据**：对敏感字段进行加密或脱敏处理
- **审计日志**：记录数据变更的审计信息
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
    
    public DbSet<User> Users { get; set; }
    public DbSet<UserProfile> UserProfiles { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // 配置实体
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(100);
            entity.HasIndex(e => e.Email).IsUnique();
            
            // 配置关系
            entity.HasOne(e => e.Profile)
                  .WithOne(e => e.User)
                  .HasForeignKey<UserProfile>(e => e.UserId)
                  .OnDelete(DeleteBehavior.Cascade);
            
            entity.HasMany(e => e.Orders)
                  .WithOne(e => e.User)
                  .HasForeignKey(e => e.UserId)
                  .OnDelete(DeleteBehavior.Restrict);
        });
        
        // 配置值对象
        modelBuilder.Entity<Order>(entity =>
        {
            entity.Property(e => e.TotalAmount)
                  .HasColumnType("decimal(18,2)")
                  .HasPrecision(18, 2);
            
            entity.Property(e => e.Status)
                  .HasConversion<string>()
                  .HasMaxLength(20);
        });
        
        // 全局查询过滤器
        modelBuilder.Entity<User>()
            .HasQueryFilter(e => !e.IsDeleted);
    }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseSqlServer("DefaultConnection");
        }
        
        // 启用敏感数据日志（仅开发环境）
        if (Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Development")
        {
            optionsBuilder.EnableSensitiveDataLogging();
        }
    }
}
```

### 1.3 种子数据
```csharp
public static class ModelBuilderExtensions
{
    public static void Seed(this ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>().HasData(
            new User
            {
                Id = 1,
                FirstName = "Admin",
                LastName = "User",
                Email = "admin@example.com",
                CreatedAt = DateTime.UtcNow
            },
            new User
            {
                Id = 2,
                FirstName = "John",
                LastName = "Doe",
                Email = "john.doe@example.com",
                CreatedAt = DateTime.UtcNow
            }
        );
        
        modelBuilder.Entity<UserProfile>().HasData(
            new UserProfile
            {
                Id = 1,
                UserId = 1,
                Bio = "System Administrator"
            }
        );
    }
}
```

## 2. 迁移管理

### 2.1 创建和应用迁移
```bash
# 创建迁移
dotnet ef migrations add InitialCreate

# 应用迁移到数据库
dotnet ef database update

# 回滚到指定迁移
dotnet ef database update PreviousMigrationName

# 生成SQL脚本
dotnet ef migrations script

# 从特定迁移生成脚本
dotnet ef migrations script FromMigrationName ToMigrationName
```

### 2.2 迁移文件示例
```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Users",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                FirstName = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                LastName = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                Email = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                CreatedAt = table.Column<DateTime>(type: "datetime2", nullable: false),
                LastLoginAt = table.Column<DateTime>(type: "datetime2", nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Users", x => x.Id);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Users_Email",
            table: "Users",
            column: "Email",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(
            name: "Users");
    }
}
```

### 2.3 数据迁移
```csharp
public partial class AddUserStatus : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // 添加新列
        migrationBuilder.AddColumn<string>(
            name: "Status",
            table: "Users",
            type: "nvarchar(20)",
            maxLength: 20,
            nullable: false,
            defaultValue: "Active");
        
        // 更新现有数据
        migrationBuilder.Sql("UPDATE Users SET Status = 'Active' WHERE Status IS NULL");
        
        // 设置非空约束
        migrationBuilder.AlterColumn<string>(
            name: "Status",
            table: "Users",
            type: "nvarchar(20)",
            maxLength: 20,
            nullable: false);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "Status",
            table: "Users");
    }
}
```

## 3. 查询优化

### 3.1 基本查询
```csharp
public class UserService
{
    private readonly ApplicationDbContext _context;
    
    public UserService(ApplicationDbContext context)
    {
        _context = context;
    }
    
    // 基本查询
    public async Task<User> GetUserByIdAsync(int id)
    {
        return await _context.Users
            .FirstOrDefaultAsync(u => u.Id == id);
    }
    
    // 包含导航属性
    public async Task<User> GetUserWithProfileAsync(int id)
    {
        return await _context.Users
            .Include(u => u.Profile)
            .Include(u => u.Orders)
            .FirstOrDefaultAsync(u => u.Id == id);
    }
    
    // 投影查询
    public async Task<IEnumerable<UserDto>> GetUserListAsync()
    {
        return await _context.Users
            .Select(u => new UserDto
            {
                Id = u.Id,
                FullName = u.FullName,
                Email = u.Email,
                OrderCount = u.Orders.Count
            })
            .ToListAsync();
    }
    
    // 分页查询
    public async Task<PaginatedResult<UserDto>> GetUsersPaginatedAsync(
        int pageNumber, int pageSize, string searchTerm = null)
    {
        var query = _context.Users.AsQueryable();
        
        if (!string.IsNullOrEmpty(searchTerm))
        {
            query = query.Where(u => 
                u.FirstName.Contains(searchTerm) || 
                u.LastName.Contains(searchTerm) || 
                u.Email.Contains(searchTerm));
        }
        
        var totalCount = await query.CountAsync();
        var users = await query
            .Select(u => new UserDto
            {
                Id = u.Id,
                FullName = u.FullName,
                Email = u.Email
            })
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        return new PaginatedResult<UserDto>
        {
            Data = users,
            PageNumber = pageNumber,
            PageSize = pageSize,
            TotalCount = totalCount
        };
    }
}
```

### 3.2 复杂查询
```csharp
// 动态查询构建
public async Task<IEnumerable<User>> GetUsersByCriteriaAsync(UserSearchCriteria criteria)
{
    var query = _context.Users.AsQueryable();
    
    if (criteria.MinAge.HasValue)
    {
        var minDate = DateTime.Today.AddYears(-criteria.MinAge.Value);
        query = query.Where(u => u.DateOfBirth <= minDate);
    }
    
    if (criteria.MaxAge.HasValue)
    {
        var maxDate = DateTime.Today.AddYears(-criteria.MaxAge.Value - 1);
        query = query.Where(u => u.DateOfBirth > maxDate);
    }
    
    if (!string.IsNullOrEmpty(criteria.City))
    {
        query = query.Where(u => u.Profile.City == criteria.City);
    }
    
    if (criteria.HasOrders)
    {
        query = query.Where(u => u.Orders.Any());
    }
    
    if (criteria.MinOrderAmount.HasValue)
    {
        query = query.Where(u => u.Orders.Sum(o => o.TotalAmount) >= criteria.MinOrderAmount.Value);
    }
    
    return await query
        .Include(u => u.Profile)
        .Include(u => u.Orders)
        .ToListAsync();
}

// 原始SQL查询
public async Task<IEnumerable<User>> GetUsersByRawSqlAsync(string city)
{
    return await _context.Users
        .FromSqlRaw("SELECT * FROM Users u JOIN UserProfiles p ON u.Id = p.UserId WHERE p.City = {0}", city)
        .Include(u => u.Profile)
        .ToListAsync();
}

// 存储过程调用
public async Task<IEnumerable<User>> GetUsersByStoredProcedureAsync(string city)
{
    return await _context.Users
        .FromSqlRaw("EXEC GetUsersByCity @City = {0}", city)
        .ToListAsync();
}
```

### 3.3 查询性能优化
```csharp
// 使用AsNoTracking提高查询性能
public async Task<IEnumerable<UserDto>> GetUsersForDisplayAsync()
{
    return await _context.Users
        .AsNoTracking()
        .Select(u => new UserDto
        {
            Id = u.Id,
            FullName = u.FullName,
            Email = u.Email
        })
        .ToListAsync();
}

// 批量查询避免N+1问题
public async Task<IEnumerable<UserWithOrders>> GetUsersWithOrdersAsync()
{
    var users = await _context.Users
        .Include(u => u.Orders)
        .ToListAsync();
    
    // 预加载所有相关数据
    var orderIds = users.SelectMany(u => u.Orders).Select(o => o.Id).ToList();
    var orderItems = await _context.OrderItems
        .Where(oi => orderIds.Contains(oi.OrderId))
        .ToListAsync();
    
    // 手动组装数据
    return users.Select(u => new UserWithOrders
    {
        User = u,
        Orders = u.Orders.Select(o => new OrderWithItems
        {
            Order = o,
            Items = orderItems.Where(oi => oi.OrderId == o.Id).ToList()
        }).ToList()
    });
}

// 使用CompiledQuery提高重复查询性能
public static class CompiledQueries
{
    public static readonly Func<ApplicationDbContext, int, Task<User>> GetUserById =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
    
    public static readonly Func<ApplicationDbContext, string, Task<IEnumerable<User>>> GetUsersByCity =
        EF.CompileAsyncQuery((ApplicationDbContext context, string city) =>
            context.Users
                .Where(u => u.Profile.City == city)
                .Select(u => new User { Id = u.Id, FirstName = u.FirstName, LastName = u.LastName })
                .ToList());
}
```

## 4. 性能优化

### 4.1 批量操作
```csharp
// 批量插入
public async Task BulkInsertUsersAsync(IEnumerable<User> users)
{
    // 使用AddRange提高性能
    _context.Users.AddRange(users);
    await _context.SaveChangesAsync();
}

// 批量更新
public async Task BulkUpdateUserStatusAsync(IEnumerable<int> userIds, string status)
{
    await _context.Users
        .Where(u => userIds.Contains(u.Id))
        .ExecuteUpdateAsync(s => s.SetProperty(u => u.Status, status));
}

// 批量删除
public async Task BulkDeleteUsersAsync(IEnumerable<int> userIds)
{
    await _context.Users
        .Where(u => userIds.Contains(u.Id))
        .ExecuteDeleteAsync();
}

// 使用SqlBulkCopy进行大批量操作
public async Task BulkInsertWithSqlBulkCopyAsync(IEnumerable<User> users)
{
    var connection = _context.Database.GetDbConnection();
    await connection.OpenAsync();
    
    using var transaction = connection.BeginTransaction();
    try
    {
        var dataTable = new DataTable();
        dataTable.Columns.Add("FirstName", typeof(string));
        dataTable.Columns.Add("LastName", typeof(string));
        dataTable.Columns.Add("Email", typeof(string));
        
        foreach (var user in users)
        {
            dataTable.Rows.Add(user.FirstName, user.LastName, user.Email);
        }
        
        using var bulkCopy = new SqlBulkCopy((SqlConnection)connection, SqlBulkCopyOptions.Default, transaction);
        bulkCopy.DestinationTableName = "Users";
        bulkCopy.ColumnMappings.Add("FirstName", "FirstName");
        bulkCopy.ColumnMappings.Add("LastName", "LastName");
        bulkCopy.ColumnMappings.Add("Email", "Email");
        
        await bulkCopy.WriteToServerAsync(dataTable);
        transaction.Commit();
    }
    catch
    {
        transaction.Rollback();
        throw;
    }
}
```

### 4.2 缓存策略
```csharp
// 内存缓存
public class CachedUserService
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedUserService> _logger;
    
    public CachedUserService(ApplicationDbContext context, IMemoryCache cache, ILogger<CachedUserService> logger)
    {
        _context = context;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<User> GetUserByIdAsync(int id)
    {
        var cacheKey = $"user_{id}";
        
        if (_cache.TryGetValue(cacheKey, out User cachedUser))
        {
            _logger.LogInformation("User {UserId} retrieved from cache", id);
            return cachedUser;
        }
        
        var user = await _context.Users
            .Include(u => u.Profile)
            .FirstOrDefaultAsync(u => u.Id == id);
        
        if (user != null)
        {
            var cacheOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(10))
                .SetAbsoluteExpiration(TimeSpan.FromHours(1));
            
            _cache.Set(cacheKey, user, cacheOptions);
            _logger.LogInformation("User {UserId} cached", id);
        }
        
        return user;
    }
    
    public async Task InvalidateUserCacheAsync(int userId)
    {
        var cacheKey = $"user_{userId}";
        _cache.Remove(cacheKey);
        _logger.LogInformation("User {UserId} cache invalidated", userId);
    }
}

// 分布式缓存
public class DistributedUserService
{
    private readonly ApplicationDbContext _context;
    private readonly IDistributedCache _cache;
    
    public DistributedUserService(ApplicationDbContext context, IDistributedCache cache)
    {
        _context = context;
        _cache = cache;
    }
    
    public async Task<User> GetUserByIdAsync(int id)
    {
        var cacheKey = $"user_{id}";
        var cachedUser = await _cache.GetStringAsync(cacheKey);
        
        if (!string.IsNullOrEmpty(cachedUser))
        {
            return JsonSerializer.Deserialize<User>(cachedUser);
        }
        
        var user = await _context.Users
            .Include(u => u.Profile)
            .FirstOrDefaultAsync(u => u.Id == id);
        
        if (user != null)
        {
            var userJson = JsonSerializer.Serialize(user);
            var cacheOptions = new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(10),
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            };
            
            await _cache.SetStringAsync(cacheKey, userJson, cacheOptions);
        }
        
        return user;
    }
}
```

## 5. 并发控制

### 5.1 乐观并发控制
```csharp
public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    
    // 使用RowVersion进行乐观并发控制
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

public class OrderService
{
    private readonly ApplicationDbContext _context;
    
    public OrderService(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<bool> UpdateOrderAsync(int orderId, decimal newAmount, byte[] rowVersion)
    {
        try
        {
            var order = await _context.Orders.FindAsync(orderId);
            if (order == null)
                return false;
            
            // 检查并发版本
            if (!order.RowVersion.SequenceEqual(rowVersion))
            {
                throw new DbUpdateConcurrencyException("Order has been modified by another user");
            }
            
            order.TotalAmount = newAmount;
            order.RowVersion = rowVersion; // 更新版本
            
            await _context.SaveChangesAsync();
            return true;
        }
        catch (DbUpdateConcurrencyException)
        {
            // 处理并发冲突
            return false;
        }
    }
}
```

### 5.2 悲观并发控制
```csharp
public async Task<bool> UpdateOrderWithLockAsync(int orderId, decimal newAmount)
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        // 使用UPDLOCK获取行级锁
        var order = await _context.Orders
            .FromSqlRaw("SELECT * FROM Orders WITH (UPDLOCK) WHERE Id = {0}", orderId)
            .FirstOrDefaultAsync();
        
        if (order == null)
            return false;
        
        order.TotalAmount = newAmount;
        await _context.SaveChangesAsync();
        
        await transaction.CommitAsync();
        return true;
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **Code First vs Database First**：选择场景和优缺点
2. **迁移管理**：创建、应用、回滚迁移
3. **查询优化**：N+1问题、Include、投影查询
4. **性能优化**：批量操作、缓存策略、CompiledQuery
5. **并发控制**：乐观锁、悲观锁、RowVersion

### 6.2 代码示例准备
- 复杂查询的构建
- 批量操作的实现
- 缓存策略的设计
- 并发控制的处理
- 性能优化的实践

### 6.3 性能优化要点
- 避免N+1查询问题
- 合理使用Include和投影
- 使用AsNoTracking提高查询性能
- 实现适当的缓存策略
- 批量操作提高数据操作性能
