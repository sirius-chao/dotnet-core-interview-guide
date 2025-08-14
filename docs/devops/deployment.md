# DevOps 与部署

## 1. CI/CD 流水线

### 1.1 GitHub Actions 配置
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '8.0.x'
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
        
    - name: Restore dependencies
      run: dotnet restore
      
    - name: Build
      run: dotnet build --no-restore --configuration Release
      
    - name: Run tests
      run: dotnet test --no-build --verbosity normal --configuration Release --collect:"XPlat Code Coverage" --results-directory ./coverage
      
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/coverage.cobertura.xml
        flags: unittests
        name: codecov-umbrella
        
    - name: Build Docker image
      run: docker build -t ${{ env.IMAGE_NAME }}:${{ github.sha }} .
      
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.DOCKER_REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
        
    - name: Push Docker image
      run: |
        docker push ${{ env.IMAGE_NAME }}:${{ github.sha }}
        docker tag ${{ env.IMAGE_NAME }}:${{ github.sha }} ${{ env.IMAGE_NAME }}:latest
        docker push ${{ env.IMAGE_NAME }}:latest

  security-scan:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
    - name: Deploy to staging
      run: |
        echo "Deploying to staging environment..."
        # 部署到测试环境的脚本
        
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying to production environment..."
        # 部署到生产环境的脚本
```

### 1.2 Azure DevOps 流水线
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
    - develop

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  dockerRegistry: 'myregistry.azurecr.io'
  imageName: 'myapp'

stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: Build
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    - task: UseDotNet@2
      inputs:
        version: '8.0.x'
        
    - script: |
        dotnet restore
      displayName: 'Restore NuGet packages'
      
    - script: |
        dotnet build --configuration $(buildConfiguration) --no-restore
      displayName: 'Build solution'
      
    - script: |
        dotnet test --configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"
      displayName: 'Run tests'
      
    - task: PublishCodeCoverageResults@1
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '**/coverage.cobertura.xml'
        
    - script: |
        docker build -t $(dockerRegistry)/$(imageName):$(Build.BuildId) .
      displayName: 'Build Docker image'
      
    - task: Docker@2
      inputs:
        containerRegistry: 'Azure Container Registry'
        repository: '$(imageName)'
        command: 'push'
        tags: |
          $(Build.BuildId)
          latest

- stage: Security
  displayName: 'Security Scan'
  dependsOn: Build
  jobs:
  - job: SecurityScan
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    - script: |
        trivy image $(dockerRegistry)/$(imageName):$(Build.BuildId) --format sarif --output trivy-results.sarif
      displayName: 'Run Trivy security scan'
      
    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: 'trivy-results.sarif'
        artifactName: 'security-scan-results'

- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: Security
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  jobs:
  - deployment: DeployStaging
    environment: 'staging'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              echo "Deploying to staging environment..."
              # 部署脚本
            displayName: 'Deploy to staging'

- stage: DeployProduction
  displayName: 'Deploy to Production'
  dependsOn: Security
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployProduction
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              echo "Deploying to production environment..."
              # 部署脚本
            displayName: 'Deploy to production'
```

### 1.3 Jenkins 流水线
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        DOTNET_VERSION = '8.0.x'
        DOCKER_REGISTRY = 'myregistry.azurecr.io'
        IMAGE_NAME = 'myapp'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup .NET') {
            steps {
                sh "dotnet --version"
                sh "dotnet restore"
            }
        }
        
        stage('Build') {
            steps {
                sh "dotnet build --configuration Release --no-restore"
            }
        }
        
        stage('Test') {
            steps {
                sh "dotnet test --configuration Release --no-build --collect:\"XPlat Code Coverage\""
            }
            post {
                always {
                    publishCoverage adapters: [coberturaAdapter('**/coverage.cobertura.xml')], sourceFileResolver: sourceFiles('NEVER')
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh "trivy image ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest --format sarif --output trivy-results.sarif"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} ."
                sh "docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-registry', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh "docker login ${DOCKER_REGISTRY} -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh "echo 'Deploying to staging environment...'"
                // 部署脚本
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?'
                sh "echo 'Deploying to production environment...'"
                // 部署脚本
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

## 2. 部署策略

### 2.1 蓝绿部署
```csharp
// 蓝绿部署服务
public class BlueGreenDeploymentService
{
    private readonly ILogger<BlueGreenDeploymentService> _logger;
    private readonly IKubernetesClient _kubernetesClient;
    private readonly string _namespace;
    
    public BlueGreenDeploymentService(
        ILogger<BlueGreenDeploymentService> logger,
        IKubernetesClient kubernetesClient,
        IConfiguration configuration)
    {
        _logger = logger;
        _kubernetesClient = kubernetesClient;
        _namespace = configuration["Kubernetes:Namespace"];
    }
    
    public async Task DeployAsync(string imageTag)
    {
        try
        {
            // 确定当前活跃环境
            var currentEnvironment = await GetCurrentActiveEnvironmentAsync();
            var newEnvironment = currentEnvironment == "blue" ? "green" : "blue";
            
            _logger.LogInformation("Starting {NewEnvironment} deployment with image {ImageTag}", newEnvironment, imageTag);
            
            // 部署到新环境
            await DeployToEnvironmentAsync(newEnvironment, imageTag);
            
            // 等待新环境就绪
            await WaitForEnvironmentReadyAsync(newEnvironment);
            
            // 运行健康检查
            if (!await RunHealthChecksAsync(newEnvironment))
            {
                throw new DeploymentException($"Health checks failed for {newEnvironment} environment");
            }
            
            // 切换流量到新环境
            await SwitchTrafficAsync(newEnvironment);
            
            // 清理旧环境
            await CleanupOldEnvironmentAsync(currentEnvironment);
            
            _logger.LogInformation("Successfully deployed to {NewEnvironment} environment", newEnvironment);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Deployment failed");
            throw;
        }
    }
    
    private async Task<string> GetCurrentActiveEnvironmentAsync()
    {
        // 检查当前活跃的环境
        var service = await _kubernetesClient.GetServiceAsync(_namespace, "myapp-service");
        var selector = service.Spec.Selector;
        
        if (selector.ContainsKey("environment") && selector["environment"] == "green")
            return "green";
        
        return "blue";
    }
    
    private async Task DeployToEnvironmentAsync(string environment, string imageTag)
    {
        var deployment = new V1Deployment
        {
            Metadata = new V1ObjectMeta
            {
                Name = $"myapp-{environment}",
                Labels = new Dictionary<string, string>
                {
                    { "app", "myapp" },
                    { "environment", environment }
                }
            },
            Spec = new V1DeploymentSpec
            {
                Replicas = 3,
                Selector = new V1LabelSelector
                {
                    MatchLabels = new Dictionary<string, string>
                    {
                        { "app", "myapp" },
                        { "environment", environment }
                    }
                },
                Template = new V1PodTemplateSpec
                {
                    Metadata = new V1ObjectMeta
                    {
                        Labels = new Dictionary<string, string>
                        {
                            { "app", "myapp" },
                            { "environment", environment }
                        }
                    },
                    Spec = new V1PodSpec
                    {
                        Containers = new List<V1Container>
                        {
                            new V1Container
                            {
                                Name = "myapp",
                                Image = $"myapp:{imageTag}",
                                Ports = new List<V1ContainerPort>
                                {
                                    new V1ContainerPort { ContainerPort = 80 }
                                }
                            }
                        }
                    }
                }
            }
        };
        
        await _kubernetesClient.CreateOrReplaceDeploymentAsync(_namespace, deployment);
    }
    
    private async Task WaitForEnvironmentReadyAsync(string environment)
    {
        var maxWaitTime = TimeSpan.FromMinutes(10);
        var checkInterval = TimeSpan.FromSeconds(10);
        var startTime = DateTime.UtcNow;
        
        while (DateTime.UtcNow - startTime < maxWaitTime)
        {
            var deployment = await _kubernetesClient.GetDeploymentAsync(_namespace, $"myapp-{environment}");
            
            if (deployment.Status.ReadyReplicas == deployment.Status.Replicas)
            {
                _logger.LogInformation("{Environment} environment is ready", environment);
                return;
            }
            
            await Task.Delay(checkInterval);
        }
        
        throw new TimeoutException($"Environment {environment} did not become ready within {maxWaitTime}");
    }
    
    private async Task<bool> RunHealthChecksAsync(string environment)
    {
        try
        {
            var service = await _kubernetesClient.GetServiceAsync(_namespace, $"myapp-{environment}-service");
            var endpoint = $"http://{service.Status.LoadBalancer.Ingress[0].Ip}/health";
            
            using var client = new HttpClient();
            var response = await client.GetAsync(endpoint);
            
            return response.IsSuccessStatusCode;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Health check failed for {Environment} environment", environment);
            return false;
        }
    }
    
    private async Task SwitchTrafficAsync(string environment)
    {
        // 更新主服务的标签选择器
        var service = await _kubernetesClient.GetServiceAsync(_namespace, "myapp-service");
        service.Spec.Selector["environment"] = environment;
        
        await _kubernetesClient.ReplaceServiceAsync(_namespace, service);
        _logger.LogInformation("Traffic switched to {Environment} environment", environment);
    }
    
    private async Task CleanupOldEnvironmentAsync(string environment)
    {
        try
        {
            await _kubernetesClient.DeleteDeploymentAsync(_namespace, $"myapp-{environment}");
            await _kubernetesClient.DeleteServiceAsync(_namespace, $"myapp-{environment}-service");
            
            _logger.LogInformation("Cleaned up {Environment} environment", environment);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to cleanup {Environment} environment", environment);
        }
    }
}
```

### 2.2 金丝雀部署
```csharp
// 金丝雀部署服务
public class CanaryDeploymentService
{
    private readonly ILogger<CanaryDeploymentService> _logger;
    private readonly IKubernetesClient _kubernetesClient;
    private readonly string _namespace;
    
    public CanaryDeploymentService(
        ILogger<CanaryDeploymentService> logger,
        IKubernetesClient kubernetesClient,
        IConfiguration configuration)
    {
        _logger = logger;
        _kubernetesClient = kubernetesClient;
        _namespace = configuration["Kubernetes:Namespace"];
    }
    
    public async Task DeployCanaryAsync(string imageTag, int initialTrafficPercentage = 10)
    {
        try
        {
            _logger.LogInformation("Starting canary deployment with {TrafficPercentage}% traffic", initialTrafficPercentage);
            
            // 部署金丝雀版本
            await DeployCanaryVersionAsync(imageTag);
            
            // 等待金丝雀版本就绪
            await WaitForCanaryReadyAsync();
            
            // 逐步增加流量
            await GraduallyIncreaseTrafficAsync(initialTrafficPercentage);
            
            _logger.LogInformation("Canary deployment completed successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Canary deployment failed");
            await RollbackCanaryAsync();
            throw;
        }
    }
    
    private async Task DeployCanaryVersionAsync(string imageTag)
    {
        var deployment = new V1Deployment
        {
            Metadata = new V1ObjectMeta
            {
                Name = "myapp-canary",
                Labels = new Dictionary<string, string>
                {
                    { "app", "myapp" },
                    { "version", "canary" }
                }
            },
            Spec = new V1DeploymentSpec
            {
                Replicas = 1,
                Selector = new V1LabelSelector
                {
                    MatchLabels = new Dictionary<string, string>
                    {
                        { "app", "myapp" },
                        { "version", "canary" }
                    }
                },
                Template = new V1PodTemplateSpec
                {
                    Metadata = new V1ObjectMeta
                    {
                        Labels = new Dictionary<string, string>
                        {
                            { "app", "myapp" },
                            { "version", "canary" }
                        }
                    },
                    Spec = new V1PodSpec
                    {
                        Containers = new List<V1Container>
                        {
                            new V1Container
                            {
                                Name = "myapp",
                                Image = $"myapp:{imageTag}",
                                Ports = new List<V1ContainerPort>
                                {
                                    new V1ContainerPort { ContainerPort = 80 }
                                }
                            }
                        }
                    }
                }
            }
        };
        
        await _kubernetesClient.CreateOrReplaceDeploymentAsync(_namespace, deployment);
    }
    
    private async Task WaitForCanaryReadyAsync()
    {
        var maxWaitTime = TimeSpan.FromMinutes(5);
        var checkInterval = TimeSpan.FromSeconds(10);
        var startTime = DateTime.UtcNow;
        
        while (DateTime.UtcNow - startTime < maxWaitTime)
        {
            var deployment = await _kubernetesClient.GetDeploymentAsync(_namespace, "myapp-canary");
            
            if (deployment.Status.ReadyReplicas == deployment.Status.Replicas)
            {
                _logger.LogInformation("Canary deployment is ready");
                return;
            }
            
            await Task.Delay(checkInterval);
        }
        
        throw new TimeoutException("Canary deployment did not become ready");
    }
    
    private async Task GraduallyIncreaseTrafficAsync(int initialPercentage)
    {
        var currentPercentage = initialPercentage;
        var maxPercentage = 100;
        var increment = 10;
        var waitTime = TimeSpan.FromMinutes(5);
        
        while (currentPercentage < maxPercentage)
        {
            await UpdateTrafficSplitAsync(currentPercentage);
            _logger.LogInformation("Traffic split updated: {Percentage}% to canary", currentPercentage);
            
            // 等待并监控
            await Task.Delay(waitTime);
            
            // 检查指标
            if (await CheckCanaryMetricsAsync())
            {
                currentPercentage += increment;
            }
            else
            {
                _logger.LogWarning("Canary metrics indicate issues, stopping traffic increase");
                break;
            }
        }
        
        if (currentPercentage >= maxPercentage)
        {
            await PromoteCanaryAsync();
        }
    }
    
    private async Task UpdateTrafficSplitAsync(int canaryPercentage)
    {
        // 更新Istio VirtualService的流量分割
        var virtualService = await _kubernetesClient.GetVirtualServiceAsync(_namespace, "myapp-virtualservice");
        
        var httpRoute = virtualService.Spec.Http[0];
        httpRoute.Route[0].Weight = 100 - canaryPercentage; // 稳定版本
        httpRoute.Route[1].Weight = canaryPercentage; // 金丝雀版本
        
        await _kubernetesClient.ReplaceVirtualServiceAsync(_namespace, virtualService);
    }
    
    private async Task<bool> CheckCanaryMetricsAsync()
    {
        try
        {
            // 检查错误率、延迟等指标
            var errorRate = await GetErrorRateAsync("canary");
            var latency = await GetLatencyAsync("canary");
            
            return errorRate < 0.05 && latency < TimeSpan.FromSeconds(1);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to check canary metrics");
            return false;
        }
    }
    
    private async Task<double> GetErrorRateAsync(string version)
    {
        // 实现从Prometheus获取错误率的逻辑
        return 0.02; // 示例值
    }
    
    private async Task<TimeSpan> GetLatencyAsync(string version)
    {
        // 实现从Prometheus获取延迟的逻辑
        return TimeSpan.FromMilliseconds(500); // 示例值
    }
    
    private async Task PromoteCanaryAsync()
    {
        _logger.LogInformation("Promoting canary to stable version");
        
        // 更新稳定版本部署
        var canaryDeployment = await _kubernetesClient.GetDeploymentAsync(_namespace, "myapp-canary");
        var stableDeployment = await _kubernetesClient.GetDeploymentAsync(_namespace, "myapp-stable");
        
        stableDeployment.Spec.Template.Spec.Containers[0].Image = canaryDeployment.Spec.Template.Spec.Containers[0].Image;
        await _kubernetesClient.ReplaceDeploymentAsync(_namespace, stableDeployment);
        
        // 删除金丝雀部署
        await _kubernetesClient.DeleteDeploymentAsync(_namespace, "myapp-canary");
        
        // 重置流量分割
        await UpdateTrafficSplitAsync(0);
        
        _logger.LogInformation("Canary successfully promoted to stable version");
    }
    
    private async Task RollbackCanaryAsync()
    {
        _logger.LogInformation("Rolling back canary deployment");
        
        try
        {
            await _kubernetesClient.DeleteDeploymentAsync(_namespace, "myapp-canary");
            await UpdateTrafficSplitAsync(0);
            
            _logger.LogInformation("Canary deployment rolled back successfully");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to rollback canary deployment");
        }
    }
}
```

## 3. 基础设施即代码

### 3.1 Terraform 配置
```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "myapp.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}

# 资源组
resource "azurerm_resource_group" "main" {
  name     = "${var.project_name}-${var.environment}-rg"
  location = var.location
  
  tags = var.tags
}

# 应用服务计划
resource "azurerm_service_plan" "main" {
  name                = "${var.project_name}-${var.environment}-plan"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  os_type            = "Linux"
  sku_name           = var.app_service_plan_sku
  
  tags = var.tags
}

# 应用服务
resource "azurerm_linux_web_app" "main" {
  name                = "${var.project_name}-${var.environment}-app"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  service_plan_id    = azurerm_service_plan.main.id
  
  site_config {
    application_stack {
      dotnet_version = "8.0"
    }
    
    always_on = true
    
    health_check_path = "/health"
    
    ip_restriction {
      virtual_network_subnet_id = azurerm_subnet.main.id
    }
  }
  
  app_settings = {
    "WEBSITES_ENABLE_APP_SERVICE_STORAGE" = "false"
    "DOCKER_REGISTRY_SERVER_URL"          = var.docker_registry_url
    "DOCKER_REGISTRY_SERVER_USERNAME"     = var.docker_registry_username
    "DOCKER_REGISTRY_SERVER_PASSWORD"     = var.docker_registry_password
    "WEBSITES_PORT"                       = "80"
  }
  
  tags = var.tags
}

# 数据库
resource "azurerm_mssql_server" "main" {
  name                         = "${var.project_name}-${var.environment}-sql"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password
  
  tags = var.tags
}

resource "azurerm_mssql_database" "main" {
  name           = "${var.project_name}_${var.environment}_db"
  server_id      = azurerm_mssql_server.main.id
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  license_type   = "LicenseIncluded"
  max_size_gb   = var.sql_database_max_size_gb
  sku_name      = var.sql_database_sku
  
  tags = var.tags
}

# Redis缓存
resource "azurerm_redis_cache" "main" {
  name                = "${var.project_name}-${var.environment}-redis"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  capacity           = var.redis_capacity
  family             = var.redis_family
  sku_name           = var.redis_sku_name
  
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
  
  tags = var.tags
}

# 虚拟网络
resource "azurerm_virtual_network" "main" {
  name                = "${var.project_name}-${var.environment}-vnet"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  address_space      = ["10.0.0.0/16"]
  
  tags = var.tags
}

resource "azurerm_subnet" "main" {
  name                 = "default"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

# 应用网关
resource "azurerm_application_gateway" "main" {
  name                = "${var.project_name}-${var.environment}-agw"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  
  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = var.app_gateway_capacity
  }
  
  gateway_ip_configuration {
    name      = "gateway-ip-config"
    subnet_id = azurerm_subnet.main.id
  }
  
  frontend_port {
    name = "http-port"
    port = 80
  }
  
  frontend_port {
    name = "https-port"
    port = 443
  }
  
  frontend_ip_configuration {
    name                 = "frontend-ip-config"
    public_ip_address_id = azurerm_public_ip.main.id
  }
  
  backend_address_pool {
    name = "backend-pool"
    fqdn = azurerm_linux_web_app.main.default_hostname
  }
  
  backend_http_settings {
    name                  = "http-settings"
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }
  
  http_listener {
    name                           = "http-listener"
    frontend_ip_configuration_name = "frontend-ip-config"
    frontend_port_name             = "http-port"
    protocol                       = "Http"
  }
  
  request_routing_rule {
    name                       = "routing-rule"
    rule_type                  = "Basic"
    http_listener_name         = "http-listener"
    backend_address_pool_name  = "backend-pool"
    backend_http_settings_name = "http-settings"
    priority                   = 100
  }
  
  tags = var.tags
}

resource "azurerm_public_ip" "main" {
  name                = "${var.project_name}-${var.environment}-pip"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  allocation_method  = "Static"
  sku                = "Standard"
  
  tags = var.tags
}

# 监控
resource "azurerm_application_insights" "main" {
  name                = "${var.project_name}-${var.environment}-appinsights"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  application_type   = "web"
  
  tags = var.tags
}

# 日志分析工作区
resource "azurerm_log_analytics_workspace" "main" {
  name                = "${var.project_name}-${var.environment}-law"
  resource_group_name = azurerm_resource_group.main.name
  location           = azurerm_resource_group.main.location
  sku                = "PerGB2018"
  retention_in_days  = 30
  
  tags = var.tags
}
```

### 3.2 ARM 模板
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "projectName": {
      "type": "string",
      "defaultValue": "myapp",
      "metadata": {
        "description": "Project name"
      }
    },
    "environment": {
      "type": "string",
      "defaultValue": "dev",
      "allowedValues": ["dev", "staging", "prod"],
      "metadata": {
        "description": "Environment"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
      }
    }
  },
  "variables": {
    "resourceGroupName": "[resourceGroup().name]",
    "appServicePlanName": "[concat(parameters('projectName'), '-', parameters('environment'), '-plan')]",
    "webAppName": "[concat(parameters('projectName'), '-', parameters('environment'), '-app')]",
    "sqlServerName": "[concat(parameters('projectName'), '-', parameters('environment'), '-sql')]",
    "databaseName": "[concat(parameters('projectName'), '_', parameters('environment'), '_db')]",
    "redisCacheName": "[concat(parameters('projectName'), '-', parameters('environment'), '-redis')]",
    "appInsightsName": "[concat(parameters('projectName'), '-', parameters('environment'), '-appinsights')]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2021-02-01",
      "name": "[variables('appServicePlanName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "B1",
        "tier": "Basic"
      },
      "kind": "linux",
      "properties": {
        "reserved": true
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2021-02-01",
      "name": "[variables('webAppName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
        "siteConfig": {
          "linuxFxVersion": "DOCKER|myapp:latest",
          "alwaysOn": true,
          "appSettings": [
            {
              "name": "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
              "value": "false"
            },
            {
              "name": "WEBSITES_PORT",
              "value": "80"
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.Sql/servers",
      "apiVersion": "2021-02-01",
      "name": "[variables('sqlServerName')]",
      "location": "[parameters('location')]",
      "properties": {
        "administratorLogin": "sqladmin",
        "administratorLoginPassword": "[newGuid()]",
        "version": "12.0"
      }
    },
    {
      "type": "Microsoft.Sql/servers/databases",
      "apiVersion": "2021-02-01",
      "name": "[concat(variables('sqlServerName'), '/', variables('databaseName'))]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"
      ],
      "properties": {
        "collation": "SQL_Latin1_General_CP1_CI_AS",
        "maxSizeBytes": "2147483648",
        "sku": {
          "name": "Basic",
          "tier": "Basic"
        }
      }
    },
    {
      "type": "Microsoft.Cache/Redis",
      "apiVersion": "2021-06-01",
      "name": "[variables('redisCacheName')]",
      "location": "[parameters('location')]",
      "properties": {
        "enableNonSslPort": false,
        "minimumTlsVersion": "1.2",
        "sku": {
          "name": "Basic",
          "family": "C",
          "capacity": 0
        }
      }
    },
    {
      "type": "Microsoft.Insights/components",
      "apiVersion": "2020-02-02",
      "name": "[variables('appInsightsName')]",
      "location": "[parameters('location')]",
      "properties": {
        "Application_Type": "web",
        "Request_Source": "rest"
      }
    }
  ],
  "outputs": {
    "webAppName": {
      "type": "string",
      "value": "[variables('webAppName')]"
    },
    "webAppUrl": {
      "type": "string",
      "value": "[concat('https://', variables('webAppName'), '.azurewebsites.net')]"
    },
    "sqlServerName": {
      "type": "string",
      "value": "[variables('sqlServerName')]"
    },
    "redisCacheName": {
      "type": "string",
      "value": "[variables('redisCacheName')]"
    },
    "appInsightsName": {
      "type": "string",
      "value": "[variables('appInsightsName')]"
    }
  }
}
```

## 4. 面试重点

### 4.1 高频问题
1. **CI/CD流水线**：流水线设计、自动化测试、部署策略
2. **部署策略**：蓝绿部署、金丝雀部署、滚动更新
3. **基础设施即代码**：Terraform、ARM模板、云资源配置
4. **监控和日志**：应用监控、日志聚合、告警机制
5. **安全性和合规性**：安全扫描、访问控制、合规检查

### 4.2 代码示例准备
- CI/CD流水线配置
- 部署策略实现
- 基础设施配置
- 监控和日志配置
- 安全扫描集成

### 4.3 DevOps要点
- 自动化程度和效率
- 部署策略的选择
- 基础设施的可重复性
- 监控和可观测性
- 安全性和合规性
