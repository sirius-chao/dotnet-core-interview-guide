# 云原生与容器化

## 1. Docker 容器化

### 1.1 Dockerfile 最佳实践
```dockerfile
# 多阶段构建示例
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]

# 优化后的Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# 安装必要的包
RUN apk add --no-cache icu-libs

# 设置环境变量
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=true

FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# 复制项目文件并恢复依赖
COPY ["MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# 复制源代码
COPY . .
WORKDIR "/src/MyApp"

# 构建应用
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish \
    --no-restore \
    --no-build \
    -p:PublishTrimmed=true \
    -p:PublishSingleFile=true

FROM base AS final
WORKDIR /app

# 创建非root用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 复制发布文件
COPY --from=publish /app/publish .

# 设置权限
RUN chown -R appuser:appgroup /app
USER appuser

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 1.2 Docker Compose 配置
```yaml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
    ports:
      - "8080:80"
      - "8443:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyApp;User=sa;Password=Your_password123
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Your_password123
      - MSSQL_PID=Express
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - webapp
    networks:
      - app-network
    restart: unless-stopped

volumes:
  sqlserver_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### 1.3 容器安全最佳实践
```csharp
// 容器健康检查
public class HealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            // 检查数据库连接
            if (!CheckDatabaseConnection())
            {
                return Task.FromResult(HealthCheckResult.Unhealthy("Database connection failed"));
            }
            
            // 检查Redis连接
            if (!CheckRedisConnection())
            {
                return Task.FromResult(HealthCheckResult.Unhealthy("Redis connection failed"));
            }
            
            // 检查磁盘空间
            if (!CheckDiskSpace())
            {
                return Task.FromResult(HealthCheckResult.Unhealthy("Insufficient disk space"));
            }
            
            return Task.FromResult(HealthCheckResult.Healthy());
        }
        catch (Exception ex)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(exception: ex));
        }
    }
    
    private bool CheckDatabaseConnection()
    {
        // 实现数据库连接检查
        return true;
    }
    
    private bool CheckRedisConnection()
    {
        // 实现Redis连接检查
        return true;
    }
    
    private bool CheckDiskSpace()
    {
        // 实现磁盘空间检查
        return true;
    }
}

// 容器配置验证
public class ContainerConfigurationValidator
{
    public static void ValidateConfiguration(IConfiguration configuration)
    {
        var requiredSettings = new[]
        {
            "ConnectionStrings:DefaultConnection",
            "Redis:ConnectionString",
            "Logging:LogLevel:Default"
        };
        
        foreach (var setting in requiredSettings)
        {
            if (string.IsNullOrEmpty(configuration[setting]))
            {
                throw new InvalidOperationException($"Required configuration setting '{setting}' is missing");
            }
        }
    }
}
```

## 2. Kubernetes 部署

### 2.1 Pod 和 Service 配置
```yaml
# Pod配置
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: myapp
    image: myapp:latest
    ports:
    - containerPort: 80
      name: http
    - containerPort: 443
      name: https
    env:
    - name: ASPNETCORE_ENVIRONMENT
      value: "Production"
    - name: ConnectionStrings__DefaultConnection
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: connection-string
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    livenessProbe:
      httpGet:
        path: /health/live
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    securityContext:
      runAsNonRoot: true
      runAsUser: 1001
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

---
# Service配置
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  labels:
    app: myapp
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    app: myapp
```

### 2.2 Deployment 和 ConfigMap
```yaml
# Deployment配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: environment
        - name: Logging__LogLevel__Default
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: log-level
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: myapp-config
      restartPolicy: Always

---
# ConfigMap配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  environment: "Production"
  log-level: "Information"
  app-settings: |
    {
      "CacheTimeout": "00:10:00",
      "MaxRetryCount": 3,
      "EnableMetrics": true
    }

---
# Secret配置
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  connection-string: U2VydmVyPWRiO0RhdGFiYXNlPU15QXBwO1VzZXI9c2E7UGFzc3dvcmQ9WW91cl9wYXNzd29yZDEyMw==
```

### 2.3 Ingress 和 HPA 配置
```yaml
# Ingress配置
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80

---
# HPA配置
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

## 3. 微服务部署

### 3.1 服务网格 (Service Mesh)
```yaml
# Istio VirtualService配置
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-virtualservice
spec:
  hosts:
  - myapp.example.com
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /api/v1
    route:
    - destination:
        host: myapp-service
        port:
          number: 80
        subset: v1
      weight: 90
    - destination:
        host: myapp-service
        port:
          number: 80
        subset: v2
      weight: 10
  - match:
    - uri:
        prefix: /api/v2
    route:
    - destination:
        host: myapp-service
        port:
          number: 80
        subset: v2
      weight: 100

---
# Istio DestinationRule配置
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-destinationrule
spec:
  host: myapp-service
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
          connectTimeout: 30ms
        http:
          http2MaxRequests: 1000
          maxRequestsPerConnection: 10
          maxRetries: 3
      outlierDetection:
        consecutive5xxErrors: 5
        interval: 10s
        baseEjectionTime: 30s
        maxEjectionPercent: 10
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 50
          connectTimeout: 30ms
        http:
          http2MaxRequests: 500
          maxRequestsPerConnection: 5
          maxRetries: 2
      outlierDetection:
        consecutive5xxErrors: 3
        interval: 10s
        baseEjectionTime: 60s
        maxEjectionPercent: 20
```

### 3.2 熔断器和重试策略
```csharp
// 熔断器实现
public class CircuitBreaker
{
    private readonly ILogger<CircuitBreaker> _logger;
    private readonly TimeSpan _timeout;
    private readonly int _failureThreshold;
    private readonly TimeSpan _resetTimeout;
    
    private CircuitBreakerState _state = CircuitBreakerState.Closed;
    private int _failureCount = 0;
    private DateTime _lastFailureTime;
    
    public CircuitBreaker(
        ILogger<CircuitBreaker> logger,
        TimeSpan timeout,
        int failureThreshold,
        TimeSpan resetTimeout)
    {
        _logger = logger;
        _timeout = timeout;
        _failureThreshold = failureThreshold;
        _resetTimeout = resetTimeout;
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (_state == CircuitBreakerState.Open)
        {
            if (DateTime.UtcNow - _lastFailureTime > _resetTimeout)
            {
                _logger.LogInformation("Circuit breaker attempting to close");
                _state = CircuitBreakerState.HalfOpen;
            }
            else
            {
                throw new CircuitBreakerOpenException("Circuit breaker is open");
            }
        }
        
        try
        {
            using var cts = new CancellationTokenSource(_timeout);
            var result = await action().WaitAsync(cts.Token);
            
            if (_state == CircuitBreakerState.HalfOpen)
            {
                _logger.LogInformation("Circuit breaker closed successfully");
                _state = CircuitBreakerState.Closed;
                _failureCount = 0;
            }
            
            return result;
        }
        catch (Exception ex)
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;
            
            if (_failureCount >= _failureThreshold)
            {
                _logger.LogWarning("Circuit breaker opened after {FailureCount} failures", _failureCount);
                _state = CircuitBreakerState.Open;
            }
            
            throw;
        }
    }
}

public enum CircuitBreakerState
{
    Closed,
    HalfOpen,
    Open
}

public class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}

// 重试策略
public class RetryPolicy
{
    private readonly ILogger<RetryPolicy> _logger;
    private readonly int _maxRetries;
    private readonly TimeSpan _baseDelay;
    private readonly TimeSpan _maxDelay;
    
    public RetryPolicy(
        ILogger<RetryPolicy> logger,
        int maxRetries = 3,
        TimeSpan? baseDelay = null,
        TimeSpan? maxDelay = null)
    {
        _logger = logger;
        _maxRetries = maxRetries;
        _baseDelay = baseDelay ?? TimeSpan.FromSeconds(1);
        _maxDelay = maxDelay ?? TimeSpan.FromSeconds(30);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action, CancellationToken cancellationToken = default)
    {
        var lastException = default(Exception);
        
        for (int attempt = 0; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await action();
            }
            catch (Exception ex) when (attempt < _maxRetries && IsRetryableException(ex))
            {
                lastException = ex;
                var delay = CalculateDelay(attempt);
                
                _logger.LogWarning(ex, "Attempt {Attempt} failed, retrying in {Delay}ms", attempt + 1, delay.TotalMilliseconds);
                
                await Task.Delay(delay, cancellationToken);
            }
        }
        
        throw new RetryPolicyException($"Operation failed after {_maxRetries + 1} attempts", lastException);
    }
    
    private TimeSpan CalculateDelay(int attempt)
    {
        var delay = TimeSpan.FromMilliseconds(_baseDelay.TotalMilliseconds * Math.Pow(2, attempt));
        return TimeSpan.FromMilliseconds(Math.Min(delay.TotalMilliseconds, _maxDelay.TotalMilliseconds));
    }
    
    private bool IsRetryableException(Exception ex)
    {
        return ex is HttpRequestException ||
               ex is TaskCanceledException ||
               ex is TimeoutException ||
               ex is SqlException sqlEx && IsRetryableSqlException(sqlEx);
    }
    
    private bool IsRetryableSqlException(SqlException sqlEx)
    {
        var retryableErrorNumbers = new[] { 2, 53, 64, 233, 10053, 10054, 10060, 40197, 40501, 40613, 49918, 49919, 49920 };
        return retryableErrorNumbers.Contains(sqlEx.Number);
    }
}

public class RetryPolicyException : Exception
{
    public RetryPolicyException(string message, Exception innerException) 
        : base(message, innerException) { }
}
```

## 4. 云平台集成

### 4.1 Azure 集成
```csharp
// Azure Key Vault集成
public class AzureKeyVaultService
{
    private readonly IKeyVaultClient _keyVaultClient;
    private readonly string _vaultUrl;
    
    public AzureKeyVaultService(IKeyVaultClient keyVaultClient, IConfiguration configuration)
    {
        _keyVaultClient = keyVaultClient;
        _vaultUrl = configuration["AzureKeyVault:VaultUrl"];
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var secret = await _keyVaultClient.GetSecretAsync($"{_vaultUrl}/secrets/{secretName}");
            return secret.Value;
        }
        catch (Exception ex)
        {
            throw new InvalidOperationException($"Failed to retrieve secret '{secretName}' from Key Vault", ex);
        }
    }
    
    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        try
        {
            await _keyVaultClient.SetSecretAsync(_vaultUrl, secretName, secretValue);
        }
        catch (Exception ex)
        {
            throw new InvalidOperationException($"Failed to set secret '{secretName}' in Key Vault", ex);
        }
    }
}

// Azure Service Bus集成
public class AzureServiceBusService
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<AzureServiceBusService> _logger;
    
    public AzureServiceBusService(ServiceBusClient client, ILogger<AzureServiceBusService> logger)
    {
        _client = client;
        _logger = logger;
    }
    
    public async Task SendMessageAsync(string queueName, object message)
    {
        var sender = _client.CreateSender(queueName);
        try
        {
            var jsonMessage = JsonSerializer.Serialize(message);
            var serviceBusMessage = new ServiceBusMessage(jsonMessage);
            
            await sender.SendMessageAsync(serviceBusMessage);
            _logger.LogInformation("Message sent to queue {QueueName}", queueName);
        }
        finally
        {
            await sender.DisposeAsync();
        }
    }
    
    public async Task ProcessMessagesAsync(string queueName, Func<ServiceBusReceivedMessage, Task> processor)
    {
        var receiver = _client.CreateReceiver(queueName);
        try
        {
            while (true)
            {
                var message = await receiver.ReceiveMessageAsync();
                if (message == null) break;
                
                try
                {
                    await processor(message);
                    await receiver.CompleteMessageAsync(message);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing message {MessageId}", message.MessageId);
                    await receiver.AbandonMessageAsync(message);
                }
            }
        }
        finally
        {
            await receiver.DisposeAsync();
        }
    }
}
```

### 4.2 AWS 集成
```csharp
// AWS Secrets Manager集成
public class AwsSecretsManagerService
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly ILogger<AwsSecretsManagerService> _logger;
    
    public AwsSecretsManagerService(IAmazonSecretsManager secretsManager, ILogger<AwsSecretsManagerService> logger)
    {
        _secretsManager = secretsManager;
        _logger = logger;
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            var request = new GetSecretValueRequest
            {
                SecretId = secretName
            };
            
            var response = await _secretsManager.GetSecretValueAsync(request);
            return response.SecretString;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve secret '{SecretName}' from AWS Secrets Manager", secretName);
            throw;
        }
    }
    
    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        try
        {
            var request = new PutSecretValueRequest
            {
                SecretId = secretName,
                SecretString = secretValue
            };
            
            await _secretsManager.PutSecretValueAsync(request);
            _logger.LogInformation("Secret '{SecretName}' updated in AWS Secrets Manager", secretName);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to update secret '{SecretName}' in AWS Secrets Manager", secretName);
            throw;
        }
    }
}

// AWS SQS集成
public class AwsSqsService
{
    private readonly IAmazonSQS _sqsClient;
    private readonly ILogger<AwsSqsService> _logger;
    
    public AwsSqsService(IAmazonSQS sqsClient, ILogger<AwsSqsService> logger)
    {
        _sqsClient = sqsClient;
        _logger = logger;
    }
    
    public async Task<string> SendMessageAsync(string queueUrl, object message)
    {
        try
        {
            var jsonMessage = JsonSerializer.Serialize(message);
            var request = new SendMessageRequest
            {
                QueueUrl = queueUrl,
                MessageBody = jsonMessage
            };
            
            var response = await _sqsClient.SendMessageAsync(request);
            _logger.LogInformation("Message sent to SQS queue {QueueUrl}, MessageId: {MessageId}", queueUrl, response.MessageId);
            
            return response.MessageId;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send message to SQS queue {QueueUrl}", queueUrl);
            throw;
        }
    }
    
    public async Task<List<Message>> ReceiveMessagesAsync(string queueUrl, int maxMessages = 10)
    {
        try
        {
            var request = new ReceiveMessageRequest
            {
                QueueUrl = queueUrl,
                MaxNumberOfMessages = maxMessages,
                WaitTimeSeconds = 20 // 长轮询
            };
            
            var response = await _sqsClient.ReceiveMessageAsync(request);
            return response.Messages;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to receive messages from SQS queue {QueueUrl}", queueUrl);
            throw;
        }
    }
    
    public async Task DeleteMessageAsync(string queueUrl, string receiptHandle)
    {
        try
        {
            var request = new DeleteMessageRequest
            {
                QueueUrl = queueUrl,
                ReceiptHandle = receiptHandle
            };
            
            await _sqsClient.DeleteMessageAsync(request);
            _logger.LogDebug("Message deleted from SQS queue {QueueUrl}", queueUrl);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to delete message from SQS queue {QueueUrl}", queueUrl);
            throw;
        }
    }
}
```

## 5. 监控和日志

### 5.1 应用监控
```csharp
// 应用指标收集
public class ApplicationMetrics
{
    private readonly IMetricsRoot _metrics;
    private readonly Counter _requestCounter;
    private readonly Histogram _requestDuration;
    private readonly Gauge _activeConnections;
    
    public ApplicationMetrics(IMetricsRoot metrics)
    {
        _metrics = metrics;
        
        _requestCounter = _metrics.CreateCounter("http_requests_total", "Total HTTP requests", new CounterConfiguration
        {
            LabelNames = new[] { "method", "endpoint", "status_code" }
        });
        
        _requestDuration = _metrics.CreateHistogram("http_request_duration_seconds", "HTTP request duration", new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint" },
            Buckets = new[] { 0.1, 0.25, 0.5, 1, 2.5, 5, 10 }
        });
        
        _activeConnections = _metrics.CreateGauge("active_connections", "Number of active connections");
    }
    
    public void RecordRequest(string method, string endpoint, int statusCode, TimeSpan duration)
    {
        _requestCounter.WithLabels(method, endpoint, statusCode.ToString()).Inc();
        _requestDuration.WithLabels(method, endpoint).Observe(duration.TotalSeconds);
    }
    
    public void SetActiveConnections(int count)
    {
        _activeConnections.Set(count);
    }
}

// 健康检查
public class HealthCheckService : IHealthCheck
{
    private readonly ILogger<HealthCheckService> _logger;
    private readonly IDbConnection _dbConnection;
    private readonly IRedisConnection _redisConnection;
    
    public HealthCheckService(
        ILogger<HealthCheckService> logger,
        IDbConnection dbConnection,
        IRedisConnection redisConnection)
    {
        _logger = logger;
        _dbConnection = dbConnection;
        _redisConnection = redisConnection;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var checks = new List<(string Name, Task<bool> Check)>
        {
            ("Database", CheckDatabaseAsync()),
            ("Redis", CheckRedisAsync()),
            ("Disk Space", CheckDiskSpaceAsync())
        };
        
        var results = await Task.WhenAll(checks.Select(c => c.Check));
        var failedChecks = checks.Where((c, i) => !results[i]).Select(c => c.Name).ToList();
        
        if (failedChecks.Any())
        {
            return HealthCheckResult.Unhealthy($"Health checks failed: {string.Join(", ", failedChecks)}");
        }
        
        return HealthCheckResult.Healthy();
    }
    
    private async Task<bool> CheckDatabaseAsync()
    {
        try
        {
            await _dbConnection.QueryAsync("SELECT 1");
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return false;
        }
    }
    
    private async Task<bool> CheckRedisAsync()
    {
        try
        {
            await _redisConnection.PingAsync();
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Redis health check failed");
            return false;
        }
    }
    
    private Task<bool> CheckDiskSpaceAsync()
    {
        try
        {
            var drive = new DriveInfo(Path.GetPathRoot(Environment.CurrentDirectory));
            var freeSpacePercent = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;
            return Task.FromResult(freeSpacePercent > 10);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Disk space health check failed");
            return Task.FromResult(false);
        }
    }
}
```

## 6. 面试重点

### 6.1 高频问题
1. **Docker优化**：多阶段构建、镜像大小优化、安全最佳实践
2. **Kubernetes部署**：Pod配置、Service类型、Ingress配置
3. **微服务架构**：服务网格、熔断器、重试策略
4. **云平台集成**：Azure、AWS服务集成
5. **监控和日志**：指标收集、健康检查、日志聚合

### 6.2 代码示例准备
- Dockerfile多阶段构建
- Kubernetes YAML配置
- 服务网格配置
- 云平台SDK使用
- 监控指标收集

### 6.3 云原生要点
- 容器化最佳实践
- 微服务部署策略
- 云平台服务集成
- 监控和可观测性
- 安全性和合规性
