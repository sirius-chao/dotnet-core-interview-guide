# CI/CD 与部署

## 1. 持续集成 (CI)

### 1.1 自动化构建

#### MSBuild 配置
```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors />
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
  </PropertyGroup>
  
  <PropertyGroup Condition="'$(Configuration)'=='Release'">
    <Optimize>true</Optimize>
    <DebugType>none</DebugType>
    <DebugSymbols>false</DebugSymbols>
  </PropertyGroup>
</Project>
```

#### 构建脚本
```bash
#!/bin/bash
# build.sh

echo "开始构建 .NET 应用..."

# 清理
dotnet clean

# 还原包
dotnet restore

# 构建
dotnet build --configuration Release --no-restore

# 运行测试
dotnet test --configuration Release --no-build --verbosity normal

# 发布
dotnet publish --configuration Release --output ./publish --no-build

echo "构建完成！"
```

### 1.2 代码质量检查

#### 静态代码分析
```xml
<!-- Directory.Build.props -->
<Project>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="8.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.507">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>
  
  <ItemGroup>
    <AdditionalFiles Include="$(MSBuildThisFileDirectory)stylecop.json" />
  </ItemGroup>
</Project>
```

#### SonarQube 集成
```yaml
# .github/workflows/sonar.yml
name: SonarQube Analysis
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
    
    - name: Install SonarQube Scanner
      run: |
        dotnet tool install --global dotnet-sonarscanner
    
    - name: SonarQube Begin
      run: |
        dotnet sonarscanner begin \
          /k:"your-project-key" \
          /d:sonar.host.url="${{ secrets.SONAR_HOST_URL }}" \
          /d:sonar.login="${{ secrets.SONAR_TOKEN }}" \
          /d:sonar.cs.opencover.reportsPaths="**/coverage.opencover.xml"
    
    - name: Build and Test
      run: |
        dotnet restore
        dotnet build --no-restore
        dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage"
    
    - name: SonarQube End
      run: |
        dotnet sonarscanner end /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
```

## 2. 持续部署 (CD)

### 2.1 部署策略

#### 蓝绿部署
```csharp
public class BlueGreenDeploymentService
{
    private readonly ILogger<BlueGreenDeploymentService> _logger;
    private readonly IDeploymentClient _deploymentClient;
    
    public BlueGreenDeploymentService(
        ILogger<BlueGreenDeploymentService> logger,
        IDeploymentClient deploymentClient)
    {
        _logger = logger;
        _deploymentClient = deploymentClient;
    }
    
    public async Task DeployAsync(string version, string environment)
    {
        try
        {
            _logger.LogInformation("开始蓝绿部署版本 {Version} 到环境 {Environment}", version, environment);
            
            // 1. 部署到绿色环境
            var greenDeployment = await DeployToGreenEnvironmentAsync(version, environment);
            
            // 2. 健康检查
            if (await IsEnvironmentHealthyAsync(greenDeployment))
            {
                // 3. 切换流量
                await SwitchTrafficAsync(greenDeployment);
                
                // 4. 清理蓝色环境
                await CleanupBlueEnvironmentAsync(environment);
                
                _logger.LogInformation("蓝绿部署成功完成");
            }
            else
            {
                // 回滚
                await RollbackDeploymentAsync(environment);
                throw new DeploymentException("部署后健康检查失败");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "蓝绿部署失败");
            throw;
        }
    }
    
    private async Task<DeploymentInfo> DeployToGreenEnvironmentAsync(string version, string environment)
    {
        var deployment = new DeploymentInfo
        {
            Version = version,
            Environment = $"{environment}-green",
            Timestamp = DateTime.UtcNow
        };
        
        await _deploymentClient.DeployAsync(deployment);
        return deployment;
    }
    
    private async Task<bool> IsEnvironmentHealthyAsync(DeploymentInfo deployment)
    {
        // 等待服务启动
        await Task.Delay(TimeSpan.FromSeconds(30));
        
        // 执行健康检查
        var healthChecks = new[]
        {
            CheckHealthEndpointAsync(deployment),
            CheckDatabaseConnectionAsync(deployment),
            CheckExternalDependenciesAsync(deployment)
        };
        
        var results = await Task.WhenAll(healthChecks);
        return results.All(r => r);
    }
}
```

#### 金丝雀发布
```csharp
public class CanaryDeploymentService
{
    private readonly ILoadBalancer _loadBalancer;
    private readonly IHealthMonitor _healthMonitor;
    
    public async Task DeployCanaryAsync(string version, int initialPercentage)
    {
        // 1. 部署金丝雀版本
        var canaryDeployment = await DeployCanaryVersionAsync(version);
        
        // 2. 逐步增加流量
        var percentages = new[] { initialPercentage, 25, 50, 75, 100 };
        
        foreach (var percentage in percentages)
        {
            await UpdateTrafficSplitAsync(canaryDeployment, percentage);
            
            // 监控一段时间
            var isHealthy = await MonitorHealthAsync(canaryDeployment, TimeSpan.FromMinutes(5));
            
            if (!isHealthy)
            {
                await RollbackCanaryAsync(canaryDeployment);
                throw new CanaryDeploymentException($"金丝雀发布在 {percentage}% 流量时失败");
            }
            
            // 等待稳定
            await Task.Delay(TimeSpan.FromMinutes(10));
        }
        
        // 3. 完全切换到新版本
        await CompleteCanaryDeploymentAsync(canaryDeployment);
    }
}
```

### 2.2 环境管理

#### 环境配置
```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp_Dev;Trusted_Connection=true;"
  },
  "FeatureFlags": {
    "NewFeature": true,
    "ExperimentalFeature": false
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Error"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=MyApp_Prod;User Id=appuser;Password=***;"
  },
  "FeatureFlags": {
    "NewFeature": false,
    "ExperimentalFeature": false
  }
}
```

#### 环境变量管理
```csharp
public class EnvironmentConfiguration
{
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                var env = context.HostingEnvironment.EnvironmentName;
                
                // 基础配置
                config.SetBasePath(Directory.GetCurrentDirectory())
                      .AddJsonFile("appsettings.json", optional: false)
                      .AddJsonFile($"appsettings.{env}.json", optional: true)
                      .AddEnvironmentVariables()
                      .AddUserSecrets<Program>(optional: true);
                
                // 云配置
                if (env == "Production")
                {
                    config.AddAzureKeyVault(
                        new Uri(Environment.GetEnvironmentVariable("KEYVAULT_URL")),
                        new DefaultAzureCredential());
                }
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## 3. 基础设施即代码 (IaC)

### 3.1 Azure ARM 模板

#### 基础资源模板
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "type": "string",
      "defaultValue": "myapp",
      "metadata": {
        "description": "应用名称"
      }
    },
    "environment": {
      "type": "string",
      "allowedValues": ["dev", "staging", "prod"],
      "defaultValue": "dev",
      "metadata": {
        "description": "环境"
      }
    }
  },
  "variables": {
    "appServicePlanName": "[concat(parameters('appName'), '-plan-', parameters('environment'))]",
    "webAppName": "[concat(parameters('appName'), '-web-', parameters('environment'))]",
    "sqlServerName": "[concat(parameters('appName'), '-sql-', parameters('environment'))]",
    "databaseName": "[concat(parameters('appName'), '-db-', parameters('environment'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2021-02-01",
      "name": "[variables('appServicePlanName')]",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "B1",
        "tier": "Basic"
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2021-02-01",
      "name": "[variables('webAppName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
        "siteConfig": {
          "netFrameworkVersion": "v4.0",
          "appSettings": [
            {
              "name": "WEBSITE_RUN_FROM_PACKAGE",
              "value": "1"
            },
            {
              "name": "ASPNETCORE_ENVIRONMENT",
              "value": "[parameters('environment')]"
            }
          ]
        }
      }
    }
  ]
}
```

### 3.2 Terraform 配置

#### 基础设施定义
```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_app_service_plan" "plan" {
  name                = "${var.app_name}-plan"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  kind                = "Windows"
  
  sku {
    tier = "Standard"
    size = "S1"
  }
}

resource "azurerm_app_service" "webapp" {
  name                = "${var.app_name}-web"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  app_service_plan_id = azurerm_app_service_plan.plan.id
  
  app_settings = {
    "WEBSITE_RUN_FROM_PACKAGE" = "1"
    "ASPNETCORE_ENVIRONMENT"   = var.environment
  }
  
  site_config {
    dotnet_framework_version = "v4.0"
  }
}

# variables.tf
variable "resource_group_name" {
  description = "资源组名称"
  type        = string
}

variable "location" {
  description = "Azure 区域"
  type        = string
  default     = "East US"
}

variable "app_name" {
  description = "应用名称"
  type        = string
}

variable "environment" {
  description = "环境"
  type        = string
  default     = "dev"
}
```

## 4. 部署自动化

### 4.1 GitHub Actions

#### 完整部署流水线
```yaml
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '8.0.x'
  AZURE_WEBAPP_NAME: myapp-web
  AZURE_WEBAPP_PACKAGE_PATH: './publish'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Run tests
      run: dotnet test --configuration Release --no-build --verbosity normal
    
    - name: Publish
      run: dotnet publish --configuration Release --output ${{ env.AZURE_WEBAPP_PACKAGE_PATH }} --no-build
    
    - name: Upload artifact
      uses: actions/upload-artifact@v3
      with:
        name: webapp
        path: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
  
  deploy-dev:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: development
    steps:
    - uses: actions/checkout@v3
    
    - name: Download artifact
      uses: actions/download-artifact@v3
      with:
        name: webapp
        path: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
    
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}-dev
        package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_DEV }}
  
  deploy-prod:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
    - uses: actions/checkout@v3
    
    - name: Download artifact
      uses: actions/download-artifact@v3
      with:
        name: webapp
        path: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
    
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}-prod
        package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_PROD }}
    
    - name: Run smoke tests
      run: |
        # 等待部署完成
        sleep 60
        # 执行冒烟测试
        curl -f https://${{ env.AZURE_WEBAPP_NAME }}-prod.azurewebsites.net/health
```

### 4.2 Azure DevOps

#### 发布管道
```yaml
# azure-pipelines.yml
trigger:
- main
- develop

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'
  dotNetVersion: '8.0.x'

stages:
- stage: Build
  displayName: '构建和测试'
  jobs:
  - job: Build
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      inputs:
        version: '$(dotNetVersion)'
    
    - task: DotNetCoreCLI@2
      displayName: '还原包'
      inputs:
        command: 'restore'
        projects: '$(solution)'
    
    - task: DotNetCoreCLI@2
      displayName: '构建'
      inputs:
        command: 'build'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration) --no-restore'
    
    - task: DotNetCoreCLI@2
      displayName: '测试'
      inputs:
        command: 'test'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
    
    - task: DotNetCoreCLI@2
      displayName: '发布'
      inputs:
        command: 'publish'
        projects: '$(solution)'
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory) --no-build'
    
    - task: PublishBuildArtifacts@1
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
        publishLocation: 'Container'

- stage: DeployDev
  displayName: '部署到开发环境'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  jobs:
  - deployment: Deploy
    pool:
      vmImage: 'ubuntu-latest'
    environment: 'development'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadBuildArtifacts@1
            inputs:
              buildType: 'current'
              artifactName: 'drop'
              downloadPath: '$(System.ArtifactsDirectory)'
          
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure Subscription'
              appName: 'myapp-web-dev'
              package: '$(System.ArtifactsDirectory)/**/*.zip'

- stage: DeployProd
  displayName: '部署到生产环境'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: Deploy
    pool:
      vmImage: 'ubuntu-latest'
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadBuildArtifacts@1
            inputs:
              buildType: 'current'
              artifactName: 'drop'
              downloadPath: '$(System.ArtifactsDirectory)'
          
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure Subscription'
              appName: 'myapp-web-prod'
              package: '$(System.ArtifactsDirectory)/**/*.zip'
```

## 5. 监控与回滚

### 5.1 部署监控

#### 健康检查
```csharp
public class DeploymentHealthMonitor
{
    private readonly ILogger<DeploymentHealthMonitor> _logger;
    private readonly IHealthCheckService _healthCheckService;
    
    public async Task<bool> MonitorDeploymentAsync(string deploymentId, TimeSpan timeout)
    {
        var startTime = DateTime.UtcNow;
        var checkInterval = TimeSpan.FromSeconds(10);
        
        while (DateTime.UtcNow - startTime < timeout)
        {
            try
            {
                var healthStatus = await _healthCheckService.CheckHealthAsync();
                
                if (healthStatus.Status == HealthStatus.Healthy)
                {
                    _logger.LogInformation("部署 {DeploymentId} 健康检查通过", deploymentId);
                    return true;
                }
                
                _logger.LogWarning("部署 {DeploymentId} 健康检查失败，状态: {Status}", 
                    deploymentId, healthStatus.Status);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "部署 {DeploymentId} 健康检查异常", deploymentId);
            }
            
            await Task.Delay(checkInterval);
        }
        
        _logger.LogError("部署 {DeploymentId} 健康检查超时", deploymentId);
        return false;
    }
}
```

### 5.2 自动回滚

#### 回滚策略
```csharp
public class RollbackService
{
    private readonly IDeploymentClient _deploymentClient;
    private readonly ILogger<RollbackService> _logger;
    
    public async Task RollbackAsync(string environment, string targetVersion)
    {
        try
        {
            _logger.LogInformation("开始回滚环境 {Environment} 到版本 {Version}", 
                environment, targetVersion);
            
            // 1. 停止当前部署
            await StopCurrentDeploymentAsync(environment);
            
            // 2. 部署目标版本
            await DeployVersionAsync(environment, targetVersion);
            
            // 3. 验证回滚成功
            var isHealthy = await ValidateRollbackAsync(environment);
            
            if (isHealthy)
            {
                _logger.LogInformation("回滚成功完成");
            }
            else
            {
                throw new RollbackException("回滚后验证失败");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "回滚失败");
            throw;
        }
    }
    
    private async Task StopCurrentDeploymentAsync(string environment)
    {
        var currentDeployment = await _deploymentClient.GetCurrentDeploymentAsync(environment);
        if (currentDeployment != null)
        {
            await _deploymentClient.StopDeploymentAsync(currentDeployment.Id);
        }
    }
}
```

## 6. 面试重点

### 6.1 CI/CD 流程
- **构建自动化**: MSBuild 配置、多环境构建
- **测试集成**: 单元测试、集成测试、自动化测试
- **质量门禁**: 代码覆盖率、静态分析、安全扫描

### 6.2 部署策略
- **蓝绿部署**: 零停机部署、快速回滚
- **金丝雀发布**: 渐进式发布、风险控制
- **滚动更新**: 分批部署、健康检查

### 6.3 基础设施管理
- **IaC 工具**: ARM 模板、Terraform、Bicep
- **环境管理**: 配置管理、环境隔离、资源命名
- **云服务集成**: Azure DevOps、GitHub Actions

### 6.4 运维能力
- **监控告警**: 部署监控、性能监控、错误追踪
- **自动化运维**: 自动扩缩容、故障自愈、备份恢复
- **安全合规**: 访问控制、审计日志、合规检查
