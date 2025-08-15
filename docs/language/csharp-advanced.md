# C# 高级特性

## 1. 异步编程 (Async/Await)

### 1.1 基本概念

#### Task vs Thread 的区别与联系
**Thread（线程）** 是操作系统级别的概念，直接映射到操作系统线程。每个Thread都有独立的栈空间和上下文，创建和销毁开销较大。

**Task（任务）** 是.NET中的高级抽象，基于线程池（ThreadPool）实现。Task提供了更丰富的功能：
- **可组合性**：支持任务链式操作（ContinueWith、WhenAll、WhenAny）
- **异常处理**：统一的异常传播机制
- **取消支持**：通过CancellationToken实现优雅取消
- **性能优化**：线程池复用，减少线程创建销毁开销

**工作原理**：Task内部使用状态机模式，当遇到await时，当前线程返回线程池，等待异步操作完成后，线程池中的任意可用线程继续执行后续代码。

#### async/await 编译器魔法
**async关键字** 告诉编译器这个方法包含异步操作，编译器会自动生成一个状态机类。

**await关键字** 表示等待异步操作完成，编译器会：
1. 检查Task是否已完成，如果已完成直接获取结果
2. 如果未完成，注册延续（continuation）并返回
3. 异步操作完成后，通过状态机恢复执行

**状态机原理**：
```csharp
// 编译器生成的代码大致如下
public class AsyncStateMachine
{
    private int _state;
    private TaskAwaiter<string> _awaiter;
    
    public void MoveNext()
    {
        switch (_state)
        {
            case 0:
                // 开始执行
                _awaiter = GetDataAsync().GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    _state = 1;
                    _awaiter.OnCompleted(MoveNext);
                    return;
                }
                goto case 1;
            case 1:
                // 异步操作完成，获取结果
                var result = _awaiter.GetResult();
                // 继续执行后续代码
                break;
        }
    }
}
```

#### ConfigureAwait(false) 深度解析
**同步上下文（SynchronizationContext）** 是一个抽象概念，代表代码执行的上下文环境：
- **UI线程**：WPF、WinForms有DispatcherSynchronizationContext
- **ASP.NET**：AspNetSynchronizationContext
- **控制台应用**：通常没有特殊的同步上下文

**ConfigureAwait(true)**（默认值）：
- 延续会在原始的同步上下文上执行
- 在UI线程中，确保回到UI线程更新界面
- 在ASP.NET中，确保访问HttpContext等特定对象

**ConfigureAwait(false)**：
- 延续会在任意可用的线程池线程上执行
- 避免不必要的上下文切换，提高性能
- 在库代码中推荐使用，因为库代码通常不需要特定的同步上下文

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

#### 避免 `async void`（除了事件处理器）
**为什么避免 `async void`：**

1. **异常处理问题**：`async void` 方法无法被等待（await），因此异常无法被调用者捕获和处理，可能导致应用程序崩溃。

2. **无法等待完成**：调用者无法知道 `async void` 方法何时完成，也无法等待其完成，这会导致程序逻辑错误。

3. **异常传播到同步上下文**：`async void` 方法的异常会传播到 `SynchronizationContext`，在UI应用程序中可能导致界面冻结。

**正确的做法：**
```csharp
// ✅ 推荐：返回 Task
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
    // 处理逻辑
}

// ✅ 推荐：返回 Task<T>
public async Task<string> GetDataAsync()
{
    await Task.Delay(1000);
    return "Data";
}

// ⚠️ 谨慎使用：仅在事件处理器中使用
public async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessDataAsync();
    }
    catch (Exception ex)
    {
        // 必须处理异常，防止传播
        MessageBox.Show($"Error: {ex.Message}");
    }
}
```

#### 使用 `CancellationToken` 支持取消
- 支持取消
- 正确处理异常和资源清理
- 避免死锁（UI线程中使用ConfigureAwait(true)）

### 1.4 线程取消机制

#### 1.4.1 CancellationToken（推荐方式）

**CancellationToken 设计原理**
CancellationToken 是基于观察者模式设计的取消机制，它实现了以下核心概念：

1. **不可变性（Immutability）**：CancellationToken 本身是不可变的，一旦创建就不能修改其状态
2. **传播性（Propagation）**：取消信号可以沿着调用链向下传播
3. **组合性（Composition）**：多个取消令牌可以组合成一个
4. **性能优化**：使用轻量级的回调机制，避免轮询检查

**内部工作机制**：
- CancellationToken 内部维护一个回调列表
- 当调用 Cancel() 时，所有注册的回调会被同步执行
- 使用 volatile 关键字确保多线程环境下的可见性
- 支持注册/取消注册回调，避免内存泄漏

**使用场景分析**：
- **库代码**：应该接受 CancellationToken 参数，支持外部取消
- **应用代码**：根据业务需求决定是否传递取消令牌
- **长时间运行的操作**：必须支持取消，避免资源浪费
```csharp
public class CancellationTokenExample
{
    public async Task ProcessDataAsync(CancellationToken cancellationToken)
    {
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                await ProcessItemAsync();
                await Task.Delay(100, cancellationToken);
            }
        }
        catch (OperationCanceledException)
        {
            // 处理取消操作
            Console.WriteLine("Operation cancelled");
        }
    }
    
    private async Task ProcessItemAsync()
    {
        // 模拟处理逻辑
        await Task.Delay(50);
    }
}
```

#### 1.4.2 超时取消 - 基于 CancellationTokenSource

**CancellationTokenSource 核心机制**
CancellationTokenSource 是 CancellationToken 的工厂类，负责创建和管理取消令牌的生命周期。

**超时实现原理**：
1. **内部定时器**：CancellationTokenSource 内部使用 Timer 实现超时
2. **自动取消**：超时时间到达后，自动调用 Cancel() 方法
3. **资源管理**：使用 using 语句确保 CancellationTokenSource 被正确释放
4. **性能考虑**：Timer 是轻量级的，不会占用大量系统资源

**超时策略选择**：
- **固定超时**：适用于已知执行时间的操作
- **动态超时**：根据业务逻辑调整超时时间
- **分层超时**：不同层级的操作使用不同的超时时间
- **用户可配置**：允许用户自定义超时时间

**超时处理最佳实践**：
- 超时时间应该合理设置，避免过短导致误判
- 超时后应该记录日志，便于问题排查
- 考虑实现重试机制，提高系统可靠性
- 超时异常应该与业务异常区分处理
```csharp
public class TimeoutCancellationExample
{
    public async Task ProcessWithTimeoutAsync()
    {
        // 设置5秒超时
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        
        try
        {
            await ProcessDataAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Operation timed out after 5 seconds");
        }
    }
    
    public async Task ProcessWithMultipleTimeoutsAsync()
    {
        // 组合多个超时条件
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        using var userCts = new CancellationTokenSource();
        
        // 链接两个取消令牌
        using var combinedCts = CancellationTokenSource.CreateLinkedTokenSource(
            timeoutCts.Token, userCts.Token);
        
        try
        {
            await ProcessDataAsync(combinedCts.Token);
        }
        catch (OperationCanceledException)
        {
            if (timeoutCts.Token.IsCancellationRequested)
                Console.WriteLine("Operation timed out");
            else
                Console.WriteLine("Operation cancelled by user");
        }
    }
}
```

#### 1.4.3 链接取消令牌 - 组合多个取消条件

**链接取消令牌的设计思想**
链接取消令牌允许将多个独立的取消条件组合成一个统一的取消令牌，实现了"任一条件满足即取消"的逻辑。

**技术实现原理**：
1. **观察者模式**：每个源令牌都注册到链接令牌上
2. **事件传播**：任一源令牌取消时，链接令牌立即传播取消信号
3. **资源管理**：链接令牌负责管理所有源令牌的生命周期
4. **性能优化**：使用轻量级的事件机制，避免轮询检查

**组合策略分析**：
- **OR 逻辑**：任一条件满足即取消（默认行为）
- **AND 逻辑**：所有条件都满足才取消（需要自定义实现）
- **优先级策略**：某些取消条件具有更高优先级
- **条件分类**：将取消条件按重要性分类管理

**实际应用场景**：
- **用户取消 + 系统超时**：用户主动取消或系统超时都触发取消
- **资源监控 + 业务超时**：内存不足或业务超时都触发取消
- **多级超时**：不同层级的操作使用不同的超时时间
- **外部依赖**：外部服务不可用时触发取消
```csharp
public class LinkedCancellationExample
{
    public async Task ProcessWithLinkedTokensAsync(CancellationToken userToken)
    {
        // 创建系统级别的取消令牌
        using var systemCts = new CancellationTokenSource();
        
        // 链接用户令牌和系统令牌
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            userToken, systemCts.Token);
        
        // 模拟系统在特定条件下取消
        _ = Task.Run(async () =>
        {
            await Task.Delay(TimeSpan.FromMinutes(1));
            systemCts.Cancel();
        });
        
        try
        {
            await ProcessDataAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            if (userToken.IsCancellationRequested)
                Console.WriteLine("Cancelled by user");
            else if (systemCts.Token.IsCancellationRequested)
                Console.WriteLine("Cancelled by system");
        }
    }
}
```

#### 1.4.4 条件取消 - 基于业务逻辑的取消

**条件取消的架构设计**
条件取消是一种主动监控机制，通过持续监控系统状态和业务指标，在满足特定条件时主动触发取消操作。

**监控策略设计**：
1. **资源监控**：监控内存、CPU、磁盘、网络等系统资源
2. **业务指标**：监控业务处理速度、成功率、队列长度等
3. **外部依赖**：监控数据库连接、外部API响应时间等
4. **用户行为**：监控用户操作模式、会话状态等

**监控实现技术**：
- **轮询监控**：定期检查系统状态，适用于变化不频繁的指标
- **事件驱动**：响应系统事件，适用于实时性要求高的场景
- **阈值管理**：设置合理的阈值，避免误触发
- **趋势分析**：分析指标变化趋势，预测潜在问题

**性能影响考虑**：
- 监控频率应该合理设置，避免过度消耗系统资源
- 使用异步监控，不阻塞主业务流程
- 监控逻辑应该轻量级，避免成为性能瓶颈
- 考虑使用缓存机制，减少重复计算
```csharp
public class ConditionalCancellationExample
{
    public async Task ProcessWithConditionsAsync()
    {
        var cts = new CancellationTokenSource();
        
        // 条件1：内存使用过高时取消
        var memoryMonitor = Task.Run(async () =>
        {
            while (!cts.Token.IsCancellationRequested)
            {
                var memoryUsage = GC.GetTotalMemory(false);
                if (memoryUsage > 100_000_000) // 100MB
                {
                    cts.Cancel();
                    break;
                }
                await Task.Delay(1000);
            }
        });
        
        // 条件2：CPU使用率过高时取消
        var cpuMonitor = Task.Run(async () =>
        {
            while (!cts.Token.IsCancellationRequested)
            {
                var cpuUsage = GetCpuUsage(); // 假设的方法
                if (cpuUsage > 80) // 80%
                {
                    cts.Cancel();
                    break;
                }
                await Task.Delay(1000);
            }
        });
        
        try
        {
            await ProcessDataAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Operation cancelled due to resource constraints");
        }
    }
    
    private double GetCpuUsage()
    {
        // 模拟获取CPU使用率
        return new Random().Next(0, 100);
    }
}
```

#### 1.4.5 优雅关闭（Graceful Shutdown）

**优雅关闭的设计哲学**
优雅关闭是一种系统设计理念，确保系统在关闭过程中能够：
1. **完成正在进行的操作**：不强制中断用户请求
2. **保存系统状态**：确保数据一致性和完整性
3. **释放系统资源**：避免资源泄漏和内存占用
4. **提供用户反馈**：告知用户系统正在关闭

**关闭流程设计**：
1. **信号接收**：接收关闭信号（Ctrl+C、系统信号、管理命令等）
2. **停止接收新请求**：拒绝新的用户请求，但继续处理已接收的请求
3. **等待完成**：等待所有正在进行的操作完成
4. **资源清理**：释放数据库连接、文件句柄、网络连接等资源
5. **状态保存**：保存必要的系统状态信息
6. **最终退出**：完成所有清理工作后退出

**技术实现要点**：
- **信号处理**：正确注册和处理系统信号
- **超时控制**：设置合理的等待超时时间
- **资源管理**：使用 using 语句和 IDisposable 接口
- **异常处理**：确保关闭过程中的异常不会影响系统稳定性
- **日志记录**：记录关闭过程中的关键步骤和状态
```csharp
public class GracefulShutdownExample
{
    private readonly CancellationTokenSource _shutdownCts = new CancellationTokenSource();
    private readonly List<Task> _runningTasks = new List<Task>();
    
    public async Task StartProcessingAsync()
    {
        // 注册应用程序关闭事件
        Console.CancelKeyPress += (sender, e) =>
        {
            e.Cancel = true; // 阻止立即退出
            _shutdownCts.Cancel();
        };
        
        try
        {
            // 启动多个任务
            for (int i = 0; i < 5; i++)
            {
                var task = ProcessDataAsync(_shutdownCts.Token);
                _runningTasks.Add(task);
            }
            
            // 等待所有任务完成或取消
            await Task.WhenAll(_runningTasks);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Shutdown requested, waiting for tasks to complete...");
            
            // 等待正在运行的任务完成
            var runningTasks = _runningTasks.Where(t => !t.IsCompleted).ToArray();
            if (runningTasks.Length > 0)
            {
                await Task.WhenAll(runningTasks);
            }
        }
    }
    
    private async Task ProcessDataAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            await ProcessItemAsync();
            await Task.Delay(100, cancellationToken);
        }
        
        // 清理资源
        await CleanupAsync();
    }
    
    private async Task CleanupAsync()
    {
        Console.WriteLine("Cleaning up resources...");
        await Task.Delay(100); // 模拟清理操作
    }
}
```

#### 1.4.6 线程取消最佳实践总结
**推荐使用的方式：**
- ✅ **CancellationToken** - 标准、安全、可组合
- ✅ **超时取消** - 基于 CancellationTokenSource
- ✅ **链接取消令牌** - 组合多个取消条件
- ✅ **条件取消** - 基于业务逻辑的取消

**不推荐使用的方式：**
- ❌ **Thread.Abort()** - 已弃用，可能导致资源泄漏
- ❌ **Thread.Interrupt()** - 不安全，可能导致异常
- ❌ **手动标志** - 需要额外的同步机制

**最佳实践：**
- 优先使用 `CancellationToken`
- 合理设置超时时间
- 实现优雅关闭机制
- 正确处理取消异常
- 及时清理资源

## 2. LINQ 高级用法

### 2.1 自定义扩展方法

**扩展方法的设计原理**
扩展方法是C#的语法糖，本质上是静态方法的调用。编译器在编译时会将扩展方法调用转换为静态方法调用。

**扩展方法的优势**：
1. **语法简洁**：提供面向对象的调用语法
2. **类型安全**：编译时类型检查，避免运行时错误
3. **可读性强**：代码更加直观和易读
4. **向后兼容**：可以为现有类型添加新功能而不修改原类型

**扩展方法的实现机制**：
- 扩展方法必须是静态类中的静态方法
- 第一个参数使用 `this` 关键字标记
- 编译器自动查找匹配的扩展方法
- 扩展方法的优先级低于实例方法

**性能考虑**：
- 扩展方法调用没有额外的性能开销
- 编译器会内联简单的扩展方法
- 避免在扩展方法中执行复杂的逻辑
- 考虑使用泛型约束提高类型安全性
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

**泛型约束的设计哲学**
泛型约束是C#类型系统的重要组成部分，它通过限制泛型类型参数的范围，在保持类型安全的同时提供编译时检查。

**约束类型的作用机制**：
1. **编译时检查**：编译器在编译时验证类型约束，避免运行时类型错误
2. **类型安全**：确保泛型代码只能操作符合约束的类型
3. **性能优化**：编译器可以根据约束生成更优化的代码
4. **API设计**：通过约束设计更清晰和安全的API接口

**约束类型的层次结构**：
- **接口约束**：最灵活的约束，支持多接口实现
- **基类约束**：确保类型继承自特定基类
- **构造函数约束**：确保类型可以被实例化
- **值类型约束**：限制为值类型，避免装箱操作
- **引用类型约束**：限制为引用类型，支持null值

**约束组合策略**：
- 多个约束可以组合使用，用逗号分隔
- 约束的顺序影响编译器的类型推断
- 某些约束组合可能导致类型无法满足要求
- 应该选择最宽松的约束来满足设计需求
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

**协变和逆变的核心概念**
协变和逆变是泛型类型系统中的重要概念，它们描述了类型参数在继承关系中的变化方向。

**协变（Covariance）**：
- **定义**：如果类型A是类型B的子类型，那么`I<A>`也是`I<B>`的子类型
- **关键字**：使用 `out` 关键字标记
- **应用场景**：通常用于"生产者"接口，如`IEnumerable<T>`
- **类型安全**：协变类型只能作为输出，不能作为输入

**逆变（Contravariance）**：
- **定义**：如果类型A是类型B的子类型，那么`I<B>`是`I<A>`的子类型
- **关键字**：使用 `in` 关键字标记
- **应用场景**：通常用于"消费者"接口，如`Action<T>`
- **类型安全**：逆变类型只能作为输入，不能作为输出

**设计原则（Liskov替换原则）**：
- 协变确保子类型可以替换父类型，保持"输出"的兼容性
- 逆变确保父类型可以替换子类型，保持"输入"的兼容性
- 这种设计使得泛型类型在继承层次中保持类型安全

**实际应用价值**：
- **集合操作**：协变使得`List<string>`可以赋值给`IEnumerable<object>`
- **委托组合**：逆变使得`Action<object>`可以赋值给`Action<string>`
- **接口设计**：通过协变逆变设计更灵活的API接口
- **类型安全**：在编译时确保类型操作的安全性
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

**反射的底层原理**
反射是.NET运行时提供的一种机制，允许程序在运行时检查和操作类型信息。它基于元数据（Metadata）系统，每个.NET程序集都包含详细的类型信息。

**元数据系统架构**：
1. **程序集元数据**：包含程序集的基本信息、版本、依赖等
2. **类型元数据**：包含类型名称、基类、接口、字段、方法等
3. **方法元数据**：包含方法签名、参数、返回值、特性等
4. **字段元数据**：包含字段类型、访问修饰符、默认值等

**反射的性能特点**：
- **启动开销**：首次访问类型信息时需要加载元数据
- **查找开销**：通过字符串查找成员信息相对较慢
- **调用开销**：反射调用方法比直接调用慢10-100倍
- **内存开销**：反射对象会占用额外的内存空间

**使用场景分析**：
- **插件系统**：动态加载和调用插件
- **序列化框架**：根据类型信息进行对象序列化
- **ORM框架**：根据实体类型生成SQL语句
- **配置系统**：根据配置信息动态创建对象
- **测试框架**：动态调用测试方法
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
