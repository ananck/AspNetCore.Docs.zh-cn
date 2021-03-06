---
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---
# <a name="dependency-injection-in-aspnet-core"></a>在 ASP.NET Core 依赖注入

作者：[Steve Smith](https://ardalis.com/) 和 [Scott Addie](https://scottaddie.com)

::: moniker range=">= aspnetcore-3.0"

ASP.NET Core 支持依赖关系注入 (DI) 软件设计模式，这是一种在类及其依赖关系之间实现[控制反转 (IoC)](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#dependency-inversion) 的技术。

有关特定于 MVC 控制器中依赖关系注入的详细信息，请参阅 <xref:mvc/controllers/dependency-injection>。

[查看或下载示例代码](https://github.com/dotnet/AspNetCore.Docs/tree/master/aspnetcore/fundamentals/dependency-injection/samples)（[如何下载](xref:index#how-to-download-a-sample)）

## <a name="overview-of-dependency-injection"></a>依赖关系注入概述

依赖项是另一个对象所需的任何对象。 使用应用中其他类所依赖的 `WriteMessage` 方法检查以下 `MyDependency` 类：

```csharp
public class MyDependency
{
    public MyDependency()
    {
    }

    public Task WriteMessage(string message)
    {
        Console.WriteLine(
            $"MyDependency.WriteMessage called. Message: {message}");

        return Task.FromResult(0);
    }
}
```

可以创建 `MyDependency` 类的实例以使 `WriteMessage` 方法可用于类。 `MyDependency` 类是 `IndexModel` 类的依赖项：

```csharp
public class IndexModel : PageModel
{
    MyDependency _dependency = new MyDependency();

    public async Task OnGetAsync()
    {
        await _dependency.WriteMessage(
            "IndexModel.OnGetAsync created this message.");
    }
}
```

该类创建并直接依赖于 `MyDependency` 实例。 代码依赖关系（如前面的示例）存在问题，应该避免使用，原因如下：

* 要用不同的实现替换 `MyDependency`，必须修改类。
* 如果 `MyDependency` 具有依赖关系，则必须由类对其进行配置。 在具有多个依赖于 `MyDependency` 的类的大型项目中，配置代码在整个应用中会变得分散。
* 这种实现很难进行单元测试。 应用应使用模拟或存根 `MyDependency` 类，该类不能使用此方法。

依赖关系注入通过以下方式解决了这些问题：

* 使用接口或基类抽象化依赖关系实现。
* 注册服务容器中的依赖关系。 ASP.NET Core 提供了一个内置的服务容器 <xref:System.IServiceProvider>。 服务已在应用的 `Startup.ConfigureServices` 方法中注册。
* 将服务注入到使用它的类的构造函数中。 框架负责创建依赖关系的实例，并在不再需要时对其进行处理。

在[示例应用](https://github.com/dotnet/AspNetCore.Docs/tree/master/aspnetcore/fundamentals/dependency-injection/samples)中，`IMyDependency` 接口定义了服务为应用提供的方法：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Interfaces/IMyDependency.cs?name=snippet1)]

此接口由具体类型 `MyDependency` 实现：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Services/MyDependency.cs?name=snippet1)]

`MyDependency` 在其构造函数中请求一个 <xref:Microsoft.Extensions.Logging.ILogger`1>。 以链式方式使用依赖关系注入并不罕见。 每个请求的依赖关系相应地请求其自己的依赖关系。 容器解析图中的依赖关系并返回完全解析的服务。 必须被解析的依赖关系的集合通常被称为“依赖关系树”、“依赖关系图”或“对象图”。

必须在服务容器中注册 `IMyDependency` 和 `ILogger<TCategoryName>`。 `IMyDependency` 已在 `Startup.ConfigureServices` 中注册。 `ILogger<TCategoryName>` 由日志记录抽象基础结构注册，因此它是框架默认注册的[框架提供的服务](#framework-provided-services)。

容器通过利用[（泛型）开放类型](/dotnet/csharp/language-reference/language-specification/types#open-and-closed-types)解析 `ILogger<TCategoryName>`，而无需注册每个[（泛型）构造类型](/dotnet/csharp/language-reference/language-specification/types#constructed-types)：

```csharp
services.AddSingleton(typeof(ILogger<>), typeof(Logger<>));
```

在示例应用中，使用具体类型 `MyDependency` 注册 `IMyDependency` 服务。 注册将服务生存期的范围限定为单个请求的生存期。 本主题后面将介绍[服务生存期](#service-lifetimes)。

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Startup.cs?name=snippet1&highlight=5)]

> [!NOTE]
> 每个 `services.Add{SERVICE_NAME}` 扩展方法添加（并可能配置）服务。 例如，`services.AddMvc()` 添加 Razor Pages 和 MVC 需要的服务。 我们建议应用遵循此约定。 将扩展方法置于 [Microsoft.Extensions.DependencyInjection](/dotnet/api/microsoft.extensions.dependencyinjection) 命名空间中以封装服务注册的组。

如果服务的构造函数需要[内置类型](/dotnet/csharp/language-reference/keywords/built-in-types-table)（如 `string`），则可以使用[配置](xref:fundamentals/configuration/index)或[选项模式](xref:fundamentals/configuration/options)注入该类型：

```csharp
public class MyDependency : IMyDependency
{
    public MyDependency(IConfiguration config)
    {
        var myStringValue = config["MyStringKey"];

        // Use myStringValue
    }

    ...
}
```

通过使用服务并分配给私有字段的类的构造函数请求服务的实例。 该字段用于在整个类中根据需要访问服务。

在示例应用中，请求 `IMyDependency` 实例并用于调用服务的 `WriteMessage` 方法：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Pages/Index.cshtml.cs?name=snippet1&highlight=3,6,13,29-30)]

## <a name="services-injected-into-startup"></a>注入启动的服务

使用泛型主机 (<xref:Microsoft.Extensions.Hosting.IHostBuilder>) 时，只能将以下服务类型注入 `Startup` 构造函数：

* `IWebHostEnvironment`
* <xref:Microsoft.Extensions.Hosting.IHostEnvironment>
* <xref:Microsoft.Extensions.Configuration.IConfiguration>

服务可以注入 `Startup.Configure`：

```csharp
public void Configure(IApplicationBuilder app, IOptions<MyOptions> options)
{
    ...
}
```

有关详细信息，请参阅 <xref:fundamentals/startup>。

## <a name="framework-provided-services"></a>框架提供的服务

`Startup.ConfigureServices` 方法负责定义应用使用的服务，包括 Entity Framework Core 和 ASP.NET Core MVC 等平台功能。 最初，提供给 `ConfigureServices` 的 `IServiceCollection` 具有框架定义的服务（具体取决于[主机配置方式](xref:fundamentals/index#host)）。 基于 ASP.NET Core 模板的应用程序具有框架注册的数百个服务的情况并不少见。 下表列出了框架注册的服务的一个小示例。

| 服务类型 | 生存期 |
| ---
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

------ | --- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---- | | <xref:Microsoft.AspNetCore.Hosting.Builder.IApplicationBuilderFactory?displayProperty=fullName> | 暂时 | | `IHostApplicationLifetime` | 单一实例 | | `IWebHostEnvironment` | 单一实例 | | <xref:Microsoft.AspNetCore.Hosting.IStartup?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Hosting.IStartupFilter?displayProperty=fullName> | 暂时 | | <xref:Microsoft.AspNetCore.Hosting.Server.IServer?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Http.IHttpContextFactory?displayProperty=fullName> | 暂时 | | <xref:Microsoft.Extensions.Logging.ILogger`1?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.Logging.ILoggerFactory?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.ObjectPool.ObjectPoolProvider?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.Options.IConfigureOptions`1?displayProperty=fullName> | 暂时 | | <xref:Microsoft.Extensions.Options.IOptions`1?displayProperty=fullName> | 单一实例 | | <xref:System.Diagnostics.DiagnosticSource?displayProperty=fullName> | 单一实例 | | <xref:System.Diagnostics.DiagnosticListener?displayProperty=fullName> | 单一实例 |

## <a name="register-additional-services-with-extension-methods"></a>使用扩展方法注册附加服务

当服务集合扩展方法可用于注册服务（及其依赖服务，如果需要）时，约定使用单个 `Add{SERVICE_NAME}` 扩展方法来注册该服务所需的所有服务。 以下代码是如何使用扩展方法 [AddDbContext\<TContext>](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext) 和 <xref:Microsoft.Extensions.DependencyInjection.IdentityServiceCollectionExtensions.AddIdentityCore*> 向容器添加附加服务的示例：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...

    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

    services.AddIdentity<ApplicationUser, IdentityRole>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultTokenProviders();

    ...
}
```

有关详细信息，请参阅 API 文档中的 <xref:Microsoft.Extensions.DependencyInjection.ServiceCollection> 类。

## <a name="service-lifetimes"></a>服务生存期

为每个注册的服务选择适当的生存期。 可以使用以下生存期配置 ASP.NET Core 服务：

### <a name="transient"></a>暂时

暂时生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddTransient*>) 是每次从服务容器进行请求时创建的。 这种生存期适合轻量级、 无状态的服务。

### <a name="scoped"></a>范围内

作用域生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddScoped*>) 以每个客户端请求（连接）一次的方式创建。

> [!WARNING]
> 在中间件内使用有作用域的服务时，请将该服务注入至 `Invoke` 或 `InvokeAsync` 方法。 请不要通过[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)进行注入，因为它会强制服务的行为与单一实例类似。 有关详细信息，请参阅 <xref:fundamentals/middleware/write#per-request-middleware-dependencies>。

### <a name="singleton"></a>单例

单一实例生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddSingleton*>) 是在第一次请求时（或者在运行 `Startup.ConfigureServices` 并且使用服务注册指定实例时）创建的。 每个后续请求都使用相同的实例。 如果应用需要单一实例行为，建议允许服务容器管理服务的生存期。 不要实现单一实例设计模式并提供用户代码来管理对象在类中的生存期。

> [!WARNING]
> 从单一实例解析有作用域的服务很危险。 当处理后续请求时，它可能会导致服务处于不正确的状态。

## <a name="service-registration-methods"></a>服务注册方法

服务注册扩展方法提供适用于特定场景的重载。

| 方法 | 自动<br>对象 (object)<br>处置 | 多种<br>实现 | 传递参数 |
| ---
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

--- | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---------------: | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-------------: | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

----: | | `Add{LIFETIME}<{SERVICE}, {IMPLEMENTATION}>()`<br>示例：<br>`services.AddSingleton<IMyDep, MyDep>();` | 是 | 是 | 否 | | `Add{LIFETIME}<{SERVICE}>(sp => new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton<IMyDep>(sp => new MyDep());`<br>`services.AddSingleton<IMyDep>(sp => new MyDep("A string!"));` | 是 | 是 | 是 | | `Add{LIFETIME}<{IMPLEMENTATION}>()`<br>示例：<br>`services.AddSingleton<MyDep>();` | 是 | 否 | 否 | | `AddSingleton<{SERVICE}>(new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton<IMyDep>(new MyDep());`<br>`services.AddSingleton<IMyDep>(new MyDep("A string!"));` | 否 | 是 | 是 | | `AddSingleton(new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton(new MyDep());`<br>`services.AddSingleton(new MyDep("A string!"));` | 否 | 否 | 是 |

要详细了解类型处置，请参阅[服务处置](#disposal-of-services)部分。 多个实现的常见场景是[为测试模拟类型](xref:test/integration-tests#inject-mock-services)。

`TryAdd{LIFETIME}` 方法仅当尚未注册实现时，注册该服务。

在以下示例中，第一行向 `IMyDependency` 注册 `MyDependency`。 第二行没有任何作用，因为 `IMyDependency` 已有一个已注册的实现：

```csharp
services.AddSingleton<IMyDependency, MyDependency>();
// The following line has no effect:
services.TryAddSingleton<IMyDependency, DifferentDependency>();
```

有关详细信息，请参见:

* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAdd*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddTransient*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddScoped*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddSingleton*>

[TryAddEnumerable(ServiceDescriptor)](xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddEnumerable*) 方法仅当没有同一类型的实现时，注册该服务。 多个服务通过 `IEnumerable<{SERVICE}>` 解析。 注册服务时，开发人员只希望在尚未添加一个相同类型时添加实例。 通常情况下，库创建者使用此方法来避免在容器中注册一个实例的两个副本。

在以下示例中，第一行向 `IMyDep1` 注册 `MyDep`。 第二行向 `IMyDep2` 注册 `MyDep`。 第三行没有任何作用，因为 `IMyDep1` 已有一个 `MyDep` 的已注册的实现：

```csharp
public interface IMyDep1 {}
public interface IMyDep2 {}

public class MyDep : IMyDep1, IMyDep2 {}

services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep1, MyDep>());
services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep2, MyDep>());
// Two registrations of MyDep for IMyDep1 is avoided by the following line:
services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep1, MyDep>());
```

### <a name="constructor-injection-behavior"></a>构造函数注入行为

服务可以通过两种机制来解析：

* <xref:System.IServiceProvider>
* <xref:Microsoft.Extensions.DependencyInjection.ActivatorUtilities>：允许在依赖关系注入容器中创建没有服务注册的对象。 `ActivatorUtilities` 用于面向用户的抽象，例如标记帮助器、MVC 控制器和模型绑定器。

构造函数可以接受依赖关系注入不提供的参数，但参数必须分配默认值。

当服务由 `IServiceProvider` 或 `ActivatorUtilities` 解析时，[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)需要 public 构造函数。

当服务由 `ActivatorUtilities` 解析时，[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)要求只存在一个适用的构造函数。 支持构造函数重载，但其参数可以全部通过依赖注入来实现的重载只能存在一个。

## <a name="entity-framework-contexts"></a>实体框架上下文

通常使用[设置了范围的生存期](#service-lifetimes)将实体框架上下文添加到服务容器中，因为 Web 应用数据库操作通常将范围设置为客户端请求。 如果在注册数据库上下文时，[AddDbContext\<TContext>](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext) 重载未指定生存期，则设置默认生存期范围。 给定生存期的服务不应使用生存期比服务短的数据库上下文。

## <a name="lifetime-and-registration-options"></a>生存期和注册选项

为了演示生存期和注册选项之间的差异，请考虑以下接口，将任务表示为具有唯一标识符 `OperationId` 的操作。 根据为以下接口配置操作服务的生存期的方式，容器在类请求时提供相同或不同的服务实例：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Interfaces/IOperation.cs?name=snippet1)]

接口在 `Operation` 类中实现。 `Operation` 构造函数将生成一个 GUID（如果未提供）：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Models/Operation.cs?name=snippet1)]

注册 `OperationService` 取决于，每个其他 `Operation` 类型。 当通过依赖关系注入请求 `OperationService` 时，它将接收每个服务的新实例或基于从属服务的生存期的现有实例。

* 如果从容器请求时创建了临时服务，则 `IOperationTransient` 服务的 `OperationId` 与 `OperationService` 的 `OperationId` 不同。 `OperationService` 将接收 `IOperationTransient` 类的新实例。 新实例将生成一个不同的 `OperationId`。
* 如果按客户端请求创建有作用域的服务，则 `IOperationScoped` 服务的 `OperationId` 与客户端请求中 `OperationService` 的该 ID 相同。 在客户端请求中，两个服务共享不同的 `OperationId` 值。
* 如果单一数据库和单一实例服务只创建一次并在所有客户端请求和所有服务中使用，则 `OperationId` 在所有服务请求中保持不变。

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Services/OperationService.cs?name=snippet1)]

在 `Startup.ConfigureServices` 中，根据其指定的生存期，将每个类型添加到容器中：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Startup.cs?name=snippet1&highlight=6-9,12)]

`IOperationSingletonInstance` 服务正在使用已知 ID 为 `Guid.Empty` 的特定实例。 此类型在使用时很明显（其 GUID 全部为零）。

示例应用演示了各个请求中和之间的对象生存期。 示例应用的 `IndexModel` 请求每种 `IOperation` 类型和 `OperationService`。 然后，页面通过属性分配显示所有页面模型类和服务的 `OperationId` 值：

[!code-csharp[](dependency-injection/samples/3.x/DependencyInjectionSample/Pages/Index.cshtml.cs?name=snippet1&highlight=7-11,14-18,21-25)]

以下两个输出显示了两个请求的结果：

**第一个请求：**

控制器操作：

暂时性：d233e165-f417-469b-a866-1cf1935d2518  
作用域：5d997e2d-55f5-4a64-8388-51c4e3a1ad19  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

`OperationService` 操作：

暂时性：c6b049eb-1318-4e31-90f1-eb2dd849ff64  
作用域：5d997e2d-55f5-4a64-8388-51c4e3a1ad19  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

**第二个请求：**

控制器操作：

暂时性：b63bd538-0a37-4ff1-90ba-081c5138dda0  
作用域：31e820c5-4834-4d22-83fc-a60118acb9f4  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

`OperationService` 操作：

暂时性：c4cbacb8-36a2-436d-81c8-8c1b78808aaf  
作用域：31e820c5-4834-4d22-83fc-a60118acb9f4  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

观察哪个 `OperationId` 值会在一个请求之内和不同请求之间变化：

* 暂时性对象始终不同。 第一个和第二个客户端请求的暂时性 `OperationId` 值对于 `OperationService` 操作和在客户端请求内都是不同的。 为每个服务请求和客户端请求提供了一个新实例。
* 作用域对象在一个客户端请求中是相同的，但在多个客户端请求中是不同的。
* 单一实例对象对每个对象和每个请求都是相同的（不管 `Startup.ConfigureServices` 中是否提供 `Operation` 实例）。

## <a name="call-services-from-main"></a>从 main 调用服务

使用 [IServiceScopeFactory.CreateScope](xref:Microsoft.Extensions.DependencyInjection.IServiceScopeFactory.CreateScope*) 创建 <xref:Microsoft.Extensions.DependencyInjection.IServiceScope> 以解析应用范围内的已设置范围的服务。 此方法可以用于在启动时访问有作用域的服务以便运行初始化任务。 以下示例演示如何在 `Program.Main` 中获取 `MyScopedService` 的上下文：

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Hosting;

public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();

        using (var serviceScope = host.Services.CreateScope())
        {
            var services = serviceScope.ServiceProvider;

            try
            {
                var serviceContext = services.GetRequiredService<MyScopedService>();
                // Use the context here
            }
            catch (Exception ex)
            {
                var logger = services.GetRequiredService<ILogger<Program>>();
                logger.LogError(ex, "An error occurred.");
            }
        }
    
        await host.RunAsync();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## <a name="scope-validation"></a>作用域验证

如果应用正在开发环境中运行，并调用 [CreateDefaultBuilder](xref:fundamentals/host/generic-host#default-builder-settings) 生成主机，默认服务提供程序会执行检查，以确认以下内容：

* 没有从根服务提供程序直接或间接解析到有作用域的服务。
* 未将有作用域的服务直接或间接注入到单一实例。

调用 <xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionContainerBuilderExtensions.BuildServiceProvider*> 时创建根服务提供程序。 在启动提供程序和应用时，根服务提供程序的生存期对应于应用/服务的生存期，并在关闭应用时释放。

有作用域的服务由创建它们的容器释放。 如果作用域创建于根容器，则该服务的生存会有效地提升至单一实例，因为根容器只会在应用/服务关闭时将其释放。 验证服务作用域，将在调用 `BuildServiceProvider` 时收集这类情况。

有关详细信息，请参阅 <xref:fundamentals/host/web-host#scope-validation>。

## <a name="request-services"></a>请求服务

来自 `HttpContext` 的 ASP.NET Core 请求中可用的服务通过 [HttpContext.RequestServices](xref:Microsoft.AspNetCore.Http.HttpContext.RequestServices) 集合公开。

请求服务表示作为应用的一部分配置和请求的服务。 当对象指定依赖关系时，`RequestServices`（而不是 `ApplicationServices`）中的类型将满足这些要求。

通常，应用不应直接使用这些属性。 相反，通过类构造函数请求类所需的类型，并允许框架注入依赖关系。 这样生成的类更易于测试。

> [!NOTE]
> 与访问 `RequestServices` 集合相比，以构造函数参数的形式请求依赖项是更优先的选择。

## <a name="design-services-for-dependency-injection"></a>设计能够进行依赖关系注入的服务

最佳做法是：

* 设计服务以使用依赖关系注入来获取其依赖关系。
* 避免有状态的、静态类和成员。 将应用设计为改用单一实例服务，可避免创建全局状态。
* 避免在服务中直接实例化依赖类。 直接实例化将代码耦合到特定实现。
* 不在应用类中包含过多内容，确保设计规范，并易于测试。

如果一个类似乎有过多的注入依赖关系，这通常表明该类拥有过多的责任并且违反了[单一责任原则 (SRP)](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#single-responsibility)。 尝试通过将某些职责移动到一个新类来重构类。 请记住，Razor Pages 页面模型类和 MVC 控制器类应关注用户界面问题。 业务规则和数据访问实现细节应保留在适用于这些[分离的关注点](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#separation-of-concerns)的类中。

### <a name="disposal-of-services"></a>服务处理

容器为其创建的 <xref:System.IDisposable> 类型调用 <xref:System.IDisposable.Dispose*>。 如果通过用户代码将实例添加到容器中，则不会自动处理该实例。

在下面的示例中，服务由服务容器创建，并自动处理：

```csharp
public class Service1 : IDisposable {}
public class Service2 : IDisposable {}

public interface IService3 {}
public class Service3 : IService3, IDisposable {}

public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<Service1>();
    services.AddSingleton<Service2>();
    services.AddSingleton<IService3>(sp => new Service3());
}
```

如下示例中：

* 服务实例不是由服务容器创建的。
* 框架不知道预期的服务生存期。
* 框架不会自动处理服务。
* 服务如果未通过开发者代码显式处理，则会一直保留，直到应用关闭。

```csharp
public class Service1 : IDisposable {}
public class Service2 : IDisposable {}

public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<Service1>(new Service1());
    services.AddSingleton(new Service2());
}
```

有关服务处理选项的讨论，请参阅 [ASP.NET Core 中处理 IDisposable 的四种方法](https://andrewlock.net/four-ways-to-dispose-idisposables-in-asp-net-core/)。

## <a name="default-service-container-replacement"></a>默认服务容器替换

内置的服务容器旨在满足框架和大多数消费者应用的需求。 我们建议使用内置容器，除非你需要的特定功能不受内置容器支持，例如：

* 属性注入
* 基于名称的注入
* 子容器
* 自定义生存期管理
* 对迟缓初始化的 `Func<T>` 支持
* 基于约定的注册

以下第三方容器可用于 ASP.NET Core 应用：

* [Autofac](https://autofac.readthedocs.io/en/latest/integration/aspnetcore.html)
* [DryIoc](https://www.nuget.org/packages/DryIoc.Microsoft.DependencyInjection)
* [Grace](https://www.nuget.org/packages/Grace.DependencyInjection.Extensions)
* [LightInject](https://github.com/seesharper/LightInject.Microsoft.DependencyInjection)
* [Lamar](https://jasperfx.github.io/lamar/)
* [Stashbox](https://github.com/z4kn4fein/stashbox-extensions-dependencyinjection)
* [Unity](https://www.nuget.org/packages/Unity.Microsoft.DependencyInjection)

### <a name="thread-safety"></a>线程安全

创建线程安全的单一实例服务。 如果单例服务依赖于一个瞬时服务，那么瞬时服务可能也需要线程安全，具体取决于单例使用它的方式。

单个服务的工厂方法（例如 [AddSingleton\<TService>(IServiceCollection, Func\<IServiceProvider,TService>)](xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddSingleton*) 的第二个参数）不必是线程安全的。 像类型 (`static`) 构造函数一样，它保证由单个线程调用一次。

## <a name="recommendations"></a>建议

* 不支持基于 `async/await` 和 `Task` 的服务解析。 C# 不支持异步构造函数；因此建议模式是在同步解析服务后使用异步方法。

* 避免在服务容器中直接存储数据和配置。 例如，用户的购物车通常不应添加到服务容器中。 配置应使用 [选项模型](xref:fundamentals/configuration/options)。 同样，避免"数据持有者"对象，也就是仅仅为实现对某些其他对象的访问而存在的对象。 最好通过 DI 请求实际项目。

* 避免静态访问服务（例如，静态键入 [IApplicationBuilder.ApplicationServices](xref:Microsoft.AspNetCore.Builder.IApplicationBuilder.ApplicationServices) 以便在其他地方使用）。

* 避免使用服务定位器模式。 例如，可以改用 DI 时，不要调用 <xref:System.IServiceProvider.GetService*> 来获取服务实例：

  **不正确：**

  ```csharp
  public class MyClass()
  {
      public void MyMethod()
      {
          var optionsMonitor = 
              _services.GetService<IOptionsMonitor<MyOptions>>();
          var option = optionsMonitor.CurrentValue.Option;

          ...
      }
  }
  ```

  **正确**：

  ```csharp
  public class MyClass
  {
      private readonly IOptionsMonitor<MyOptions> _optionsMonitor;

      public MyClass(IOptionsMonitor<MyOptions> optionsMonitor)
      {
          _optionsMonitor = optionsMonitor;
      }

      public void MyMethod()
      {
          var option = _optionsMonitor.CurrentValue.Option;

          ...
      }
  }
  ```

* 要避免的另一个服务定位器变体是注入可在运行时解析依赖项的工厂。 这两种做法混合了[控制反转](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#dependency-inversion)策略。

* 避免静态访问 `HttpContext`（例如，[IHttpContextAccessor.HttpContext](xref:Microsoft.AspNetCore.Http.IHttpContextAccessor.HttpContext)）。

像任何一组建议一样，你可能会遇到需要忽略某建议的情况。 例外情况很少见 &mdash; 主要是框架本身内部的特殊情况。

DI 是静态/全局对象访问模式的替代方法。 如果将其与静态对象访问混合使用，则可能无法实现 DI 的优点。

## <a name="recommended-patterns-for-multi-tenancy-in-di"></a>DI 中多租户的推荐模式

[Orchard Core](https://github.com/OrchardCMS/OrchardCore) 提供多租户。 有关详细信息，请参阅 [Orchard Core 文档](https://docs.orchardcore.net/en/dev/)。

请参阅 https://github.com/OrchardCMS/OrchardCore.Samples 上的示例应用，获取有关如何仅使用 Orchard Core Framework 而无需任何 CMS 特定功能来构建模块化和多租户应用的示例。

## <a name="additional-resources"></a>其他资源

* <xref:mvc/views/dependency-injection>
* <xref:mvc/controllers/dependency-injection>
* <xref:security/authorization/dependencyinjection>
* <xref:blazor/dependency-injection>
* <xref:fundamentals/startup>
* <xref:fundamentals/middleware/extensibility>
* [在 ASP.NET Core 中使用依赖关系注入编写干净代码 (MSDN) ](https://msdn.microsoft.com/magazine/mt703433.aspx)
* [Explicit Dependencies Principle](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#explicit-dependencies)（显式依赖关系原则）
* [控制反转容器和依赖关系注入模式 (Martin Fowler)](https://www.martinfowler.com/articles/injection.html)
* [如何在 ASP.NET Core DI 中注册具有多个接口的服务](https://andrewlock.net/how-to-register-a-service-with-multiple-interfaces-for-in-asp-net-core-di/)

::: moniker-end

::: moniker range="< aspnetcore-3.0"

ASP.NET Core 支持依赖关系注入 (DI) 软件设计模式，这是一种在类及其依赖关系之间实现[控制反转 (IoC)](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#dependency-inversion) 的技术。

有关特定于 MVC 控制器中依赖关系注入的详细信息，请参阅 <xref:mvc/controllers/dependency-injection>。

[查看或下载示例代码](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/fundamentals/dependency-injection/samples)（[如何下载](xref:index#how-to-download-a-sample)）

## <a name="overview-of-dependency-injection"></a>依赖关系注入概述

依赖项是另一个对象所需的任何对象。 使用应用中其他类所依赖的 `WriteMessage` 方法检查以下 `MyDependency` 类：

```csharp
public class MyDependency
{
    public MyDependency()
    {
    }

    public Task WriteMessage(string message)
    {
        Console.WriteLine(
            $"MyDependency.WriteMessage called. Message: {message}");

        return Task.FromResult(0);
    }
}
```

可以创建 `MyDependency` 类的实例以使 `WriteMessage` 方法可用于类。 `MyDependency` 类是 `IndexModel` 类的依赖项：

```csharp
public class IndexModel : PageModel
{
    MyDependency _dependency = new MyDependency();

    public async Task OnGetAsync()
    {
        await _dependency.WriteMessage(
            "IndexModel.OnGetAsync created this message.");
    }
}
```

该类创建并直接依赖于 `MyDependency` 实例。 代码依赖关系（如前面的示例）存在问题，应该避免使用，原因如下：

* 要用不同的实现替换 `MyDependency`，必须修改类。
* 如果 `MyDependency` 具有依赖关系，则必须由类对其进行配置。 在具有多个依赖于 `MyDependency` 的类的大型项目中，配置代码在整个应用中会变得分散。
* 这种实现很难进行单元测试。 应用应使用模拟或存根 `MyDependency` 类，该类不能使用此方法。

依赖关系注入通过以下方式解决了这些问题：

* 使用接口或基类抽象化依赖关系实现。
* 注册服务容器中的依赖关系。 ASP.NET Core 提供了一个内置的服务容器 <xref:System.IServiceProvider>。 服务已在应用的 `Startup.ConfigureServices` 方法中注册。
* 将服务注入到使用它的类的构造函数中。 框架负责创建依赖关系的实例，并在不再需要时对其进行处理。

在[示例应用](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/fundamentals/dependency-injection/samples)中，`IMyDependency` 接口定义了服务为应用提供的方法：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Interfaces/IMyDependency.cs?name=snippet1)]

此接口由具体类型 `MyDependency` 实现：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Services/MyDependency.cs?name=snippet1)]

`MyDependency` 在其构造函数中请求一个 <xref:Microsoft.Extensions.Logging.ILogger`1>。 以链式方式使用依赖关系注入并不罕见。 每个请求的依赖关系相应地请求其自己的依赖关系。 容器解析图中的依赖关系并返回完全解析的服务。 必须被解析的依赖关系的集合通常被称为“依赖关系树”、“依赖关系图”或“对象图”。

必须在服务容器中注册 `IMyDependency` 和 `ILogger<TCategoryName>`。 `IMyDependency` 已在 `Startup.ConfigureServices` 中注册。 `ILogger<TCategoryName>` 由日志记录抽象基础结构注册，因此它是框架默认注册的[框架提供的服务](#framework-provided-services)。

容器通过利用[（泛型）开放类型](/dotnet/csharp/language-reference/language-specification/types#open-and-closed-types)解析 `ILogger<TCategoryName>`，而无需注册每个[（泛型）构造类型](/dotnet/csharp/language-reference/language-specification/types#constructed-types)：

```csharp
services.AddSingleton(typeof(ILogger<>), typeof(Logger<>));
```

在示例应用中，使用具体类型 `MyDependency` 注册 `IMyDependency` 服务。 注册将服务生存期的范围限定为单个请求的生存期。 本主题后面将介绍[服务生存期](#service-lifetimes)。

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Startup.cs?name=snippet1&highlight=5)]

> [!NOTE]
> 每个 `services.Add{SERVICE_NAME}` 扩展方法添加（并可能配置）服务。 例如，`services.AddMvc()` 添加 Razor Pages 和 MVC 需要的服务。 我们建议应用遵循此约定。 将扩展方法置于 [Microsoft.Extensions.DependencyInjection](/dotnet/api/microsoft.extensions.dependencyinjection) 命名空间中以封装服务注册的组。

如果服务的构造函数需要[内置类型](/dotnet/csharp/language-reference/keywords/built-in-types-table)（如 `string`），则可以使用[配置](xref:fundamentals/configuration/index)或[选项模式](xref:fundamentals/configuration/options)注入该类型：

```csharp
public class MyDependency : IMyDependency
{
    public MyDependency(IConfiguration config)
    {
        var myStringValue = config["MyStringKey"];

        // Use myStringValue
    }

    ...
}
```

通过使用服务并分配给私有字段的类的构造函数请求服务的实例。 该字段用于在整个类中根据需要访问服务。

在示例应用中，请求 `IMyDependency` 实例并用于调用服务的 `WriteMessage` 方法：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Pages/Index.cshtml.cs?name=snippet1&highlight=3,6,13,29-30)]

## <a name="services-injected-into-startup"></a>注入启动的服务

使用泛型主机 (<xref:Microsoft.Extensions.Hosting.IHostBuilder>) 时，只能将以下服务类型注入 `Startup` 构造函数：

* `IWebHostEnvironment`
* <xref:Microsoft.Extensions.Hosting.IHostEnvironment>
* <xref:Microsoft.Extensions.Configuration.IConfiguration>

服务可以注入 `Startup.Configure`：

```csharp
public void Configure(IApplicationBuilder app, IOptions<MyOptions> options)
{
    ...
}
```

有关详细信息，请参阅 <xref:fundamentals/startup>。

## <a name="framework-provided-services"></a>框架提供的服务

`Startup.ConfigureServices` 方法负责定义应用使用的服务，包括 Entity Framework Core 和 ASP.NET Core MVC 等平台功能。 最初，提供给 `ConfigureServices` 的 `IServiceCollection` 具有框架定义的服务（具体取决于[主机配置方式](xref:fundamentals/index#host)）。 基于 ASP.NET Core 模板的应用程序具有框架注册的数百个服务的情况并不少见。 下表列出了框架注册的服务的一个小示例。

| 服务类型 | 生存期 |
| ---
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

------ | --- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---- | | <xref:Microsoft.AspNetCore.Hosting.Builder.IApplicationBuilderFactory?displayProperty=fullName> | 暂时 | | <xref:Microsoft.AspNetCore.Hosting.IApplicationLifetime?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Hosting.IHostingEnvironment?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Hosting.IStartup?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Hosting.IStartupFilter?displayProperty=fullName> | 暂时 | | <xref:Microsoft.AspNetCore.Hosting.Server.IServer?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.AspNetCore.Http.IHttpContextFactory?displayProperty=fullName> | 暂时 | | <xref:Microsoft.Extensions.Logging.ILogger`1?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.Logging.ILoggerFactory?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.ObjectPool.ObjectPoolProvider?displayProperty=fullName> | 单一实例 | | <xref:Microsoft.Extensions.Options.IConfigureOptions`1?displayProperty=fullName> | 暂时 | | <xref:Microsoft.Extensions.Options.IOptions`1?displayProperty=fullName> | 单一实例 | | <xref:System.Diagnostics.DiagnosticSource?displayProperty=fullName> | 单一实例 | | <xref:System.Diagnostics.DiagnosticListener?displayProperty=fullName> | 单一实例 |

## <a name="register-additional-services-with-extension-methods"></a>使用扩展方法注册附加服务

当服务集合扩展方法可用于注册服务（及其依赖服务，如果需要）时，约定使用单个 `Add{SERVICE_NAME}` 扩展方法来注册该服务所需的所有服务。 以下代码是如何使用扩展方法 [AddDbContext\<TContext>](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext) 和 <xref:Microsoft.Extensions.DependencyInjection.IdentityServiceCollectionExtensions.AddIdentityCore*> 向容器添加附加服务的示例：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...

    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

    services.AddIdentity<ApplicationUser, IdentityRole>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultTokenProviders();

    ...
}
```

有关详细信息，请参阅 API 文档中的 <xref:Microsoft.Extensions.DependencyInjection.ServiceCollection> 类。

## <a name="service-lifetimes"></a>服务生存期

为每个注册的服务选择适当的生存期。 可以使用以下生存期配置 ASP.NET Core 服务：

### <a name="transient"></a>暂时

暂时生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddTransient*>) 是每次从服务容器进行请求时创建的。 这种生存期适合轻量级、 无状态的服务。

### <a name="scoped"></a>范围内

作用域生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddScoped*>) 以每个客户端请求（连接）一次的方式创建。

> [!WARNING]
> 在中间件内使用有作用域的服务时，请将该服务注入至 `Invoke` 或 `InvokeAsync` 方法。 请不要通过[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)进行注入，因为它会强制服务的行为与单一实例类似。 有关详细信息，请参阅 <xref:fundamentals/middleware/write#per-request-middleware-dependencies>。

### <a name="singleton"></a>单例

单一实例生存期服务 (<xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddSingleton*>) 是在第一次请求时（或者在运行 `Startup.ConfigureServices` 并且使用服务注册指定实例时）创建的。 每个后续请求都使用相同的实例。 如果应用需要单一实例行为，建议允许服务容器管理服务的生存期。 不要实现单一实例设计模式并提供用户代码来管理对象在类中的生存期。

> [!WARNING]
> 从单一实例解析有作用域的服务很危险。 当处理后续请求时，它可能会导致服务处于不正确的状态。

## <a name="service-registration-methods"></a>服务注册方法

服务注册扩展方法提供适用于特定场景的重载。

| 方法 | 自动<br>对象 (object)<br>处置 | 多种<br>实现 | 传递参数 |
| ---
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

--- | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

---------------: | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-
title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

-------------: | :--- title: author: description: monikerRange: ms.author: ms.custom: ms.date: no-loc:
- 'Blazor'
- 'Identity'
- 'Let's Encrypt'
- 'Razor'
- 'SignalR' uid: 

----: | | `Add{LIFETIME}<{SERVICE}, {IMPLEMENTATION}>()`<br>示例：<br>`services.AddSingleton<IMyDep, MyDep>();` | 是 | 是 | 否 | | `Add{LIFETIME}<{SERVICE}>(sp => new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton<IMyDep>(sp => new MyDep());`<br>`services.AddSingleton<IMyDep>(sp => new MyDep("A string!"));` | 是 | 是 | 是 | | `Add{LIFETIME}<{IMPLEMENTATION}>()`<br>示例：<br>`services.AddSingleton<MyDep>();` | 是 | 否 | 否 | | `AddSingleton<{SERVICE}>(new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton<IMyDep>(new MyDep());`<br>`services.AddSingleton<IMyDep>(new MyDep("A string!"));` | 否 | 是 | 是 | | `AddSingleton(new {IMPLEMENTATION})`<br>示例：<br>`services.AddSingleton(new MyDep());`<br>`services.AddSingleton(new MyDep("A string!"));` | 否 | 否 | 是 |

要详细了解类型处置，请参阅[服务处置](#disposal-of-services)部分。 多个实现的常见场景是[为测试模拟类型](xref:test/integration-tests#inject-mock-services)。

`TryAdd{LIFETIME}` 方法仅当尚未注册实现时，注册该服务。

在以下示例中，第一行向 `IMyDependency` 注册 `MyDependency`。 第二行没有任何作用，因为 `IMyDependency` 已有一个已注册的实现：

```csharp
services.AddSingleton<IMyDependency, MyDependency>();
// The following line has no effect:
services.TryAddSingleton<IMyDependency, DifferentDependency>();
```

有关详细信息，请参见:

* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAdd*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddTransient*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddScoped*>
* <xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddSingleton*>

[TryAddEnumerable(ServiceDescriptor)](xref:Microsoft.Extensions.DependencyInjection.Extensions.ServiceCollectionDescriptorExtensions.TryAddEnumerable*) 方法仅当没有同一类型的实现时，注册该服务。 多个服务通过 `IEnumerable<{SERVICE}>` 解析。 注册服务时，开发人员只希望在尚未添加一个相同类型时添加实例。 通常情况下，库创建者使用此方法来避免在容器中注册一个实例的两个副本。

在以下示例中，第一行向 `IMyDep1` 注册 `MyDep`。 第二行向 `IMyDep2` 注册 `MyDep`。 第三行没有任何作用，因为 `IMyDep1` 已有一个 `MyDep` 的已注册的实现：

```csharp
public interface IMyDep1 {}
public interface IMyDep2 {}

public class MyDep : IMyDep1, IMyDep2 {}

services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep1, MyDep>());
services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep2, MyDep>());
// Two registrations of MyDep for IMyDep1 is avoided by the following line:
services.TryAddEnumerable(ServiceDescriptor.Singleton<IMyDep1, MyDep>());
```

### <a name="constructor-injection-behavior"></a>构造函数注入行为

服务可以通过两种机制来解析：

* <xref:System.IServiceProvider>
* <xref:Microsoft.Extensions.DependencyInjection.ActivatorUtilities>：允许在依赖关系注入容器中创建没有服务注册的对象。 `ActivatorUtilities` 用于面向用户的抽象，例如标记帮助器、MVC 控制器和模型绑定器。

构造函数可以接受依赖关系注入不提供的参数，但参数必须分配默认值。

当服务由 `IServiceProvider` 或 `ActivatorUtilities` 解析时，[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)需要 public 构造函数。

当服务由 `ActivatorUtilities` 解析时，[构造函数注入](xref:mvc/controllers/dependency-injection#constructor-injection)要求只存在一个适用的构造函数。 支持构造函数重载，但其参数可以全部通过依赖注入来实现的重载只能存在一个。

## <a name="entity-framework-contexts"></a>实体框架上下文

通常使用[设置了范围的生存期](#service-lifetimes)将实体框架上下文添加到服务容器中，因为 Web 应用数据库操作通常将范围设置为客户端请求。 如果在注册数据库上下文时，[AddDbContext\<TContext>](/dotnet/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions.adddbcontext) 重载未指定生存期，则设置默认生存期范围。 给定生存期的服务不应使用生存期比服务短的数据库上下文。

## <a name="lifetime-and-registration-options"></a>生存期和注册选项

为了演示生存期和注册选项之间的差异，请考虑以下接口，将任务表示为具有唯一标识符 `OperationId` 的操作。 根据为以下接口配置操作服务的生存期的方式，容器在类请求时提供相同或不同的服务实例：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Interfaces/IOperation.cs?name=snippet1)]

接口在 `Operation` 类中实现。 `Operation` 构造函数将生成一个 GUID（如果未提供）：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Models/Operation.cs?name=snippet1)]

注册 `OperationService` 取决于，每个其他 `Operation` 类型。 当通过依赖关系注入请求 `OperationService` 时，它将接收每个服务的新实例或基于从属服务的生存期的现有实例。

* 如果从容器请求时创建了临时服务，则 `IOperationTransient` 服务的 `OperationId` 与 `OperationService` 的 `OperationId` 不同。 `OperationService` 将接收 `IOperationTransient` 类的新实例。 新实例将生成一个不同的 `OperationId`。
* 如果按客户端请求创建有作用域的服务，则 `IOperationScoped` 服务的 `OperationId` 与客户端请求中 `OperationService` 的该 ID 相同。 在客户端请求中，两个服务共享不同的 `OperationId` 值。
* 如果单一数据库和单一实例服务只创建一次并在所有客户端请求和所有服务中使用，则 `OperationId` 在所有服务请求中保持不变。

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Services/OperationService.cs?name=snippet1)]

在 `Startup.ConfigureServices` 中，根据其指定的生存期，将每个类型添加到容器中：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Startup.cs?name=snippet1&highlight=6-9,12)]

`IOperationSingletonInstance` 服务正在使用已知 ID 为 `Guid.Empty` 的特定实例。 此类型在使用时很明显（其 GUID 全部为零）。

示例应用演示了各个请求中和之间的对象生存期。 示例应用的 `IndexModel` 请求每种 `IOperation` 类型和 `OperationService`。 然后，页面通过属性分配显示所有页面模型类和服务的 `OperationId` 值：

[!code-csharp[](dependency-injection/samples/2.x/DependencyInjectionSample/Pages/Index.cshtml.cs?name=snippet1&highlight=7-11,14-18,21-25)]

以下两个输出显示了两个请求的结果：

**第一个请求：**

控制器操作：

暂时性：d233e165-f417-469b-a866-1cf1935d2518  
作用域：5d997e2d-55f5-4a64-8388-51c4e3a1ad19  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

`OperationService` 操作：

暂时性：c6b049eb-1318-4e31-90f1-eb2dd849ff64  
作用域：5d997e2d-55f5-4a64-8388-51c4e3a1ad19  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

**第二个请求：**

控制器操作：

暂时性：b63bd538-0a37-4ff1-90ba-081c5138dda0  
作用域：31e820c5-4834-4d22-83fc-a60118acb9f4  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

`OperationService` 操作：

暂时性：c4cbacb8-36a2-436d-81c8-8c1b78808aaf  
作用域：31e820c5-4834-4d22-83fc-a60118acb9f4  
单一实例：01271bc1-9e31-48e7-8f7c-7261b040ded9  
实例：00000000-0000-0000-0000-000000000000

观察哪个 `OperationId` 值会在一个请求之内和不同请求之间变化：

* 暂时性对象始终不同。 第一个和第二个客户端请求的暂时性 `OperationId` 值对于 `OperationService` 操作和在客户端请求内都是不同的。 为每个服务请求和客户端请求提供了一个新实例。
* 作用域对象在一个客户端请求中是相同的，但在多个客户端请求中是不同的。
* 单一实例对象对每个对象和每个请求都是相同的（不管 `Startup.ConfigureServices` 中是否提供 `Operation` 实例）。

## <a name="call-services-from-main"></a>从 main 调用服务

使用 [IServiceScopeFactory.CreateScope](xref:Microsoft.Extensions.DependencyInjection.IServiceScopeFactory.CreateScope*) 创建 <xref:Microsoft.Extensions.DependencyInjection.IServiceScope> 以解析应用范围内的已设置范围的服务。 此方法可以用于在启动时访问有作用域的服务以便运行初始化任务。 以下示例演示如何在 `Program.Main` 中获取 `MyScopedService` 的上下文：

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateWebHostBuilder(args).Build();

        using (var serviceScope = host.Services.CreateScope())
        {
            var services = serviceScope.ServiceProvider;

            try
            {
                var serviceContext = services.GetRequiredService<MyScopedService>();
                // Use the context here
            }
            catch (Exception ex)
            {
                var logger = services.GetRequiredService<ILogger<Program>>();
                logger.LogError(ex, "An error occurred.");
            }
        }
    
        await host.RunAsync();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>();
}
```

## <a name="scope-validation"></a>作用域验证

如果在开发环境中运行应用，默认的服务提供程序会执行检查，从而确认以下内容：

* 没有从根服务提供程序直接或间接解析到有作用域的服务。
* 未将有作用域的服务直接或间接注入到单一实例。

调用 <xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionContainerBuilderExtensions.BuildServiceProvider*> 时创建根服务提供程序。 在启动提供程序和应用时，根服务提供程序的生存期对应于应用/服务的生存期，并在关闭应用时释放。

有作用域的服务由创建它们的容器释放。 如果作用域创建于根容器，则该服务的生存会有效地提升至单一实例，因为根容器只会在应用/服务关闭时将其释放。 验证服务作用域，将在调用 `BuildServiceProvider` 时收集这类情况。

有关详细信息，请参阅 <xref:fundamentals/host/web-host#scope-validation>。

## <a name="request-services"></a>请求服务

来自 `HttpContext` 的 ASP.NET Core 请求中可用的服务通过 [HttpContext.RequestServices](xref:Microsoft.AspNetCore.Http.HttpContext.RequestServices) 集合公开。

请求服务表示作为应用的一部分配置和请求的服务。 当对象指定依赖关系时，`RequestServices`（而不是 `ApplicationServices`）中的类型将满足这些要求。

通常，应用不应直接使用这些属性。 相反，通过类构造函数请求类所需的类型，并允许框架注入依赖关系。 这样生成的类更易于测试。

> [!NOTE]
> 与访问 `RequestServices` 集合相比，以构造函数参数的形式请求依赖项是更优先的选择。

## <a name="design-services-for-dependency-injection"></a>设计能够进行依赖关系注入的服务

最佳做法是：

* 设计服务以使用依赖关系注入来获取其依赖关系。
* 避免有状态的、静态类和成员。 将应用设计为改用单一实例服务，可避免创建全局状态。
* 避免在服务中直接实例化依赖类。 直接实例化将代码耦合到特定实现。
* 不在应用类中包含过多内容，确保设计规范，并易于测试。

如果一个类似乎有过多的注入依赖关系，这通常表明该类拥有过多的责任并且违反了[单一责任原则 (SRP)](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#single-responsibility)。 尝试通过将某些职责移动到一个新类来重构类。 请记住，Razor Pages 页面模型类和 MVC 控制器类应关注用户界面问题。 业务规则和数据访问实现细节应保留在适用于这些[分离的关注点](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#separation-of-concerns)的类中。

### <a name="disposal-of-services"></a>服务处理

容器为其创建的 <xref:System.IDisposable> 类型调用 <xref:System.IDisposable.Dispose*>。 如果通过用户代码将实例添加到容器中，则不会自动处理该实例。

在下面的示例中，服务由服务容器创建，并自动处理：

```csharp
public class Service1 : IDisposable {}
public class Service2 : IDisposable {}

public interface IService3 {}
public class Service3 : IService3, IDisposable {}

public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<Service1>();
    services.AddSingleton<Service2>();
    services.AddSingleton<IService3>(sp => new Service3());
}
```

如下示例中：

* 服务实例不是由服务容器创建的。
* 框架不知道预期的服务生存期。
* 框架不会自动处理服务。
* 服务如果未通过开发者代码显式处理，则会一直保留，直到应用关闭。

```csharp
public class Service1 : IDisposable {}
public class Service2 : IDisposable {}

public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<Service1>(new Service1());
    services.AddSingleton(new Service2());
}
```

## <a name="default-service-container-replacement"></a>默认服务容器替换

内置的服务容器旨在满足框架和大多数消费者应用的需求。 我们建议使用内置容器，除非你需要的特定功能不受内置容器支持，例如：

* 属性注入
* 基于名称的注入
* 子容器
* 自定义生存期管理
* 对迟缓初始化的 `Func<T>` 支持
* 基于约定的注册

以下第三方容器可用于 ASP.NET Core 应用：

* [Autofac](https://autofac.readthedocs.io/en/latest/integration/aspnetcore.html)
* [DryIoc](https://www.nuget.org/packages/DryIoc.Microsoft.DependencyInjection)
* [Grace](https://www.nuget.org/packages/Grace.DependencyInjection.Extensions)
* [LightInject](https://github.com/seesharper/LightInject.Microsoft.DependencyInjection)
* [Lamar](https://jasperfx.github.io/lamar/)
* [Stashbox](https://github.com/z4kn4fein/stashbox-extensions-dependencyinjection)
* [Unity](https://www.nuget.org/packages/Unity.Microsoft.DependencyInjection)

### <a name="thread-safety"></a>线程安全

创建线程安全的单一实例服务。 如果单例服务依赖于一个瞬时服务，那么瞬时服务可能也需要线程安全，具体取决于单例使用它的方式。

单个服务的工厂方法（例如 [AddSingleton\<TService>(IServiceCollection, Func\<IServiceProvider,TService>)](xref:Microsoft.Extensions.DependencyInjection.ServiceCollectionServiceExtensions.AddSingleton*) 的第二个参数）不必是线程安全的。 像类型 (`static`) 构造函数一样，它保证由单个线程调用一次。

## <a name="recommendations"></a>建议

* 不支持基于 `async/await` 和 `Task` 的服务解析。 C# 不支持异步构造函数；因此建议模式是在同步解析服务后使用异步方法。

* 避免在服务容器中直接存储数据和配置。 例如，用户的购物车通常不应添加到服务容器中。 配置应使用 [选项模型](xref:fundamentals/configuration/options)。 同样，避免"数据持有者"对象，也就是仅仅为实现对某些其他对象的访问而存在的对象。 最好通过 DI 请求实际项目。

* 避免静态访问服务（例如，静态键入 [IApplicationBuilder.ApplicationServices](xref:Microsoft.AspNetCore.Builder.IApplicationBuilder.ApplicationServices) 以便在其他地方使用）。

* 避免使用服务定位器模式，该模式混合了[控制反转](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#dependency-inversion)策略。

  * 可以改用 DI 时，不要调用 <xref:System.IServiceProvider.GetService*> 来获取服务实例：

    **不正确：**

    ```csharp
    public class MyClass()
    {
        public void MyMethod()
        {
            var optionsMonitor = 
                _services.GetService<IOptionsMonitor<MyOptions>>();
            var option = optionsMonitor.CurrentValue.Option;

            ...
        }
    }
    ```

    **正确**：

    ```csharp
    public class MyClass
    {
        private readonly IOptionsMonitor<MyOptions> _optionsMonitor;

        public MyClass(IOptionsMonitor<MyOptions> optionsMonitor)
        {
            _optionsMonitor = optionsMonitor;
        }

        public void MyMethod()
        {
            var option = _optionsMonitor.CurrentValue.Option;

            ...
        }
    }
    ```

  * 避免注入一个使用 <xref:System.IServiceProvider.GetService*> 在运行时解析依赖项的中心。

* 避免静态访问 `HttpContext`（例如，[IHttpContextAccessor.HttpContext](xref:Microsoft.AspNetCore.Http.IHttpContextAccessor.HttpContext)）。

像任何一组建议一样，你可能会遇到需要忽略某建议的情况。 例外情况很少见 &mdash; 主要是框架本身内部的特殊情况。

DI 是静态/全局对象访问模式的替代方法。 如果将其与静态对象访问混合使用，则可能无法实现 DI 的优点。

## <a name="additional-resources"></a>其他资源

* <xref:mvc/views/dependency-injection>
* <xref:mvc/controllers/dependency-injection>
* <xref:security/authorization/dependencyinjection>
* <xref:blazor/dependency-injection>
* <xref:fundamentals/startup>
* <xref:fundamentals/middleware/extensibility>
* [在 ASP.NET Core 中使用依赖关系注入编写干净代码 (MSDN) ](https://msdn.microsoft.com/magazine/mt703433.aspx)
* [Explicit Dependencies Principle](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#explicit-dependencies)（显式依赖关系原则）
* [控制反转容器和依赖关系注入模式 (Martin Fowler)](https://www.martinfowler.com/articles/injection.html)
* [如何在 ASP.NET Core DI 中注册具有多个接口的服务](https://andrewlock.net/how-to-register-a-service-with-multiple-interfaces-for-in-asp-net-core-di/)

::: moniker-end
