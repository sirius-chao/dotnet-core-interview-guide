# 云原生与容器化

## 1. 容器化技术

### 1.1 Docker 基础

#### 容器概念
- **容器 vs 虚拟机**: 轻量级、共享内核、快速启动
- **镜像层**: 只读层、可写层、UnionFS
- **容器生命周期**: 创建、启动、停止、删除

#### Dockerfile 最佳实践
```dockerfile
# 多阶段构建
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["WebApp/WebApp.csproj", "WebApp/"]
RUN dotnet restore "WebApp/WebApp.csproj"
COPY . .
WORKDIR "/src/WebApp"
RUN dotnet build "WebApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "WebApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WebApp.dll"]
```

#### Docker Compose
```yaml
version: '3.8'
services:
  webapp:
    build: .
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - db
      - redis
  
  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
    ports:
      - "1433:1433"
  
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

### 1.2 容器编排

#### Kubernetes 基础概念
- **Pod**: 最小部署单元
- **Service**: 服务发现和负载均衡
- **Deployment**: 无状态应用部署
- **StatefulSet**: 有状态应用部署
- **ConfigMap/Secret**: 配置管理

#### K8s 部署示例
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dotnet-app
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      containers:
      - name: dotnet-app
        image: your-registry/dotnet-app:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

## 2. 云原生架构

### 2.1 微服务架构

#### 服务网格 (Service Mesh)
- **Istio**: 流量管理、安全、可观测性
- **Linkerd**: 轻量级服务网格
- **Envoy**: 高性能代理

#### 服务发现
```csharp
// 服务注册
public class ServiceRegistration
{
    public string ServiceName { get; set; }
    public string ServiceId { get; set; }
    public string Address { get; set; }
    public int Port { get; set; }
    public string HealthCheckUrl { get; set; }
}

// 健康检查
public class HealthCheckService : IHealthCheckService
{
    public async Task<bool> IsHealthyAsync()
    {
        try
        {
            // 检查数据库连接
            // 检查外部依赖
            // 检查系统资源
            return true;
        }
        catch
        {
            return false;
        }
    }
}
```

### 2.2 云原生设计原则

#### 12-Factor App
1. **代码库**: 版本控制
2. **依赖**: 显式声明
3. **配置**: 环境变量
4. **后端服务**: 资源绑定
5. **构建、发布、运行**: 严格分离
6. **进程**: 无状态
7. **端口绑定**: 自包含
8. **并发**: 进程模型
9. **易处理**: 快速启动/优雅关闭
10. **开发/生产环境等价**: 环境一致性
11. **日志**: 事件流
12. **管理进程**: 一次性任务

#### 配置管理
```csharp
// 环境变量配置
public class ConfigurationService
{
    public string GetConnectionString()
    {
        return Environment.GetEnvironmentVariable("DB_CONNECTION_STRING") 
               ?? "DefaultConnection";
    }
    
    public string GetEnvironment()
    {
        return Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") 
               ?? "Development";
    }
}
```

## 3. 云服务集成

### 3.1 Azure 集成

#### Azure App Service
```csharp
// Azure Key Vault 集成
public class AzureKeyVaultService
{
    private readonly SecretClient _secretClient;
    
    public AzureKeyVaultService(SecretClient secretClient)
    {
        _secretClient = secretClient;
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        var secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value.Value;
    }
}

// 在 Program.cs 中配置
builder.Services.AddAzureKeyVault(
    new Uri(builder.Configuration["KeyVault:BaseUrl"]),
    new DefaultAzureCredential());
```

#### Azure Service Bus
```csharp
public class MessageService
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;
    
    public MessageService(ServiceBusClient client, ServiceBusSender sender)
    {
        _client = client;
        _sender = sender;
    }
    
    public async Task SendMessageAsync(string message)
    {
        var serviceBusMessage = new ServiceBusMessage(message);
        await _sender.SendMessageAsync(serviceBusMessage);
    }
    
    public async Task ProcessMessagesAsync(string queueName)
    {
        var processor = _client.CreateProcessor(queueName);
        processor.ProcessMessageAsync += ProcessMessageAsync;
        processor.ProcessErrorAsync += ProcessErrorAsync;
        
        await processor.StartProcessingAsync();
    }
}
```

### 3.2 AWS 集成

#### AWS SDK
```csharp
// AWS S3 集成
public class S3Service
{
    private readonly IAmazonS3 _s3Client;
    
    public S3Service(IAmazonS3 s3Client)
    {
        _s3Client = s3Client;
    }
    
    public async Task<string> UploadFileAsync(string bucketName, string key, Stream fileStream)
    {
        var putRequest = new PutObjectRequest
        {
            BucketName = bucketName,
            Key = key,
            InputStream = fileStream
        };
        
        await _s3Client.PutObjectAsync(putRequest);
        return $"https://{bucketName}.s3.amazonaws.com/{key}";
    }
}
```

## 4. 监控与可观测性

### 4.1 日志聚合

#### ELK Stack
```csharp
// Serilog 配置
public class Program
{
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog((context, services, configuration) => configuration
                .ReadFrom.Configuration(context.Configuration)
                .ReadFrom.Services(services)
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
                {
                    AutoRegisterTemplate = true,
                    AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv7
                }));
}
```

### 4.2 分布式追踪

#### OpenTelemetry
```csharp
// 在 Program.cs 中配置
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddJaegerExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddPrometheusExporter());
```

### 4.3 健康检查

#### 自定义健康检查
```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly DbContext _context;
    
    public DatabaseHealthCheck(DbContext context)
    {
        _context = context;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is unhealthy", ex);
        }
    }
}

// 注册健康检查
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<ExternalApiHealthCheck>("external-api");
```

## 5. 部署策略

### 5.1 CI/CD 流水线

#### GitHub Actions
```yaml
name: Build and Deploy
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
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
    
    - name: Test
      run: dotnet test --no-build --verbosity normal
    
    - name: Publish
      run: dotnet publish -c Release -o ./publish
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Push to registry
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push myapp:${{ github.sha }}
```

### 5.2 蓝绿部署

#### 部署策略实现
```csharp
public class BlueGreenDeploymentService
{
    public async Task DeployAsync(string version)
    {
        // 1. 部署新版本到绿色环境
        await DeployToGreenEnvironmentAsync(version);
        
        // 2. 健康检查
        if (await IsGreenEnvironmentHealthyAsync())
        {
            // 3. 切换流量
            await SwitchTrafficToGreenAsync();
            
            // 4. 清理蓝色环境
            await CleanupBlueEnvironmentAsync();
        }
        else
        {
            // 回滚
            await RollbackToBlueAsync();
        }
    }
}
```

## 6. 面试重点

### 6.1 技术深度
- **容器原理**: 理解 Docker 的底层实现
- **编排系统**: K8s 的架构和组件
- **云原生设计**: 12-Factor App 原则
- **微服务通信**: 服务发现、负载均衡、熔断

### 6.2 实践经验
- **生产环境部署**: 蓝绿部署、金丝雀发布
- **监控告警**: 日志聚合、指标收集、分布式追踪
- **故障排查**: 容器日志、网络问题、资源限制
- **性能优化**: 镜像大小、启动时间、资源利用率

### 6.3 架构设计
- **服务网格**: Istio 的流量管理
- **云服务集成**: Azure/AWS SDK 使用
- **配置管理**: 环境变量、密钥管理
- **安全考虑**: 容器安全、网络策略、RBAC

### 6.4 运维能力
- **CI/CD 流水线**: 自动化构建部署
- **基础设施即代码**: Terraform、ARM 模板
- **容器编排**: K8s 集群管理
- **云原生监控**: Prometheus、Grafana、Jaeger
