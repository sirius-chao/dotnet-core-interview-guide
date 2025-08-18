# Web 开发技术概览

## 📚 目录导航

- [Web API 设计与实现](./web-api-design.md)
- [实时通信与 SignalR](./signalr-realtime.md)
- [Blazor 全栈开发](./blazor-development.md)

## 🎯 学习目标

本模块旨在帮助开发者掌握 .NET Core Web 开发的核心技术，包括：

- **Web API 设计**: 掌握 RESTful API 的设计原则和最佳实践
- **前端集成**: 了解现代前端技术与 .NET Core 的集成
- **实时通信**: 掌握 SignalR 的实时通信机制
- **全栈开发**: 使用 Blazor 进行前后端统一开发

## 🚀 快速开始

### 1. 环境准备
```bash
# 安装 .NET 8 SDK
dotnet --version

# 创建新的 Web 项目
dotnet new web -n MyWebApp
cd MyWebApp

# 运行项目
dotnet run
```

### 2. 项目结构
```
MyWebApp/
├── Controllers/          # 控制器
├── Models/              # 数据模型
├── Views/               # 视图
├── wwwroot/             # 静态文件
├── Program.cs           # 应用程序入口
└── appsettings.json     # 配置文件
```

## 📖 核心概念

### MVC 架构模式
- **Model**: 数据模型和业务逻辑
- **View**: 用户界面和展示层
- **Controller**: 请求处理和流程控制

### 中间件管道
- 请求处理流程
- 中间件顺序和配置
- 自定义中间件开发

### 依赖注入
- 服务生命周期管理
- 服务注册和配置
- 构造函数注入最佳实践

## 🔧 常用工具和库

### 开发工具
- **Visual Studio**: 完整的 IDE 环境
- **VS Code**: 轻量级编辑器
- **Rider**: JetBrains 的 .NET IDE

### 常用 NuGet 包
- **Swashbuckle**: API 文档生成
- **AutoMapper**: 对象映射
- **FluentValidation**: 数据验证
- **Serilog**: 结构化日志

### 前端工具
- **npm/yarn**: 包管理器
- **Webpack/Vite**: 构建工具
- **TypeScript**: 类型安全的 JavaScript

## 📚 学习路径

### 初级 → 中级
1. **基础 MVC**: 控制器、视图、模型
2. **Web API**: RESTful 设计、数据序列化
3. **中间件**: 管道机制、自定义中间件
4. **依赖注入**: 服务注册、生命周期管理

### 中级 → 高级
1. **高级 MVC**: 过滤器、模型绑定、验证
2. **性能优化**: 缓存策略、异步编程
3. **安全实践**: 认证授权、数据保护
4. **测试策略**: 单元测试、集成测试

### 高级 → 专家
1. **架构设计**: 微服务、CQRS、事件驱动
2. **实时通信**: SignalR、WebSocket
3. **全栈开发**: Blazor、WebAssembly
4. **云原生**: 容器化、Kubernetes

## 💡 最佳实践

### 代码组织
- 使用领域驱动设计 (DDD) 组织代码
- 遵循 SOLID 原则
- 实现适当的异常处理策略

### 性能优化
- 使用异步编程模式
- 实现适当的缓存策略
- 优化数据库查询

### 安全考虑
- 实现适当的认证和授权
- 防止常见的安全漏洞
- 保护敏感数据

## 🔍 常见问题

### Q: 如何选择 MVC 还是 Web API？
A: MVC 适用于传统的 Web 应用，Web API 适用于前后端分离的架构。

### Q: 什么时候使用 SignalR？
A: 当需要实时双向通信时，如聊天应用、实时通知、协作工具等。

### Q: Blazor 适合什么场景？
A: 适合需要快速开发原型、团队主要使用 C# 技能、需要前后端代码共享的场景。

## 📖 推荐资源

### 官方文档
- [ASP.NET Core 官方文档](https://docs.microsoft.com/aspnet/core/)
- [.NET 官方文档](https://docs.microsoft.com/dotnet/)

### 社区资源
- [Stack Overflow](https://stackoverflow.com/questions/tagged/asp.net-core)
- [GitHub 示例项目](https://github.com/aspnet/AspNetCore)
- [.NET 社区](https://dotnet.microsoft.com/community)

### 书籍推荐
- 《ASP.NET Core in Action》
- 《Building Web APIs with ASP.NET Core》
- 《Blazor in Action》

## 🎯 下一步

选择你感兴趣的主题开始学习：

1. **[Web API 设计](./web-api-design.md)** - 掌握 RESTful API 设计原则
2. **[实时通信](./signalr-realtime.md)** - 学习 SignalR 实时通信
3. **[Blazor 开发](./blazor-development.md)** - 探索全栈开发新方式
