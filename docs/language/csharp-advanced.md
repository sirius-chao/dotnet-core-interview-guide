# C# 高级特性

## 1. 异步编程 (Async/Await)

### 1.1 基本概念
- **Task vs Thread**: Task是更高级的抽象，基于线程池，支持取消、异常处理和组合
- **async/await**: 编译器生成的状态机，避免阻塞线程
- **ConfigureAwait(false)**: 避免死锁，提高性能

### 1.2 异步方法示例
```csharp
// 异步方法示例
public async Task<string> GetDataAsync(CancellationToken cancellationToken = default)
{
    try
    {
        using var httpClient = new HttpClient();
        var response = await httpClient.GetAsync("https://api.example.com/data", cancellationToken);
        return await response.Content.ReadAsStringAsync();
    }
    catch (OperationCanceledException)
    {
        // 处理取消操作
        return string.Empty;
    }
}

// ConfigureAwait(false) 使用
await GetDataAsync().ConfigureAwait(false);
```

### 1.3 异步最佳实践
- 避免 `async void`（除了事件处理器）
- 使用 `CancellationToken` 支持取消
- 正确处理异常和资源清理
- 避免死锁（UI线程中使用ConfigureAwait(true)）

## 2. LINQ 高级用法

### 2.1 自定义扩展方法
```csharp
public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereIf<T>(this IEnumerable<T> source, bool condition, Func<T, bool> predicate)
    {
        return condition ? source.Where(predicate) : source;
    }
    
    public static IEnumerable<T> DistinctBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seenKeys = new HashSet<TKey>();
        return source.Where(element => seenKeys.Add(keySelector(element)));
    }
}
```

### 2.2 性能优化
- 使用 `AsParallel()` 进行并行处理
- 避免在LINQ中执行数据库查询
- 使用 `ToList()` 或 `ToArray()` 避免重复枚举

## 3. 泛型高级特性

### 3.1 约束类型
```csharp
// 接口约束
public class Repository<T> where T : IEntity

// 基类约束
public class Service<T> where T : BaseEntity

// 构造函数约束
public class Factory<T> where T : new()

// 值类型约束
public class Cache<T> where T : struct

// 引用类型约束
public class Handler<T> where T : class
```

### 3.2 协变和逆变
```csharp
// 协变 (out)
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // 协变

// 逆变 (in)
Action<object> objectAction = obj => Console.WriteLine(obj);
Action<string> stringAction = objectAction; // 逆变

// 自定义协变接口
public interface IProducer<out T>
{
    T GetItem();
}

// 自定义逆变接口
public interface IConsumer<in T>
{
    void Process(T item);
}
```

## 4. 反射 (Reflection)

### 4.1 类型信息获取
```csharp
var type = typeof(MyClass);
var properties = type.GetProperties();
var methods = type.GetMethods();
var attributes = type.GetCustomAttributes();

// 获取泛型类型
var genericType = typeof(List<>);
var closedType = genericType.MakeGenericType(typeof(string));
```

### 4.2 动态创建实例
```csharp
var instance = Activator.CreateInstance<MyClass>();
var instance2 = Activator.CreateInstance(type, "constructor parameter");

// 使用构造函数信息
var ctor = type.GetConstructor(new[] { typeof(string) });
var instance3 = ctor.Invoke(new object[] { "parameter" });
```

### 4.3 性能考虑
- 反射操作较慢，避免在性能关键路径使用
- 缓存反射结果
- 考虑使用 `Expression` 或 `IL` 生成代码

## 5. 表达式树 (Expression Trees)

### 5.1 基本概念
表达式树是代码的树形表示，可以：
- 在运行时分析和修改代码
- 转换为其他语言（如SQL）
- 动态生成代码

### 5.2 创建表达式树
```csharp
Expression<Func<Person, bool>> expression = p => p.Age > 18 && p.Name.StartsWith("A");

// 手动构建表达式树
var parameter = Expression.Parameter(typeof(Person), "p");
var ageProperty = Expression.Property(parameter, "Age");
var constant18 = Expression.Constant(18);
var ageComparison = Expression.GreaterThan(ageProperty, constant18);
var lambda = Expression.Lambda<Func<Person, bool>>(ageComparison, parameter);
```

### 5.3 动态查询构建
```csharp
public static IQueryable<T> WhereDynamic<T>(this IQueryable<T> source, string propertyName, object value)
{
    var parameter = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(parameter, propertyName);
    var constant = Expression.Constant(value);
    var comparison = Expression.GreaterThan(property, constant);
    var lambda = Expression.Lambda<Func<T, bool>>(comparison, parameter);

    return source.Where(lambda);
}
```

## 6. 模式匹配 (Pattern Matching)

### 6.1 switch 表达式
```csharp
var result = obj switch
{
    string s => $"String: {s}",
    int i => $"Integer: {i}",
    Person p => $"Person: {p.Name}",
    _ => "Unknown type"
};
```

### 6.2 is 模式匹配
```csharp
if (obj is string s)
{
    Console.WriteLine($"String length: {s.Length}");
}

// 属性模式
if (person is { Age: > 18, Name: { Length: > 0 } })
{
    Console.WriteLine("Valid adult person");
}

// 元组模式
if ((x, y) is (0, 0))
{
    Console.WriteLine("Origin point");
}
```

### 6.3 递归模式
```csharp
public static int GetDepth(object obj) => obj switch
{
    null => 0,
    string => 1,
    IEnumerable<object> enumerable => 1 + enumerable.Max(GetDepth),
    _ => 1
};
```

## 7. 值类型优化

### 7.1 Span<T> 和 Memory<T>
```csharp
// Span<T> - 栈分配，高性能
public static int Sum(ReadOnlySpan<int> numbers)
{
    int sum = 0;
    for (int i = 0; i < numbers.Length; i++)
    {
        sum += numbers[i];
    }
    return sum;
}

// Memory<T> - 堆分配，可传递
public static async Task<int> SumAsync(Memory<int> numbers)
{
    // 异步操作
    return Sum(numbers.Span);
}
```

### 7.2 ValueTask
```csharp
public static ValueTask<int> GetValueAsync()
{
    // 同步返回
    if (cachedValue.HasValue)
        return new ValueTask<int>(cachedValue.Value);
    
    // 异步返回
    return new ValueTask<int>(GetValueFromSourceAsync());
}
```

## 8. 面试重点

### 8.1 高频问题
1. **async/await 工作原理**：状态机、线程池、ConfigureAwait
2. **泛型约束和协变逆变**：理解类型安全
3. **反射性能影响**：何时使用，如何优化
4. **表达式树应用**：动态查询、代码生成
5. **模式匹配优势**：类型安全、性能提升

### 8.2 代码示例准备
- 异步方法的异常处理
- 自定义LINQ扩展方法
- 泛型约束的实际应用
- 反射的动态对象创建
- 表达式树的动态查询构建

### 8.3 性能优化要点
- 避免在循环中使用反射
- 合理使用ConfigureAwait
- 缓存表达式树编译结果
- 使用Span<T>减少内存分配
