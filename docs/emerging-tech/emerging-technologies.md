# 新兴技术

## 1. .NET 8 新特性

### 1.1 性能优化

#### 原生 AOT 编译

**.NET 8 的技术突破**
.NET 8 是微软推出的长期支持（LTS）版本，带来了多项重大技术突破，特别是在性能优化、云原生支持和开发体验方面有显著提升。

**原生AOT编译的革命性意义**：
1. **启动性能**：应用程序启动时间从毫秒级降低到微秒级
2. **内存占用**：显著减少运行时内存占用，适合资源受限环境
3. **部署简化**：无需安装.NET运行时，直接部署可执行文件
4. **云原生友好**：特别适合容器化部署和Serverless场景
5. **跨平台支持**：支持Windows、Linux、macOS等主流平台

**AOT编译的工作原理**：
- **预编译**：在构建时将IL代码编译为原生机器码
- **优化分析**：在编译时进行深度优化，包括内联、常量折叠等
- **反射限制**：某些反射功能在AOT模式下不可用，需要特殊处理
- **大小优化**：只包含实际使用的代码，减少可执行文件大小

**性能优化的技术手段**：
- **方法内联**：使用MethodImplOptions.AggressiveInlining强制内联
- **内存管理**：使用SkipLocalsInit跳过局部变量初始化
- **SIMD指令**：利用CPU的向量化指令提高计算性能
- **内存布局**：优化对象内存布局，减少缓存未命中
- **垃圾回收**：优化GC策略，减少暂停时间

**适用场景分析**：
- **微服务**：快速启动和低内存占用
- **边缘计算**：资源受限的IoT设备
- **云函数**：Serverless环境下的函数计算
- **游戏开发**：需要高性能的实时应用
- **嵌入式系统**：对性能和资源有严格要求的场景
```csharp
// 启用原生 AOT 编译
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
</PropertyGroup>

// 原生 AOT 优化示例
public class OptimizedService
{
    // 使用 [MethodImpl(MethodImplOptions.AggressiveInlining)] 优化内联
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static int FastCalculation(int a, int b)
    {
        return a + b;
    }
    
    // 使用 [SkipLocalsInit] 跳过局部变量初始化
    [SkipLocalsInit]
    public static unsafe void ProcessBuffer(byte[] buffer)
    {
        fixed (byte* ptr = buffer)
        {
            // 直接操作内存，提高性能
            for (int i = 0; i < buffer.Length; i++)
            {
                ptr[i] = (byte)(ptr[i] + 1);
            }
        }
    }
}
```

#### 集合优化
```csharp
public class CollectionOptimizations
{
    // 使用新的集合类型
    public void DemonstrateNewCollections()
    {
        // FrozenDictionary - 不可变字典，适合只读场景
        var frozenDict = new Dictionary<string, int>
        {
            ["A"] = 1,
            ["B"] = 2,
            ["C"] = 3
        }.ToFrozenDictionary();
        
        // 快速查找
        var value = frozenDict["A"]; // O(1) 查找
        
        // IndexedSet - 支持索引访问的集合
        var indexedSet = new IndexedSet<string> { "A", "B", "C" };
        var first = indexedSet[0]; // 支持索引访问
        
        // 使用新的 LINQ 方法
        var numbers = Enumerable.Range(1, 1000);
        var chunked = numbers.Chunk(100); // 分块处理
        var indexed = numbers.Index(); // 带索引的枚举
    }
}
```

### 1.2 语言特性

#### 模式匹配增强
```csharp
public class PatternMatchingExamples
{
    public string ProcessObject(object obj) => obj switch
    {
        // 列表模式
        [var first, var second, .. var rest] => $"First: {first}, Second: {second}, Rest count: {rest.Count}",
        
        // 属性模式增强
        Person { Age: >= 18, Name: var name } => $"Adult: {name}",
        Person { Age: < 18, Name: var name } => $"Minor: {name}",
        
        // 类型模式
        string s when s.Length > 10 => $"Long string: {s[..10]}...",
        string s => $"Short string: {s}",
        
        // 元组模式
        (int x, int y) when x == y => "Equal coordinates",
        (int x, int y) => $"Coordinates: ({x}, {y})",
        
        _ => "Unknown object"
    };
    
    // 使用 when 子句的 switch 表达式
    public string GetStatus(int value) => value switch
    {
        < 0 => "Negative",
        >= 0 and < 100 => "Normal",
        >= 100 and < 1000 => "High",
        >= 1000 => "Very High"
    };
}
```

#### 记录类型增强
```csharp
// 位置记录类型
public record Point(int X, int Y)
{
    // 计算属性
    public double Distance => Math.Sqrt(X * X + Y * Y);
    
    // 静态成员
    public static Point Origin => new(0, 0);
    
    // 操作符重载
    public static Point operator +(Point a, Point b) => new(a.X + b.X, a.Y + b.Y);
    public static Point operator -(Point a, Point b) => new(a.X - b.X, a.Y - b.Y);
}

// 继承记录类型
public record ColoredPoint(int X, int Y, string Color) : Point(X, Y)
{
    // 自定义 ToString
    public override string ToString() => $"ColoredPoint({X}, {Y}, {Color})";
}

// 使用记录类型
public class RecordUsage
{
    public void DemonstrateRecords()
    {
        var p1 = new Point(1, 2);
        var p2 = new Point(1, 2);
        
        // 值相等性
        Console.WriteLine(p1 == p2); // True
        
        // 不可变性
        var p3 = p1 with { X = 3 }; // 创建新实例
        Console.WriteLine(p3); // Point { X = 3, Y = 2 }
        
        // 解构
        var (x, y) = p1;
        Console.WriteLine($"X: {x}, Y: {y}");
    }
}
```

## 2. 机器学习集成

### 2.1 ML.NET 基础

#### 模型训练
```csharp
public class MLNetExample
{
    public class HouseData
    {
        [LoadColumn(0)]
        public float Size { get; set; }
        
        [LoadColumn(1)]
        public float Bedrooms { get; set; }
        
        [LoadColumn(2)]
        public float Price { get; set; }
    }
    
    public class HousePrediction
    {
        [ColumnName("Score")]
        public float Price { get; set; }
    }
    
    public void TrainModel()
    {
        var context = new MLContext(seed: 42);
        
        // 加载数据
        var data = context.Data.LoadFromTextFile<HouseData>("house-data.csv", separatorChar: ',');
        
        // 数据预处理
        var pipeline = context.Transforms.Concatenate("Features", "Size", "Bedrooms")
            .Append(context.Transforms.NormalizeMinMax("Features"))
            .Append(context.Transforms.Concatenate("Features"))
            .Append(context.Regression.Trainers.Sdca());
        
        // 训练模型
        var model = pipeline.Fit(data);
        
        // 保存模型
        context.Model.Save(model, data.Schema, "house-price-model.zip");
    }
    
    public float PredictPrice(float size, float bedrooms)
    {
        var context = new MLContext();
        var model = context.Model.Load("house-price-model.zip", out var schema);
        
        var predictionEngine = context.Model.CreatePredictionEngine<HouseData, HousePrediction>(model);
        
        var prediction = predictionEngine.Predict(new HouseData
        {
            Size = size,
            Bedrooms = bedrooms
        });
        
        return prediction.Price;
    }
}
```

### 2.2 自定义模型

#### 自定义转换器
```csharp
public class CustomTextFeaturizer : IEstimator<ITransformer>
{
    private readonly string _inputColumn;
    private readonly string _outputColumn;
    
    public CustomTextFeaturizer(string inputColumn, string outputColumn)
    {
        _inputColumn = inputColumn;
        _outputColumn = outputColumn;
    }
    
    public ITransformer Fit(IDataView input)
    {
        return new CustomTextTransformer(_inputColumn, _outputColumn);
    }
    
    private class CustomTextTransformer : ITransformer
    {
        private readonly string _inputColumn;
        private readonly string _outputColumn;
        
        public CustomTextTransformer(string inputColumn, string outputColumn)
        {
            _inputColumn = inputColumn;
            _outputColumn = outputColumn;
        }
        
        public IDataView Transform(IDataView input)
        {
            // 实现文本特征化逻辑
            return input;
        }
        
        public bool IsRowToRowMapper => true;
        public DataViewSchema GetOutputSchema(DataViewSchema inputSchema) => inputSchema;
        public IRowToRowMapper GetRowToRowMapper(DataViewSchema inputSchema) => null;
    }
}
```

## 3. 微服务架构演进

### 3.1 服务网格

#### Istio 集成
```csharp
public class ServiceMeshExample
{
    // 服务发现
    public async Task<string> DiscoverServiceAsync(string serviceName)
    {
        using var client = new HttpClient();
        
        // 通过 Istio 服务发现
        var response = await client.GetAsync($"http://{serviceName}/health");
        return await response.Content.ReadAsStringAsync();
    }
    
    // 熔断器模式
    public async Task<string> CallServiceWithCircuitBreakerAsync(string serviceUrl)
    {
        var policy = Policy
            .Handle<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30));
        
        return await policy.ExecuteAsync(async () =>
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(serviceUrl);
            return await response.Content.ReadAsStringAsync();
        });
    }
}
```

### 3.2 事件驱动架构

#### 事件溯源
```csharp
public abstract class Event
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string EventType { get; set; }
    public long Version { get; set; }
}

public class UserCreatedEvent : Event
{
    public string UserId { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    
    public UserCreatedEvent()
    {
        EventType = nameof(UserCreatedEvent);
    }
}

public class EventStore : IEventStore
{
    private readonly DbContext _context;
    
    public async Task SaveEventsAsync(string aggregateId, IEnumerable<Event> events, long expectedVersion)
    {
        var eventList = events.ToList();
        var version = expectedVersion;
        
        foreach (var @event in eventList)
        {
            version++;
            @event.Version = version;
            
            var eventData = new EventData
            {
                AggregateId = aggregateId,
                Version = version,
                EventType = @event.EventType,
                Data = JsonSerializer.Serialize(@event),
                Timestamp = @event.Timestamp
            };
            
            _context.Events.Add(eventData);
        }
        
        await _context.SaveChangesAsync();
    }
    
    public async Task<IEnumerable<Event>> GetEventsAsync(string aggregateId, long startVersion)
    {
        var events = await _context.Events
            .Where(e => e.AggregateId == aggregateId && e.Version > startVersion)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        return events.Select(e => JsonSerializer.Deserialize<Event>(e.Data));
    }
}
```

## 4. 云原生技术

### 4.1 无服务器计算

#### Azure Functions
```csharp
public class ServerlessExample
{
    [FunctionName("ProcessOrder")]
    public static async Task<IActionResult> ProcessOrder(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
        [Queue("order-queue")] ICollector<string> orderQueue,
        ILogger log)
    {
        log.LogInformation("Processing order request");
        
        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        var order = JsonSerializer.Deserialize<Order>(requestBody);
        
        // 验证订单
        if (!IsValidOrder(order))
        {
            return new BadRequestObjectResult("Invalid order");
        }
        
        // 添加到处理队列
        orderQueue.Add(JsonSerializer.Serialize(order));
        
        return new OkObjectResult("Order queued successfully");
    }
    
    [FunctionName("ProcessOrderQueue")]
    public static async Task ProcessOrderQueue(
        [QueueTrigger("order-queue")] string orderJson,
        [CosmosDB("orders", "orders", ConnectionStringSetting = "CosmosDBConnection")] IAsyncCollector<Order> orders,
        ILogger log)
    {
        var order = JsonSerializer.Deserialize<Order>(orderJson);
        log.LogInformation($"Processing order: {order.Id}");
        
        // 处理订单逻辑
        order.Status = OrderStatus.Processing;
        order.ProcessedAt = DateTime.UtcNow;
        
        await orders.AddAsync(order);
    }
}
```

### 4.2 容器编排

#### Kubernetes 操作器
```csharp
public class CustomResourceDefinition
{
    public string ApiVersion { get; set; } = "v1";
    public string Kind { get; set; } = "CustomResource";
    public Metadata Metadata { get; set; }
    public CustomResourceSpec Spec { get; set; }
}

public class CustomResourceController : IHostedService
{
    private readonly IKubernetes _kubernetes;
    private readonly ILogger<CustomResourceController> _logger;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // 监听自定义资源变化
        var watcher = _kubernetes.ListNamespacedCustomObjectWithHttpMessagesAsync(
            "your-group.com",
            "v1",
            "default",
            "customresources",
            watch: true,
            cancellationToken: cancellationToken);
        
        await foreach (var item in watcher)
        {
            await ProcessCustomResourceAsync(item);
        }
    }
    
    private async Task ProcessCustomResourceAsync(WatchEvent<CustomResourceDefinition> watchEvent)
    {
        switch (watchEvent.Type)
        {
            case WatchEventType.Added:
                await HandleResourceAddedAsync(watchEvent.Object);
                break;
            case WatchEventType.Modified:
                await HandleResourceModifiedAsync(watchEvent.Object);
                break;
            case WatchEventType.Deleted:
                await HandleResourceDeletedAsync(watchEvent.Object);
                break;
        }
    }
}
```

## 5. 边缘计算

### 5.1 IoT 设备集成

#### MQTT 客户端
```csharp
public class IoTDeviceService
{
    private readonly IMqttClient _mqttClient;
    private readonly ILogger<IoTDeviceService> _logger;
    
    public async Task ConnectAsync()
    {
        var options = new MqttClientOptionsBuilder()
            .WithClientId($"dotnet-client-{Guid.NewGuid()}")
            .WithTcpServer("localhost", 1883)
            .WithCredentials("username", "password")
            .WithTls()
            .Build();
        
        await _mqttClient.ConnectAsync(options);
        
        // 订阅主题
        await _mqttClient.SubscribeAsync("devices/+/data");
        await _mqttClient.SubscribeAsync("devices/+/status");
    }
    
    public async Task PublishDeviceDataAsync(string deviceId, object data)
    {
        var message = new MqttApplicationMessageBuilder()
            .WithTopic($"devices/{deviceId}/data")
            .WithPayload(JsonSerializer.Serialize(data))
            .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
            .Build();
        
        await _mqttClient.PublishAsync(message);
    }
    
    public async Task ProcessMessageAsync(MqttApplicationMessageReceivedEventArgs e)
    {
        var topic = e.ApplicationMessage.Topic;
        var payload = Encoding.UTF8.GetString(e.ApplicationMessage.Payload);
        
        _logger.LogInformation($"Received message on topic {topic}: {payload}");
        
        // 处理设备数据
        if (topic.EndsWith("/data"))
        {
            await ProcessDeviceDataAsync(topic, payload);
        }
        else if (topic.EndsWith("/status"))
        {
            await ProcessDeviceStatusAsync(topic, payload);
        }
    }
}
```

### 5.2 边缘 AI

#### 本地模型推理
```csharp
public class EdgeAI
{
    private readonly MLContext _mlContext;
    private readonly ITransformer _model;
    
    public EdgeAI()
    {
        _mlContext = new MLContext();
        _model = _mlContext.Model.Load("edge-model.zip", out var schema);
    }
    
    public async Task<float> PredictAsync(float[] features)
    {
        // 本地模型推理
        var predictionEngine = _mlContext.Model.CreatePredictionEngine<InputData, Prediction>(_model);
        
        var input = new InputData { Features = features };
        var prediction = predictionEngine.Predict(input);
        
        return prediction.Score;
    }
    
    // 模型更新
    public async Task UpdateModelAsync(Stream newModelStream)
    {
        var newModel = _mlContext.Model.Load(newModelStream, out var schema);
        
        // 原子性更新
        var tempPath = Path.GetTempFileName();
        _mlContext.Model.Save(newModel, schema, tempPath);
        
        // 替换旧模型
        File.Move(tempPath, "edge-model.zip", true);
        
        // 重新加载模型
        _model = newModel;
    }
}
```

## 6. 面试重点

### 6.1 技术趋势
- **性能优化**: 原生 AOT、集合优化、内存管理
- **语言特性**: 模式匹配、记录类型、空安全
- **云原生**: 无服务器、容器编排、服务网格

### 6.2 实践应用
- **机器学习**: ML.NET 集成、自定义模型、模型部署
- **事件驱动**: 事件溯源、CQRS、消息队列
- **边缘计算**: IoT 集成、本地推理、模型更新

### 6.3 架构演进
- **微服务**: 服务网格、API 网关、分布式追踪
- **云原生**: 容器化、Kubernetes、DevOps
- **新兴技术**: 区块链、量子计算、5G 应用
