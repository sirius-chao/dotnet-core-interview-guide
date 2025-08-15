# 性能与优化

## 1. 内存管理

### 1.1 内存压力分析

**内存管理的核心挑战**
在.NET应用程序中，内存管理是一个复杂而关键的问题。虽然.NET提供了自动垃圾回收机制，但开发者仍然需要理解内存使用模式，避免内存泄漏和性能问题。

**内存类型和特点**：
1. **托管内存（Managed Memory）**：由CLR管理，包括对象堆、大对象堆等
2. **非托管内存（Unmanaged Memory）**：需要手动管理，如文件句柄、网络连接等
3. **栈内存（Stack Memory）**：存储局部变量和方法调用，自动管理
4. **静态内存（Static Memory）**：应用程序生命周期内持续存在

**内存压力的识别指标**：
- **工作集（Working Set）**：进程当前使用的物理内存
- **私有内存（Private Memory）**：进程独占的内存空间
- **虚拟内存（Virtual Memory）**：进程可访问的地址空间
- **堆大小（Heap Size）**：托管堆当前占用的内存
- **可用内存（Available Memory）**：系统可用的物理内存

**内存泄漏的常见原因**：
- **事件订阅**：对象订阅事件后没有取消订阅
- **静态引用**：静态字段持有大对象的引用
- **循环引用**：对象之间形成循环引用
- **资源未释放**：IDisposable对象没有正确释放
- **缓存无限增长**：缓存没有大小限制和过期策略

**内存优化策略**：
- **对象池化**：重用对象，减少内存分配
- **延迟加载**：按需加载数据，避免一次性加载大量数据
- **分页处理**：分批处理大量数据，控制内存使用
- **弱引用**：使用WeakReference避免阻止对象被回收
- **内存映射文件**：对于大文件，使用内存映射减少内存占用
```csharp
public class MemoryMonitor
{
    private readonly ILogger<MemoryMonitor> _logger;
    private readonly Timer _timer;
    
    public MemoryMonitor(ILogger<MemoryMonitor> logger)
    {
        _logger = logger;
        _timer = new Timer(CheckMemoryPressure, null, TimeSpan.Zero, TimeSpan.FromMinutes(1));
    }
    
    private void CheckMemoryPressure(object state)
    {
        var process = Process.GetCurrentProcess();
        var workingSet = process.WorkingSet64;
        var privateMemory = process.PrivateMemorySize64;
        var virtualMemory = process.VirtualMemorySize64;
        
        var gcInfo = GC.GetGCMemoryInfo();
        var heapSize = gcInfo.HeapSizeBytes;
        var totalAvailableMemory = gcInfo.TotalAvailableMemoryBytes;
        
        _logger.LogInformation(
            "Memory Status - Working Set: {WorkingSetMB}MB, Private: {PrivateMB}MB, " +
            "Virtual: {VirtualMB}MB, Heap: {HeapMB}MB, Available: {AvailableMB}MB",
            workingSet / 1024 / 1024,
            privateMemory / 1024 / 1024,
            virtualMemory / 1024 / 1024,
            heapSize / 1024 / 1024,
            totalAvailableMemory / 1024 / 1024);
        
        // 检查内存压力
        if (workingSet > 1024 * 1024 * 1024) // 1GB
        {
            _logger.LogWarning("High memory usage detected");
            GC.Collect();
        }
    }
}

// 内存使用分析
public class MemoryAnalyzer
{
    public static void AnalyzeMemoryUsage()
    {
        var process = Process.GetCurrentProcess();
        
        Console.WriteLine($"Process ID: {process.Id}");
        Console.WriteLine($"Working Set: {process.WorkingSet64 / 1024 / 1024} MB");
        Console.WriteLine($"Private Memory: {process.PrivateMemorySize64 / 1024 / 1024} MB");
        Console.WriteLine($"Virtual Memory: {process.VirtualMemorySize64 / 1024 / 1024} MB");
        
        // GC信息
        for (int i = 0; i <= GC.MaxGeneration; i++)
        {
            var count = GC.CollectionCount(i);
            Console.WriteLine($"Gen {i} Collections: {count}");
        }
        
        // 内存压力
        var memoryPressure = GC.GetTotalMemory(false);
        Console.WriteLine($"Total Memory: {memoryPressure / 1024 / 1024} MB");
    }
}
```

### 1.2 MemoryPool 使用
```csharp
public class MemoryPoolService
{
    private readonly MemoryPool<byte> _memoryPool;
    
    public MemoryPoolService()
    {
        _memoryPool = MemoryPool<byte>.Shared;
    }
    
    public async Task ProcessDataAsync(Stream inputStream)
    {
        using var memoryOwner = _memoryPool.Rent(8192); // 8KB buffer
        var buffer = memoryOwner.Memory;
        
        int bytesRead;
        while ((bytesRead = await inputStream.ReadAsync(buffer)) > 0)
        {
            var data = buffer.Slice(0, bytesRead);
            await ProcessBufferAsync(data);
        }
    }
    
    private async Task ProcessBufferAsync(ReadOnlyMemory<byte> data)
    {
        // 处理数据
        await Task.Delay(1); // 模拟处理
    }
}

// 自定义内存池
public class CustomMemoryPool
{
    private readonly ConcurrentQueue<byte[]> _pool = new ConcurrentQueue<byte[]>();
    private readonly int _bufferSize;
    private readonly int _maxPoolSize;
    private int _currentPoolSize;
    
    public CustomMemoryPool(int bufferSize, int maxPoolSize)
    {
        _bufferSize = bufferSize;
        _maxPoolSize = maxPoolSize;
    }
    
    public byte[] Rent()
    {
        if (_pool.TryDequeue(out var buffer))
        {
            return buffer;
        }
        
        if (_currentPoolSize < _maxPoolSize)
        {
            Interlocked.Increment(ref _currentPoolSize);
            return new byte[_bufferSize];
        }
        
        return new byte[_bufferSize];
    }
    
    public void Return(byte[] buffer)
    {
        if (buffer.Length == _bufferSize && _pool.Count < _maxPoolSize)
        {
            Array.Clear(buffer, 0, buffer.Length);
            _pool.Enqueue(buffer);
        }
    }
}
```

### 1.3 Span<T> 和 Memory<T> 优化
```csharp
public class SpanOptimizer
{
    // 使用Span<T>避免内存分配
    public static int SumArray(ReadOnlySpan<int> numbers)
    {
        int sum = 0;
        for (int i = 0; i < numbers.Length; i++)
        {
            sum += numbers[i];
        }
        return sum;
    }
    
    // 字符串处理优化
    public static bool IsValidEmail(ReadOnlySpan<char> email)
    {
        if (email.IsEmpty) return false;
        
        var atIndex = email.IndexOf('@');
        if (atIndex <= 0 || atIndex >= email.Length - 1) return false;
        
        var localPart = email.Slice(0, atIndex);
        var domainPart = email.Slice(atIndex + 1);
        
        return !localPart.IsEmpty && !domainPart.IsEmpty && 
               domainPart.IndexOf('.') > 0;
    }
    
    // 数组复制优化
    public static void CopyArray<T>(ReadOnlySpan<T> source, Span<T> destination)
    {
        source.CopyTo(destination);
    }
}

// Memory<T> 用于异步操作
public class MemoryOptimizer
{
    public static async Task<int> ProcessDataAsync(Memory<byte> data)
    {
        // 异步处理数据
        await Task.Delay(10);
        return data.Length;
    }
    
    public static async Task<Memory<byte>> GetDataAsync()
    {
        var data = new byte[1024];
        // 模拟异步获取数据
        await Task.Delay(10);
        return data;
    }
}
```

## 2. GC 优化

### 2.1 GC 配置
```csharp
// 在 Program.cs 中配置GC
public class Program
{
    public static void Main(string[] args)
    {
        // 配置GC
        ConfigureGC();
        
        var host = CreateHostBuilder(args).Build();
        host.Run();
    }
    
    private static void ConfigureGC()
    {
        // 使用服务器GC（多核环境）
        if (Environment.ProcessorCount > 1)
        {
            GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
        }
        
        // 设置GC延迟模式
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
        
        // 注册GC事件
        GC.RegisterForFullGCNotification(10, 10);
        
        var thread = new Thread(GCNotificationThread);
        thread.Start();
    }
    
    private static void GCNotificationThread()
    {
        while (true)
        {
            GC.WaitForFullGCApproach();
            Console.WriteLine("GC approaching...");
            
            GC.WaitForFullGCComplete();
            Console.WriteLine("GC completed.");
        }
    }
}

// GC监控
public class GCMonitor
{
    private readonly ILogger<GCMonitor> _logger;
    private readonly Timer _timer;
    
    public GCMonitor(ILogger<GCMonitor> logger)
    {
        _logger = logger;
        _timer = new Timer(MonitorGC, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
    }
    
    private void MonitorGC(object state)
    {
        var gcInfo = GC.GetGCMemoryInfo();
        
        _logger.LogInformation(
            "GC Status - Heap: {HeapMB}MB, Available: {AvailableMB}MB, " +
            "Gen0: {Gen0Count}, Gen1: {Gen1Count}, Gen2: {Gen2Count}",
            gcInfo.HeapSizeBytes / 1024 / 1024,
            gcInfo.TotalAvailableMemoryBytes / 1024 / 1024,
            GC.CollectionCount(0),
            GC.CollectionCount(1),
            GC.CollectionCount(2));
    }
}
```

### 2.2 大对象堆优化
```csharp
public class LargeObjectOptimizer
{
    // 避免大对象堆分配
    public static byte[] GetBuffer(int size)
    {
        if (size > 85000) // 大对象堆阈值
        {
            // 使用池化或分块处理
            return GetChunkedBuffer(size);
        }
        
        return new byte[size];
    }
    
    private static byte[] GetChunkedBuffer(int size)
    {
        var chunkSize = 8192; // 8KB chunks
        var chunks = new List<byte[]>();
        
        for (int i = 0; i < size; i += chunkSize)
        {
            var currentChunkSize = Math.Min(chunkSize, size - i);
            chunks.Add(new byte[currentChunkSize]);
        }
        
        // 合并chunks（如果需要）
        if (chunks.Count == 1)
            return chunks[0];
        
        var result = new byte[size];
        var offset = 0;
        foreach (var chunk in chunks)
        {
            chunk.CopyTo(result, offset);
            offset += chunk.Length;
        }
        
        return result;
    }
    
    // 使用ArrayPool避免大对象分配
    public static async Task ProcessLargeDataAsync(Stream inputStream)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(8192);
        try
        {
            int bytesRead;
            while ((bytesRead = await inputStream.ReadAsync(buffer, 0, buffer.Length)) > 0)
            {
                var data = new ArraySegment<byte>(buffer, 0, bytesRead);
                await ProcessDataAsync(data);
            }
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}
```

## 3. 异步最佳实践

### 3.1 ValueTask vs Task
```csharp
public class AsyncOptimizer
{
    private readonly Dictionary<int, string> _cache = new Dictionary<int, string>();
    
    // 使用ValueTask避免Task分配
    public ValueTask<string> GetValueAsync(int key)
    {
        if (_cache.TryGetValue(key, out var value))
        {
            return new ValueTask<string>(value); // 同步返回
        }
        
        return new ValueTask<string>(GetValueFromSourceAsync(key)); // 异步返回
    }
    
    private async Task<string> GetValueFromSourceAsync(int key)
    {
        await Task.Delay(100); // 模拟异步操作
        var value = $"Value_{key}";
        _cache[key] = value;
        return value;
    }
    
    // 使用IAsyncEnumerable进行流式处理
    public async IAsyncEnumerable<string> GetValuesAsync(int count)
    {
        for (int i = 0; i < count; i++)
        {
            var value = await GetValueAsync(i);
            yield return value;
        }
    }
    
    // 异步流处理
    public static async Task ProcessStreamAsync(IAsyncEnumerable<string> stream)
    {
        await foreach (var item in stream)
        {
            await ProcessItemAsync(item);
        }
    }
    
    private static async Task ProcessItemAsync(string item)
    {
        await Task.Delay(10); // 模拟处理
    }
}
```

### 3.2 异步反模式避免
```csharp
public class AsyncAntiPatterns
{
    // 反模式：async void（除了事件处理器）
    // public async void BadMethodAsync() { }
    
    // 正确：返回Task
    public async Task GoodMethodAsync()
    {
        await Task.Delay(100);
    }
    
    // 反模式：同步等待异步方法
    public string BadSyncWait()
    {
        // return GetValueAsync(1).Result; // 可能导致死锁
        // return GetValueAsync(1).GetAwaiter().GetResult(); // 同样危险
        
        // 正确：使用同步方法或重构为异步
        return GetValueSync(1);
    }
    
    private string GetValueSync(int key)
    {
        return $"Value_{key}";
    }
    
    // 反模式：在循环中使用async
    public async Task BadLoopAsync()
    {
        var tasks = new List<Task>();
        for (int i = 0; i < 100; i++)
        {
            tasks.Add(ProcessItemAsync(i)); // 正确：并行执行
        }
        
        await Task.WhenAll(tasks);
    }
    
    // 正确：并行处理
    public async Task GoodParallelAsync()
    {
        var items = Enumerable.Range(0, 100);
        var tasks = items.Select(ProcessItemAsync);
        await Task.WhenAll(tasks);
    }
    
    private async Task ProcessItemAsync(int item)
    {
        await Task.Delay(10);
    }
}
```

## 4. 缓存策略

### 4.1 多级缓存
```csharp
public class MultiLevelCache
{
    private readonly IMemoryCache _l1Cache; // 内存缓存
    private readonly IDistributedCache _l2Cache; // 分布式缓存
    private readonly ILogger<MultiLevelCache> _logger;
    
    public MultiLevelCache(
        IMemoryCache l1Cache,
        IDistributedCache l2Cache,
        ILogger<MultiLevelCache> logger)
    {
        _l1Cache = l1Cache;
        _l2Cache = l2Cache;
        _logger = logger;
    }
    
    public async Task<T> GetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null)
    {
        // L1缓存查找
        if (_l1Cache.TryGetValue(key, out T l1Value))
        {
            _logger.LogDebug("L1 cache hit for key: {Key}", key);
            return l1Value;
        }
        
        // L2缓存查找
        var l2Value = await _l2Cache.GetStringAsync(key);
        if (!string.IsNullOrEmpty(l2Value))
        {
            var deserializedValue = JsonSerializer.Deserialize<T>(l2Value);
            
            // 回填L1缓存
            var l1Options = new MemoryCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(5),
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
            };
            _l1Cache.Set(key, deserializedValue, l1Options);
            
            _logger.LogDebug("L2 cache hit for key: {Key}", key);
            return deserializedValue;
        }
        
        // 缓存未命中，执行工厂方法
        var value = await factory();
        
        // 序列化并存储到L2缓存
        var serializedValue = JsonSerializer.Serialize(value);
        var l2Options = new DistributedCacheEntryOptions
        {
            SlidingExpiration = TimeSpan.FromMinutes(10),
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
        };
        await _l2Cache.SetStringAsync(key, serializedValue, l2Options);
        
        // 存储到L1缓存
        var l1Options = new MemoryCacheEntryOptions
        {
            SlidingExpiration = TimeSpan.FromMinutes(5),
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        };
        _l1Cache.Set(key, value, l1Options);
        
        _logger.LogDebug("Cache miss for key: {Key}, value stored", key);
        return value;
    }
    
    public async Task InvalidateAsync(string key)
    {
        _l1Cache.Remove(key);
        await _l2Cache.RemoveAsync(key);
        _logger.LogDebug("Cache invalidated for key: {Key}", key);
    }
}
```

### 4.2 缓存一致性
```csharp
public class CacheConsistencyManager
{
    private readonly IDistributedCache _cache;
    private readonly IEventBus _eventBus;
    private readonly ILogger<CacheConsistencyManager> _logger;
    
    public CacheConsistencyManager(
        IDistributedCache cache,
        IEventBus eventBus,
        ILogger<CacheConsistencyManager> logger)
    {
        _cache = cache;
        _eventBus = eventBus;
        _logger = logger;
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null)
    {
        var serializedValue = JsonSerializer.Serialize(value);
        var options = new DistributedCacheEntryOptions();
        
        if (expiration.HasValue)
        {
            options.AbsoluteExpirationRelativeToNow = expiration;
        }
        
        await _cache.SetStringAsync(key, serializedValue, options);
        
        // 发布缓存更新事件
        await _eventBus.PublishAsync(new CacheUpdatedEvent
        {
            Key = key,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public async Task InvalidatePatternAsync(string pattern)
    {
        // 发布缓存失效事件
        await _eventBus.PublishAsync(new CacheInvalidatedEvent
        {
            Pattern = pattern,
            Timestamp = DateTime.UtcNow
        });
    }
}

// 缓存事件
public class CacheUpdatedEvent
{
    public string Key { get; set; }
    public DateTime Timestamp { get; set; }
}

public class CacheInvalidatedEvent
{
    public string Pattern { get; set; }
    public DateTime Timestamp { get; set; }
}

// 缓存事件处理器
public class CacheEventHandler : IEventHandler<CacheUpdatedEvent>, IEventHandler<CacheInvalidatedEvent>
{
    private readonly IMemoryCache _localCache;
    
    public CacheEventHandler(IMemoryCache localCache)
    {
        _localCache = localCache;
    }
    
    public Task HandleAsync(CacheUpdatedEvent @event)
    {
        // 更新本地缓存
        _localCache.Remove(@event.Key);
        return Task.CompletedTask;
    }
    
    public Task HandleAsync(CacheInvalidatedEvent @event)
    {
        // 根据模式失效本地缓存
        var keysToRemove = GetKeysByPattern(@event.Pattern);
        foreach (var key in keysToRemove)
        {
            _localCache.Remove(key);
        }
        return Task.CompletedTask;
    }
    
    private IEnumerable<string> GetKeysByPattern(string pattern)
    {
        // 实现模式匹配逻辑
        return new List<string>();
    }
}
```

## 5. 代码级优化

### 5.1 JIT 优化
```csharp
public class JITOptimizer
{
    // 内联优化
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static int Add(int a, int b)
    {
        return a + b;
    }
    
    // 避免装箱
    public static void AvoidBoxing()
    {
        // 错误：会导致装箱
        // object obj = 42;
        
        // 正确：使用泛型
        var list = new List<int>();
        list.Add(42);
        
        // 避免ToString装箱
        var number = 42;
        var str = number.ToString(); // 直接调用，无装箱
    }
    
    // 结构体优化
    public struct Point
    {
        public int X, Y;
        
        public Point(int x, int y)
        {
            X = x;
            Y = y;
        }
        
        // 实现Equals避免装箱
        public override bool Equals(object obj)
        {
            if (obj is Point other)
            {
                return X == other.X && Y == other.Y;
            }
            return false;
        }
        
        public override int GetHashCode()
        {
            return HashCode.Combine(X, Y);
        }
    }
}
```

### 5.2 循环优化
```csharp
public class LoopOptimizer
{
    // 循环展开
    public static int SumArray(int[] array)
    {
        int sum = 0;
        int length = array.Length;
        
        // 循环展开，每次处理4个元素
        int i = 0;
        for (; i < length - 3; i += 4)
        {
            sum += array[i] + array[i + 1] + array[i + 2] + array[i + 3];
        }
        
        // 处理剩余元素
        for (; i < length; i++)
        {
            sum += array[i];
        }
        
        return sum;
    }
    
    // 边界检查消除
    public static int SafeArrayAccess(int[] array, int index)
    {
        // 编译器可能优化掉边界检查
        if (index >= 0 && index < array.Length)
        {
            return array[index];
        }
        throw new IndexOutOfRangeException();
    }
    
    // 避免在循环中分配对象
    public static List<string> ProcessItems(List<int> items)
    {
        var result = new List<string>(items.Count); // 预分配容量
        
        foreach (var item in items)
        {
            result.Add($"Item_{item}"); // 避免重复分配
        }
        
        return result;
    }
}
```

### 5.3 字符串优化
```csharp
public class StringOptimizer
{
    // 使用StringBuilder进行字符串拼接
    public static string BuildString(int count)
    {
        var sb = new StringBuilder(count * 10); // 预分配容量
        
        for (int i = 0; i < count; i++)
        {
            sb.Append($"Item_{i}");
        }
        
        return sb.ToString();
    }
    
    // 使用Span<char>进行字符串操作
    public static bool IsPalindrome(ReadOnlySpan<char> text)
    {
        int left = 0;
        int right = text.Length - 1;
        
        while (left < right)
        {
            if (char.ToLowerInvariant(text[left]) != char.ToLowerInvariant(text[right]))
                return false;
            
            left++;
            right--;
        }
        
        return true;
    }
    
    // 字符串池化
    public static string GetString(int id)
    {
        return string.Intern($"User_{id}"); // 字符串池化
    }
}
```

### 5.4 集合优化
```csharp
public class CollectionOptimizer
{
    // 选择合适的集合类型
    public static void ChooseRightCollection()
    {
        // 频繁查找：使用HashSet
        var uniqueItems = new HashSet<string>();
        
        // 频繁插入/删除：使用LinkedList
        var linkedList = new LinkedList<int>();
        
        // 需要排序：使用SortedSet
        var sortedSet = new SortedSet<int>();
        
        // 键值对：根据访问模式选择
        var dictionary = new Dictionary<string, int>(); // 无序
        var sortedDictionary = new SortedDictionary<string, int>(); // 有序
    }
    
    // 预分配容量
    public static List<int> CreateListWithCapacity(int capacity)
    {
        return new List<int>(capacity);
    }
    
    // 使用值类型集合
    public static void UseValueTypeCollections()
    {
        // 避免装箱
        var intList = new List<int>();
        var pointList = new List<Point>();
        
        // 使用Stack<T>和Queue<T>
        var stack = new Stack<int>();
        var queue = new Queue<int>();
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **内存管理**：内存压力分析、MemoryPool使用、Span<T>/Memory<T>
2. **GC优化**：GC配置、大对象堆、延迟模式
3. **异步优化**：ValueTask vs Task、IAsyncEnumerable、反模式避免
4. **缓存策略**：多级缓存、一致性、失效策略
5. **代码优化**：JIT优化、循环优化、字符串优化

### 6.2 代码示例准备
- 内存监控和分析工具
- GC配置和监控实现
- 异步性能优化实践
- 多级缓存架构设计
- 代码级性能优化技巧

### 6.3 性能优化要点
- 避免不必要的内存分配
- 合理使用缓存策略
- 选择合适的数据结构
- 利用编译器优化
- 监控和分析性能瓶颈
