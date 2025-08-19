# 架构设计与模式面试指南 🚀

## 📚 快速导航
- [面试高频问题](#面试高频问题)
- [创建型模式](#1-创建型模式-creational-patterns)
- [结构型模式](#2-结构型模式-structural-patterns)
- [行为型模式](#3-行为型模式-behavioral-patterns)
- [架构模式](#4-架构模式-architectural-patterns)
- [实战应用](#5-实战应用与最佳实践)

## ❓ 面试高频问题

### Q1: 单例模式如何实现线程安全？

**面试官想了解什么**：你对多线程编程的理解，以及设计模式的实现能力。

**🎯 标准答案**：

| 实现方式 | 线程安全 | 性能 | 延迟初始化 | 推荐指数 |
|----------|----------|------|------------|----------|
| **双重检查锁定** | ✅ 安全 | 高 | ✅ 支持 | ⭐⭐⭐⭐⭐ |
| **静态构造函数** | ✅ 安全 | 高 | ❌ 不支持 | ⭐⭐⭐⭐ |
| **Lazy<T>** | ✅ 安全 | 中等 | ✅ 支持 | ⭐⭐⭐⭐⭐ |
| **锁机制** | ✅ 安全 | 低 | ✅ 支持 | ⭐⭐⭐ |

**💡 面试加分点**：提到"我会使用Lazy<T>，它提供了线程安全和延迟初始化的最佳平衡"

---

### Q2: 工厂模式和抽象工厂的区别？

**面试官想了解什么**：你对设计模式的理解深度。

**🎯 标准答案**：

| 特性 | **简单工厂** | **工厂方法** | **抽象工厂** |
|------|--------------|--------------|--------------|
| **产品数量** | 多个产品 | 一个产品族 | 多个产品族 |
| **扩展性** | 差（修改工厂类） | 好（新增工厂） | 好（新增工厂） |
| **复杂度** | 简单 | 中等 | 复杂 |
| **使用场景** | 简单应用 | 单一产品族 | 复杂产品族 |

**💡 面试加分点**：提到"我会根据产品族的复杂度和扩展需求来选择工厂模式"

---

### Q3: 观察者模式和发布订阅的区别？

**面试官想了解什么**：你对事件驱动架构的理解。

**🎯 标准答案**：

| 特性 | **观察者模式** | **发布订阅模式** |
|------|----------------|------------------|
| **耦合度** | 高（直接引用） | 低（通过中介） |
| **通知方式** | 推模式 | 推/拉模式 |
| **实现复杂度** | 简单 | 复杂 |
| **扩展性** | 差 | 好 |

**💡 面试加分点**：提到"我会在需要松耦合的场景中使用发布订阅模式"

---

## 🏗️ 实战场景分析

### 场景1：电商订单系统设计

**业务需求**：设计支持多种支付方式的订单处理系统

**🎯 技术方案**：

```
订单创建 → 支付策略选择 → 支付处理 → 订单状态更新
    ↓           ↓            ↓           ↓
  订单工厂    策略模式      工厂模式     观察者模式
```

**核心实现**：
1. **订单工厂**：创建不同类型的订单
2. **策略模式**：选择支付方式
3. **工厂模式**：创建支付处理器
4. **观察者模式**：通知相关系统

**🔑 关键决策**：使用策略模式处理不同的支付方式，提高扩展性

---

### 场景2：日志系统架构设计

**业务需求**：设计支持多种输出目标的日志系统

**🎯 技术方案**：

```
日志记录 → 日志格式化 → 日志输出 → 日志存储
   ↓          ↓           ↓          ↓
  单例模式   策略模式    工厂模式    装饰器模式
```

**核心实现**：
1. **单例模式**：确保日志管理器唯一
2. **策略模式**：选择日志格式
3. **工厂模式**：创建输出目标
4. **装饰器模式**：添加日志功能

---

## 📊 模式分类图表

### 设计模式分类

```
设计模式分类：
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   创建型模式    │    │   结构型模式    │    │   行为型模式    │
│                │    │                │    │                │
│ 单例、工厂     │    │ 适配器、代理     │    │ 观察者、策略     │
│ 建造者、原型   │    │ 装饰器、外观     │    │ 命令、状态       │
│ 对象创建       │    │ 对象组合       │    │ 对象交互       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 模式使用频率统计

| 模式类型 | 使用频率 | 重要性 | 学习优先级 |
|----------|----------|--------|------------|
| **单例模式** | ⭐⭐⭐⭐⭐ | 高 | 最高 |
| **工厂模式** | ⭐⭐⭐⭐⭐ | 高 | 最高 |
| **观察者模式** | ⭐⭐⭐⭐ | 高 | 高 |
| **策略模式** | ⭐⭐⭐⭐ | 高 | 高 |
| **装饰器模式** | ⭐⭐⭐ | 中等 | 中等 |
| **适配器模式** | ⭐⭐⭐ | 中等 | 中等 |

---

## 1. 创建型模式 (Creational Patterns)

### 1.1 单例模式 (Singleton)
```csharp
// 线程安全的单例模式
public sealed class DatabaseConnection
{
    private static readonly Lazy<DatabaseConnection> _instance = 
        new Lazy<DatabaseConnection>(() => new DatabaseConnection());
    
    public static DatabaseConnection Instance => _instance.Value;
    
    private DatabaseConnection() { }
    
    public void Connect()
    {
        Console.WriteLine("Connected to database");
    }
}

// 使用
var connection = DatabaseConnection.Instance;
connection.Connect();

// 依赖注入中的单例
public class SingletonService
{
    private static readonly object _lock = new object();
    private static SingletonService _instance;
    
    public static SingletonService Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new SingletonService();
                    }
                }
            }
            return _instance;
        }
    }
    
    private SingletonService() { }
}
```

### 1.2 工厂模式 (Factory)
```csharp
// 简单工厂
public interface IPaymentMethod
{
    void ProcessPayment(decimal amount);
}

public class CreditCardPayment : IPaymentMethod
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing credit card payment: {amount}");
    }
}

public class PayPalPayment : IPaymentMethod
{
    public void ProcessPayment(decimal amount)
    {
        Console.WriteLine($"Processing PayPal payment: {amount}");
    }
}

public class PaymentFactory
{
    public static IPaymentMethod CreatePaymentMethod(string type)
    {
        return type.ToLower() switch
        {
            "creditcard" => new CreditCardPayment(),
            "paypal" => new PayPalPayment(),
            _ => throw new ArgumentException($"Unknown payment type: {type}")
        };
    }
}

// 使用
var payment = PaymentFactory.CreatePaymentMethod("creditcard");
payment.ProcessPayment(100.00m);

// 抽象工厂
public interface IPaymentFactory
{
    IPaymentMethod CreatePaymentMethod();
    IPaymentValidator CreateValidator();
}

public class CreditCardFactory : IPaymentFactory
{
    public IPaymentMethod CreatePaymentMethod() => new CreditCardPayment();
    public IPaymentValidator CreateValidator() => new CreditCardValidator();
}

public class PayPalFactory : IPaymentFactory
{
    public IPaymentMethod CreatePaymentMethod() => new PayPalPayment();
    public IPaymentValidator CreateValidator() => new PayPalValidator();
}
```

### 1.3 建造者模式 (Builder)
```csharp
public class User
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
    public string Address { get; set; }
    public string Phone { get; set; }
    
    public User() { }
    
    public User(string firstName, string lastName, string email)
    {
        FirstName = firstName;
        LastName = lastName;
        Email = email;
    }
}

public class UserBuilder
{
    private readonly User _user = new User();
    
    public UserBuilder WithName(string firstName, string lastName)
    {
        _user.FirstName = firstName;
        _user.LastName = lastName;
        return this;
    }
    
    public UserBuilder WithEmail(string email)
    {
        _user.Email = email;
        return this;
    }
    
    public UserBuilder WithAge(int age)
    {
        _user.Age = age;
        return this;
    }
    
    public UserBuilder WithAddress(string address)
    {
        _user.Address = address;
        return this;
    }
    
    public UserBuilder WithPhone(string phone)
    {
        _user.Phone = phone;
        return this;
    }
    
    public User Build()
    {
        if (string.IsNullOrEmpty(_user.FirstName) || 
            string.IsNullOrEmpty(_user.LastName) || 
            string.IsNullOrEmpty(_user.Email))
        {
            throw new InvalidOperationException("First name, last name, and email are required");
        }
        
        return _user;
    }
}

// 使用
var user = new UserBuilder()
    .WithName("John", "Doe")
    .WithEmail("john.doe@example.com")
    .WithAge(30)
    .WithAddress("123 Main St")
    .WithPhone("555-1234")
    .Build();
```

### 1.4 原型模式 (Prototype)
```csharp
public interface ICloneable<T>
{
    T Clone();
}

public class Document : ICloneable<Document>
{
    public string Title { get; set; }
    public string Content { get; set; }
    public List<string> Tags { get; set; }
    public DateTime CreatedAt { get; set; }
    
    public Document Clone()
    {
        return new Document
        {
            Title = this.Title,
            Content = this.Content,
            Tags = new List<string>(this.Tags), // 深拷贝
            CreatedAt = this.CreatedAt
        };
    }
    
    // 深拷贝方法
    public Document DeepClone()
    {
        var clone = (Document)this.MemberwiseClone();
        clone.Tags = new List<string>(this.Tags);
        return clone;
    }
}

// 原型注册表
public class PrototypeRegistry
{
    private readonly Dictionary<string, ICloneable<object>> _prototypes = new();
    
    public void Register(string key, ICloneable<object> prototype)
    {
        _prototypes[key] = prototype;
    }
    
    public T Clone<T>(string key) where T : class
    {
        if (_prototypes.TryGetValue(key, out var prototype))
        {
            return (T)prototype.Clone();
        }
        throw new KeyNotFoundException($"Prototype '{key}' not found");
    }
}
```

### 1.5 对象池模式 (Object Pool)
```csharp
public class ConnectionPool
{
    private readonly ConcurrentQueue<IDbConnection> _connections = new();
    private readonly Func<IDbConnection> _connectionFactory;
    private readonly int _maxPoolSize;
    private int _currentPoolSize;
    
    public ConnectionPool(Func<IDbConnection> connectionFactory, int maxPoolSize = 10)
    {
        _connectionFactory = connectionFactory;
        _maxPoolSize = maxPoolSize;
    }
    
    public IDbConnection GetConnection()
    {
        if (_connections.TryDequeue(out var connection))
        {
            return connection;
        }
        
        if (_currentPoolSize < _maxPoolSize)
        {
            Interlocked.Increment(ref _currentPoolSize);
            return _connectionFactory();
        }
        
        // 等待可用连接
        SpinWait.SpinUntil(() => _connections.TryDequeue(out connection));
        return connection;
    }
    
    public void ReturnConnection(IDbConnection connection)
    {
        if (connection != null)
        {
            _connections.Enqueue(connection);
        }
    }
}

// 使用
var pool = new ConnectionPool(() => new SqlConnection("connectionString"));
var connection = pool.GetConnection();
try
{
    // 使用连接
}
finally
{
    pool.ReturnConnection(connection);
}
```

## 2. 结构型模式 (Structural Patterns)

### 2.1 适配器模式 (Adapter)
```csharp
// 目标接口
public interface ILogger
{
    void Log(string message);
    void LogError(string error);
}

// 第三方日志库
public class ThirdPartyLogger
{
    public void WriteLog(string message) { }
    public void WriteError(string error) { }
}

// 适配器
public class ThirdPartyLoggerAdapter : ILogger
{
    private readonly ThirdPartyLogger _logger;
    
    public ThirdPartyLoggerAdapter(ThirdPartyLogger logger)
    {
        _logger = logger;
    }
    
    public void Log(string message)
    {
        _logger.WriteLog(message);
    }
    
    public void LogError(string error)
    {
        _logger.WriteError(error);
    }
}

// 使用
ILogger logger = new ThirdPartyLoggerAdapter(new ThirdPartyLogger());
logger.Log("Application started");
```

### 2.2 桥接模式 (Bridge)
```csharp
public interface IRenderer
{
    void RenderCircle(float radius);
    void RenderRectangle(float width, float height);
}

public class VectorRenderer : IRenderer
{
    public void RenderCircle(float radius)
    {
        Console.WriteLine($"Drawing a circle of radius {radius} using vector graphics");
    }
    
    public void RenderRectangle(float width, float height)
    {
        Console.WriteLine($"Drawing a rectangle of {width}x{height} using vector graphics");
    }
}

public class RasterRenderer : IRenderer
{
    public void RenderCircle(float radius)
    {
        Console.WriteLine($"Drawing a circle of radius {radius} using raster graphics");
    }
    
    public void RenderRectangle(float width, float height)
    {
        Console.WriteLine($"Drawing a rectangle of {width}x{height} using raster graphics");
    }
}

public abstract class Shape
{
    protected IRenderer Renderer;
    
    protected Shape(IRenderer renderer)
    {
        Renderer = renderer;
    }
    
    public abstract void Draw();
}

public class Circle : Shape
{
    private float _radius;
    
    public Circle(IRenderer renderer, float radius) : base(renderer)
    {
        _radius = radius;
    }
    
    public override void Draw()
    {
        Renderer.RenderCircle(_radius);
    }
}

// 使用
IRenderer vectorRenderer = new VectorRenderer();
IRenderer rasterRenderer = new RasterRenderer();

Shape circle1 = new Circle(vectorRenderer, 5);
Shape circle2 = new Circle(rasterRenderer, 5);

circle1.Draw(); // 使用向量图形
circle2.Draw(); // 使用光栅图形
```

### 2.3 组合模式 (Composite)
```csharp
public abstract class FileSystemItem
{
    public string Name { get; set; }
    
    public abstract void Display(int depth = 0);
    public abstract long GetSize();
}

public class File : FileSystemItem
{
    private long _size;
    
    public File(string name, long size)
    {
        Name = name;
        _size = size;
    }
    
    public override void Display(int depth = 0)
    {
        Console.WriteLine($"{new string('-', depth)}File: {Name} ({_size} bytes)");
    }
    
    public override long GetSize()
    {
        return _size;
    }
}

public class Directory : FileSystemItem
{
    private List<FileSystemItem> _children = new List<FileSystemItem>();
    
    public Directory(string name)
    {
        Name = name;
    }
    
    public void Add(FileSystemItem item)
    {
        _children.Add(item);
    }
    
    public void Remove(FileSystemItem item)
    {
        _children.Remove(item);
    }
    
    public override void Display(int depth = 0)
    {
        Console.WriteLine($"{new string('-', depth)}Directory: {Name}");
        foreach (var child in _children)
        {
            child.Display(depth + 2);
        }
    }
    
    public override long GetSize()
    {
        return _children.Sum(child => child.GetSize());
    }
}

// 使用
var root = new Directory("Root");
var documents = new Directory("Documents");
var file1 = new File("file1.txt", 1024);
var file2 = new File("file2.txt", 2048);

documents.Add(file1);
documents.Add(file2);
root.Add(documents);

root.Display();
Console.WriteLine($"Total size: {root.GetSize()} bytes");
```

### 2.4 装饰器模式 (Decorator)
```csharp
public interface ICoffee
{
    string GetDescription();
    double GetCost();
}

public class SimpleCoffee : ICoffee
{
    public string GetDescription() => "Simple coffee";
    public double GetCost() => 1.0;
}

public abstract class CoffeeDecorator : ICoffee
{
    protected ICoffee _coffee;
    
    protected CoffeeDecorator(ICoffee coffee)
    {
        _coffee = coffee;
    }
    
    public virtual string GetDescription() => _coffee.GetDescription();
    public virtual double GetCost() => _coffee.GetCost();
}

public class MilkDecorator : CoffeeDecorator
{
    public MilkDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() => _coffee.GetDescription() + ", milk";
    public override double GetCost() => _coffee.GetCost() + 0.5;
}

public class SugarDecorator : CoffeeDecorator
{
    public SugarDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() => _coffee.GetDescription() + ", sugar";
    public override double GetCost() => _coffee.GetCost() + 0.2;
}

// 使用
ICoffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);

Console.WriteLine($"Description: {coffee.GetDescription()}");
Console.WriteLine($"Cost: ${coffee.GetCost():F2}");
```

### 2.5 外观模式 (Facade)
```csharp
public class SubsystemA
{
    public string OperationA1() => "SubsystemA: Ready!";
    public string OperationA2() => "SubsystemA: Go!";
}

public class SubsystemB
{
    public string OperationB1() => "SubsystemB: Get ready!";
    public string OperationB2() => "SubsystemB: Fire!";
}

public class SubsystemC
{
    public string OperationC1() => "SubsystemC: Action!";
    public string OperationC2() => "SubsystemC: End!";
}

public class Facade
{
    private readonly SubsystemA _subsystemA;
    private readonly SubsystemB _subsystemB;
    private readonly SubsystemC _subsystemC;
    
    public Facade()
    {
        _subsystemA = new SubsystemA();
        _subsystemB = new SubsystemB();
        _subsystemC = new SubsystemC();
    }
    
    public string Operation1()
    {
        var result = "Facade initializes subsystems:\n";
        result += _subsystemA.OperationA1() + "\n";
        result += _subsystemB.OperationB1() + "\n";
        result += _subsystemC.OperationC1();
        return result;
    }
    
    public string Operation2()
    {
        var result = "Facade orders subsystems to perform the action:\n";
        result += _subsystemA.OperationA2() + "\n";
        result += _subsystemB.OperationB2() + "\n";
        result += _subsystemC.OperationC2();
        return result;
    }
}

// 使用
var facade = new Facade();
Console.WriteLine(facade.Operation1());
Console.WriteLine();
Console.WriteLine(facade.Operation2());
```

### 2.6 享元模式 (Flyweight)
```csharp
public class CharacterStyle
{
    public string FontFamily { get; set; }
    public int FontSize { get; set; }
    public bool IsBold { get; set; }
    public bool IsItalic { get; set; }
    
    public override bool Equals(object obj)
    {
        if (obj is CharacterStyle other)
        {
            return FontFamily == other.FontFamily &&
                   FontSize == other.FontSize &&
                   IsBold == other.IsBold &&
                   IsItalic == other.IsItalic;
        }
        return false;
    }
    
    public override int GetHashCode()
    {
        return HashCode.Combine(FontFamily, FontSize, IsBold, IsItalic);
    }
}

public class CharacterStyleFactory
{
    private readonly Dictionary<string, CharacterStyle> _styles = new();
    
    public CharacterStyle GetStyle(string fontFamily, int fontSize, bool isBold, bool isItalic)
    {
        var key = $"{fontFamily}_{fontSize}_{isBold}_{isItalic}";
        
        if (!_styles.TryGetValue(key, out var style))
        {
            style = new CharacterStyle
            {
                FontFamily = fontFamily,
                FontSize = fontSize,
                IsBold = isBold,
                IsItalic = isItalic
            };
            _styles[key] = style;
        }
        
        return style;
    }
    
    public int TotalStylesCreated => _styles.Count;
}

public class Character
{
    public char Symbol { get; set; }
    public CharacterStyle Style { get; set; }
    
    public Character(char symbol, CharacterStyle style)
    {
        Symbol = symbol;
        Style = style;
    }
    
    public void Display()
    {
        Console.WriteLine($"Character '{Symbol}' with style: {Style.FontFamily} {Style.FontSize}");
    }
}

// 使用
var factory = new CharacterStyleFactory();

var style1 = factory.GetStyle("Arial", 12, false, false);
var style2 = factory.GetStyle("Arial", 12, false, false); // 复用相同样式

var char1 = new Character('A', style1);
var char2 = new Character('B', style2);

Console.WriteLine($"Total styles created: {factory.TotalStylesCreated}"); // 应该是1
```

### 2.7 代理模式 (Proxy)
```csharp
public interface IImage
{
    void Display();
}

public class RealImage : IImage
{
    private readonly string _filename;
    
    public RealImage(string filename)
    {
        _filename = filename;
        LoadFromDisk();
    }
    
    private void LoadFromDisk()
    {
        Console.WriteLine($"Loading {_filename}");
    }
    
    public void Display()
    {
        Console.WriteLine($"Displaying {_filename}");
    }
}

public class ProxyImage : IImage
{
    private RealImage _realImage;
    private readonly string _filename;
    
    public ProxyImage(string filename)
    {
        _filename = filename;
    }
    
    public void Display()
    {
        if (_realImage == null)
        {
            _realImage = new RealImage(_filename);
        }
        _realImage.Display();
    }
}

// 使用
IImage image = new ProxyImage("test.jpg");
// 图像还没有加载
image.Display(); // 图像现在加载并显示
image.Display(); // 图像已经加载，直接显示
```

## 3. 行为型模式 (Behavioral Patterns)

### 3.1 责任链模式 (Chain of Responsibility)
```csharp
public abstract class Handler
{
    protected Handler _successor;
    
    public void SetSuccessor(Handler successor)
    {
        _successor = successor;
    }
    
    public abstract void HandleRequest(int request);
}

public class ConcreteHandlerA : Handler
{
    public override void HandleRequest(int request)
    {
        if (request >= 0 && request < 10)
        {
            Console.WriteLine($"{this.GetType().Name} handled request {request}");
        }
        else if (_successor != null)
        {
            _successor.HandleRequest(request);
        }
    }
}

public class ConcreteHandlerB : Handler
{
    public override void HandleRequest(int request)
    {
        if (request >= 10 && request < 20)
        {
            Console.WriteLine($"{this.GetType().Name} handled request {request}");
        }
        else if (_successor != null)
        {
            _successor.HandleRequest(request);
        }
    }
}

public class ConcreteHandlerC : Handler
{
    public override void HandleRequest(int request)
    {
        if (request >= 20 && request < 30)
        {
            Console.WriteLine($"{this.GetType().Name} handled request {request}");
        }
        else if (_successor != null)
        {
            _successor.HandleRequest(request);
        }
        else
        {
            Console.WriteLine($"Request {request} was not handled");
        }
    }
}

// 使用
var handlerA = new ConcreteHandlerA();
var handlerB = new ConcreteHandlerB();
var handlerC = new ConcreteHandlerC();

handlerA.SetSuccessor(handlerB);
handlerB.SetSuccessor(handlerC);

handlerA.HandleRequest(5);   // A处理
handlerA.HandleRequest(15);  // B处理
handlerA.HandleRequest(25);  // C处理
handlerA.HandleRequest(35);  // 无人处理
```

### 3.2 命令模式 (Command)
```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class Light
{
    public void TurnOn() => Console.WriteLine("Light is on");
    public void TurnOff() => Console.WriteLine("Light is off");
}

public class LightOnCommand : ICommand
{
    private readonly Light _light;
    
    public LightOnCommand(Light light)
    {
        _light = light;
    }
    
    public void Execute() => _light.TurnOn();
    public void Undo() => _light.TurnOff();
}

public class LightOffCommand : ICommand
{
    private readonly Light _light;
    
    public LightOffCommand(Light light)
    {
        _light = light;
    }
    
    public void Execute() => _light.TurnOff();
    public void Undo() => _light.TurnOn();
}

public class RemoteControl
{
    private readonly ICommand[] _onCommands;
    private readonly ICommand[] _offCommands;
    private ICommand _undoCommand;
    
    public RemoteControl()
    {
        _onCommands = new ICommand[7];
        _offCommands = new ICommand[7];
        
        var noCommand = new NoCommand();
        for (int i = 0; i < 7; i++)
        {
            _onCommands[i] = noCommand;
            _offCommands[i] = noCommand;
        }
        _undoCommand = noCommand;
    }
    
    public void SetCommand(int slot, ICommand onCommand, ICommand offCommand)
    {
        _onCommands[slot] = onCommand;
        _offCommands[slot] = offCommand;
    }
    
    public void OnButtonWasPushed(int slot)
    {
        _onCommands[slot].Execute();
        _undoCommand = _onCommands[slot];
    }
    
    public void OffButtonWasPushed(int slot)
    {
        _offCommands[slot].Execute();
        _undoCommand = _offCommands[slot];
    }
    
    public void UndoButtonWasPushed()
    {
        _undoCommand.Undo();
    }
}

public class NoCommand : ICommand
{
    public void Execute() { }
    public void Undo() { }
}

// 使用
var light = new Light();
var lightOn = new LightOnCommand(light);
var lightOff = new LightOffCommand(light);

var remote = new RemoteControl();
remote.SetCommand(0, lightOn, lightOff);

remote.OnButtonWasPushed(0);   // 开灯
remote.OffButtonWasPushed(0);  // 关灯
remote.UndoButtonWasPushed();  // 撤销
```

### 3.3 解释器模式 (Interpreter)
```csharp
public abstract class Expression
{
    public abstract int Interpret();
}

public class NumberExpression : Expression
{
    private readonly int _number;
    
    public NumberExpression(int number)
    {
        _number = number;
    }
    
    public override int Interpret()
    {
        return _number;
    }
}

public class AddExpression : Expression
{
    private readonly Expression _left;
    private readonly Expression _right;
    
    public AddExpression(Expression left, Expression right)
    {
        _left = left;
        _right = right;
    }
    
    public override int Interpret()
    {
        return _left.Interpret() + _right.Interpret();
    }
}

public class SubtractExpression : Expression
{
    private readonly Expression _left;
    private readonly Expression _right;
    
    public SubtractExpression(Expression left, Expression right)
    {
        _left = left;
        _right = right;
    }
    
    public override int Interpret()
    {
        return _left.Interpret() - _right.Interpret();
    }
}

// 使用
// 表达式: (5 + 3) - 2
var expression = new SubtractExpression(
    new AddExpression(new NumberExpression(5), new NumberExpression(3)),
    new NumberExpression(2)
);

var result = expression.Interpret(); // 结果: 6
Console.WriteLine($"Result: {result}");
```

### 3.4 迭代器模式 (Iterator)
```csharp
public interface IIterator<T>
{
    bool HasNext();
    T Next();
    void Reset();
}

public interface IIterableCollection<T>
{
    IIterator<T> CreateIterator();
}

public class ListCollection<T> : IIterableCollection<T>
{
    private readonly List<T> _items = new List<T>();
    
    public void Add(T item) => _items.Add(item);
    public void Remove(T item) => _items.Remove(item);
    public int Count => _items.Count;
    
    public IIterator<T> CreateIterator()
    {
        return new ListIterator<T>(this);
    }
    
    public T GetItem(int index) => _items[index];
}

public class ListIterator<T> : IIterator<T>
{
    private readonly ListCollection<T> _collection;
    private int _currentIndex = 0;
    
    public ListIterator(ListCollection<T> collection)
    {
        _collection = collection;
    }
    
    public bool HasNext()
    {
        return _currentIndex < _collection.Count;
    }
    
    public T Next()
    {
        if (HasNext())
        {
            return _collection.GetItem(_currentIndex++);
        }
        throw new InvalidOperationException("No more elements");
    }
    
    public void Reset()
    {
        _currentIndex = 0;
    }
}

// 使用
var collection = new ListCollection<string>();
collection.Add("Item 1");
collection.Add("Item 2");
collection.Add("Item 3");

var iterator = collection.CreateIterator();
while (iterator.HasNext())
{
    Console.WriteLine(iterator.Next());
}
```

### 3.5 中介者模式 (Mediator)
```csharp
public interface IMediator
{
    void Send(string message, Colleague colleague);
}

public abstract class Colleague
{
    protected IMediator _mediator;
    
    public Colleague(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    public abstract void Send(string message);
    public abstract void Receive(string message);
}

public class ConcreteColleagueA : Colleague
{
    public ConcreteColleagueA(IMediator mediator) : base(mediator) { }
    
    public override void Send(string message)
    {
        Console.WriteLine($"Colleague A sends: {message}");
        _mediator.Send(message, this);
    }
    
    public override void Receive(string message)
    {
        Console.WriteLine($"Colleague A receives: {message}");
    }
}

public class ConcreteColleagueB : Colleague
{
    public ConcreteColleagueB(IMediator mediator) : base(mediator) { }
    
    public override void Send(string message)
    {
        Console.WriteLine($"Colleague B sends: {message}");
        _mediator.Send(message, this);
    }
    
    public override void Receive(string message)
    {
        Console.WriteLine($"Colleague B receives: {message}");
    }
}

public class ConcreteMediator : IMediator
{
    private ConcreteColleagueA _colleagueA;
    private ConcreteColleagueB _colleagueB;
    
    public void SetColleagueA(ConcreteColleagueA colleague)
    {
        _colleagueA = colleague;
    }
    
    public void SetColleagueB(ConcreteColleagueB colleague)
    {
        _colleagueB = colleague;
    }
    
    public void Send(string message, Colleague colleague)
    {
        if (colleague == _colleagueA)
        {
            _colleagueB.Receive(message);
        }
        else if (colleague == _colleagueB)
        {
            _colleagueA.Receive(message);
        }
    }
}

// 使用
var mediator = new ConcreteMediator();
var colleagueA = new ConcreteColleagueA(mediator);
var colleagueB = new ConcreteColleagueB(mediator);

mediator.SetColleagueA(colleagueA);
mediator.SetColleagueB(colleagueB);

colleagueA.Send("Hello from A");
colleagueB.Send("Hello from B");
```

### 3.6 备忘录模式 (Memento)
```csharp
public class Memento
{
    public string State { get; }
    
    public Memento(string state)
    {
        State = state;
    }
}

public class Originator
{
    private string _state;
    
    public string State
    {
        get => _state;
        set
        {
            _state = value;
            Console.WriteLine($"State set to: {_state}");
        }
    }
    
    public Memento CreateMemento()
    {
        return new Memento(_state);
    }
    
    public void RestoreMemento(Memento memento)
    {
        _state = memento.State;
        Console.WriteLine($"State restored to: {_state}");
    }
}

public class Caretaker
{
    private readonly List<Memento> _mementos = new List<Memento>();
    
    public void AddMemento(Memento memento)
    {
        _mementos.Add(memento);
    }
    
    public Memento GetMemento(int index)
    {
        return _mementos[index];
    }
}

// 使用
var originator = new Originator();
var caretaker = new Caretaker();

originator.State = "State 1";
caretaker.AddMemento(originator.CreateMemento());

originator.State = "State 2";
caretaker.AddMemento(originator.CreateMemento());

originator.State = "State 3";

// 恢复到第一个状态
originator.RestoreMemento(caretaker.GetMemento(0));
```

### 3.7 观察者模式 (Observer)
```csharp
public interface IObserver
{
    void Update(string message);
}

public interface ISubject
{
    void Attach(IObserver observer);
    void Detach(IObserver observer);
    void Notify(string message);
}

public class Subject : ISubject
{
    private readonly List<IObserver> _observers = new List<IObserver>();
    
    public void Attach(IObserver observer)
    {
        _observers.Add(observer);
    }
    
    public void Detach(IObserver observer)
    {
        _observers.Remove(observer);
    }
    
    public void Notify(string message)
    {
        foreach (var observer in _observers)
        {
            observer.Update(message);
        }
    }
    
    public void DoSomething()
    {
        Console.WriteLine("Subject is doing something...");
        Notify("Something happened!");
    }
}

public class ConcreteObserverA : IObserver
{
    public void Update(string message)
    {
        Console.WriteLine($"Observer A received: {message}");
    }
}

public class ConcreteObserverB : IObserver
{
    public void Update(string message)
    {
        Console.WriteLine($"Observer B received: {message}");
    }
}

// 使用
var subject = new Subject();
var observerA = new ConcreteObserverA();
var observerB = new ConcreteObserverB();

subject.Attach(observerA);
subject.Attach(observerB);

subject.DoSomething();

subject.Detach(observerA);
subject.DoSomething();
```

### 3.8 状态模式 (State)
```csharp
public interface IState
{
    void Handle(Context context);
}

public class Context
{
    private IState _state;
    
    public Context(IState state)
    {
        _state = state;
    }
    
    public IState State
    {
        get => _state;
        set
        {
            _state = value;
            Console.WriteLine($"State changed to: {_state.GetType().Name}");
        }
    }
    
    public void Request()
    {
        _state.Handle(this);
    }
}

public class ConcreteStateA : IState
{
    public void Handle(Context context)
    {
        Console.WriteLine("Handling request in State A");
        context.State = new ConcreteStateB();
    }
}

public class ConcreteStateB : IState
{
    public void Handle(Context context)
    {
        Console.WriteLine("Handling request in State B");
        context.State = new ConcreteStateC();
    }
}

public class ConcreteStateC : IState
{
    public void Handle(Context context)
    {
        Console.WriteLine("Handling request in State C");
        context.State = new ConcreteStateA();
    }
}

// 使用
var context = new Context(new ConcreteStateA());

context.Request(); // A -> B
context.Request(); // B -> C
context.Request(); // C -> A
```

### 3.9 策略模式 (Strategy)
```csharp
public interface IPaymentStrategy
{
    void Pay(decimal amount);
}

public class CreditCardPayment : IPaymentStrategy
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Paid ${amount} using Credit Card");
    }
}

public class PayPalPayment : IPaymentStrategy
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Paid ${amount} using PayPal");
    }
}

public class CashPayment : IPaymentStrategy
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Paid ${amount} using Cash");
    }
}

public class ShoppingCart
{
    private readonly List<decimal> _items = new List<decimal>();
    private IPaymentStrategy _paymentStrategy;
    
    public void AddItem(decimal price)
    {
        _items.Add(price);
    }
    
    public void SetPaymentStrategy(IPaymentStrategy strategy)
    {
        _paymentStrategy = strategy;
    }
    
    public void Checkout()
    {
        var total = _items.Sum();
        _paymentStrategy?.Pay(total);
        _items.Clear();
    }
}

// 使用
var cart = new ShoppingCart();
cart.AddItem(10.50m);
cart.AddItem(25.00m);

cart.SetPaymentStrategy(new CreditCardPayment());
cart.Checkout();

cart.AddItem(15.75m);
cart.SetPaymentStrategy(new PayPalPayment());
cart.Checkout();
```

### 3.10 模板方法模式 (Template Method)
```csharp
public abstract class DataProcessor
{
    public void ProcessData()
    {
        ReadData();
        ProcessDataCore();
        WriteData();
    }
    
    protected abstract void ReadData();
    protected abstract void ProcessDataCore();
    protected abstract void WriteData();
    
    protected virtual void ValidateData()
    {
        Console.WriteLine("Validating data...");
    }
}

public class CsvDataProcessor : DataProcessor
{
    protected override void ReadData()
    {
        Console.WriteLine("Reading CSV data...");
    }
    
    protected override void ProcessDataCore()
    {
        Console.WriteLine("Processing CSV data...");
        ValidateData();
    }
    
    protected override void WriteData()
    {
        Console.WriteLine("Writing CSV data...");
    }
}

public class JsonDataProcessor : DataProcessor
{
    protected override void ReadData()
    {
        Console.WriteLine("Reading JSON data...");
    }
    
    protected override void ProcessDataCore()
    {
        Console.WriteLine("Processing JSON data...");
        ValidateData();
    }
    
    protected override void WriteData()
    {
        Console.WriteLine("Writing JSON data...");
    }
}

// 使用
DataProcessor csvProcessor = new CsvDataProcessor();
csvProcessor.ProcessData();

Console.WriteLine();

DataProcessor jsonProcessor = new JsonDataProcessor();
jsonProcessor.ProcessData();
```

### 3.11 访问者模式 (Visitor)
```csharp
public interface IVisitor
{
    void Visit(ConcreteElementA element);
    void Visit(ConcreteElementB element);
}

public interface IElement
{
    void Accept(IVisitor visitor);
}

public class ConcreteElementA : IElement
{
    public string OperationA()
    {
        return "ConcreteElementA";
    }
    
    public void Accept(IVisitor visitor)
    {
        visitor.Visit(this);
    }
}

public class ConcreteElementB : IElement
{
    public string OperationB()
    {
        return "ConcreteElementB";
    }
    
    public void Accept(IVisitor visitor)
    {
        visitor.Visit(this);
    }
}

public class ConcreteVisitor1 : IVisitor
{
    public void Visit(ConcreteElementA element)
    {
        Console.WriteLine($"Visitor1: {element.OperationA()}");
    }
    
    public void Visit(ConcreteElementB element)
    {
        Console.WriteLine($"Visitor1: {element.OperationB()}");
    }
}

public class ConcreteVisitor2 : IVisitor
{
    public void Visit(ConcreteElementA element)
    {
        Console.WriteLine($"Visitor2: {element.OperationA()}");
    }
    
    public void Visit(ConcreteElementB element)
    {
        Console.WriteLine($"Visitor2: {element.OperationB()}");
    }
}

public class ObjectStructure
{
    private readonly List<IElement> _elements = new List<IElement>();
    
    public void Attach(IElement element)
    {
        _elements.Add(element);
    }
    
    public void Detach(IElement element)
    {
        _elements.Remove(element);
    }
    
    public void Accept(IVisitor visitor)
    {
        foreach (var element in _elements)
        {
            element.Accept(visitor);
        }
    }
}

// 使用
var objectStructure = new ObjectStructure();
objectStructure.Attach(new ConcreteElementA());
objectStructure.Attach(new ConcreteElementB());

var visitor1 = new ConcreteVisitor1();
var visitor2 = new ConcreteVisitor2();

objectStructure.Accept(visitor1);
Console.WriteLine();
objectStructure.Accept(visitor2);
```

## 4. 面试重点

### 4.1 高频问题
1. **单例模式**：线程安全、懒加载、依赖注入
2. **工厂模式**：简单工厂、工厂方法、抽象工厂的选择
3. **建造者模式**：链式调用、参数验证
4. **装饰器模式**：动态扩展功能、组合优于继承
5. **观察者模式**：事件驱动、松耦合设计

### 4.2 代码示例准备
- 线程安全单例的实现
- 工厂模式的实际应用
- 装饰器模式的组合使用
- 观察者模式的事件处理
- 策略模式的算法切换

### 4.3 设计原则
- **SOLID原则**：单一职责、开闭原则、里氏替换、接口隔离、依赖倒置
- **DRY原则**：不要重复自己
- **KISS原则**：保持简单
- **组合优于继承**：优先使用组合和委托
- **针对接口编程**：依赖抽象而非具体实现
