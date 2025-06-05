# 可扩展套接字应用程序框架

除了代理功能外，由于通过反射释放了Kestrel的能力，你也可以把 VKProxy 当成可扩展套接字应用程序框架使用

使用它轻松构建始终连接的套接字应用程序，而无需考虑如何使用套接字，如何维护套接字连接以及套接字如何工作。

(在Kestrel基础上开发，理论可以帮大家节省一些比如直接socket要自己管理 socket之类的事情)

## IListenHandler

大家只要实现 `IListenHandler`, 就可以实现自己的任何协议服务了 

（`IListenHandler` 支持同时运行多个，只要监听的端口不冲突就好， 甚至还可以和 asp.net core 在一个程序里面一起运行 ）

``` csharp
namespace VKProxy.Core.Hosting;

public interface IListenHandler
{
    /// <summary>
    /// init whatever you need
    /// </summary>
    /// <param name="cancellationToken"></param>
    /// <returns></returns>
    Task InitAsync(CancellationToken cancellationToken);

    /// <summary>
    /// if not need Rebind, please return null
    /// </summary>
    /// <returns></returns>
    IChangeToken? GetReloadToken();

    Task BindAsync(ITransportManager transportManager, CancellationToken cancellationToken);

    /// <summary>
    /// Rebind will be called when ReloadToken change
    /// </summary>
    /// <param name="transportManager"></param>
    /// <param name="cancellationToken"></param>
    /// <returns></returns>
    Task RebindAsync(ITransportManager transportManager, CancellationToken cancellationToken);

    Task StopAsync(ITransportManager transportManager, CancellationToken cancellationToken);
}
```

如果大家的服务不需要像 proxy 这种支持动态变更监听， 可以直接继承 `ListenHandlerBase`， 这个类帮大家省略一些不需要的方法

``` csharp
public abstract class ListenHandlerBase : IListenHandler
{
    public abstract Task BindAsync(ITransportManager transportManager, CancellationToken cancellationToken);

    public virtual IChangeToken? GetReloadToken()
    {
        return null;
    }

    public virtual Task InitAsync(CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }

    public virtual Task RebindAsync(ITransportManager transportManager, CancellationToken cancellationToken)
    {
        throw new NotImplementedException();
    }

    public virtual Task StopAsync(ITransportManager transportManager, CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }
}
```

如下列举一些示例

## HTTP

哈哈，虽然推荐普通业务还是使用 asp.net core, 但是如果想尝试还是有办法的

首先 我们实现一个 http 的`IListenHandler`

``` csharp
public class HttpListenHandler : ListenHandlerBase
{
    private readonly ILogger<HttpListenHandler> logger;
    private readonly ICertificateLoader certificateLoader;

    public HttpListenHandler(ILogger<HttpListenHandler> logger, ICertificateLoader certificateLoader)
    {
        this.logger = logger;
        this.certificateLoader = certificateLoader;
    }

    public override async Task BindAsync(ITransportManager transportManager, CancellationToken cancellationToken)
    {
        try
        {
            // 监听 127.0.0.1:4000 只允许 http
            var ip = new EndPointOptions()
            {
                EndPoint = IPEndPoint.Parse("127.0.0.1:4000"),
                Key = "http"
            };
            await transportManager.BindHttpAsync(ip, Proxy, cancellationToken);
            logger.LogInformation($"listen {ip.EndPoint}");

            // 监听 127.0.0.1:4001 但允许 https + Http1AndHttp2AndHttp3, 同时自签证书设置不验证
            ip = new EndPointOptions()
            {
                EndPoint = IPEndPoint.Parse("127.0.0.1:4001"),
                Key = "https"
            };

            var (c, f) = certificateLoader.LoadCertificate(new CertificateConfig() { Path = "testCert.pfx", Password = "testPassword" });
            await transportManager.BindHttpAsync(ip, Proxy, cancellationToken, HttpProtocols.Http1AndHttp2AndHttp3, callbackOptions: new HttpsConnectionAdapterOptions()
            {
                //ServerCertificateSelector = (context, host) => c
                ServerCertificate = c,
                CheckCertificateRevocation = false,
                ClientCertificateMode = ClientCertificateMode.AllowCertificate
            });
            logger.LogInformation($"listen {ip.EndPoint}");
        }
        catch (Exception ex)
        {
            logger.LogError(ex.Message, ex);
        }
    }


   // http 处理， 这里随意写了几段， 大家可以切换 http 协议版本观察差异
    private async Task Proxy(HttpContext context)
    {
        var resp = context.Response;
        if (string.Equals(context.Request.Method, "OPTIONS", StringComparison.OrdinalIgnoreCase))
        {
            resp.Headers.Origin = "*";
            resp.Headers.AccessControlAllowOrigin = "*";
            return;
        }

        if (string.Equals(context.Request.Path, "/testhttp", StringComparison.OrdinalIgnoreCase))
        {
            resp.Headers.ContentType = "text/html";
            await resp.WriteAsync("""
                <!DOCTYPE html>
                <html>
                <body>
                <p id="demo">Fetch a file to change this text.</p>
                <script>

                fetch('https://127.0.0.1:4001/api', {
                  method: 'POST',
                  headers: {
                    'Accept': 'application/json',
                    'Content-Type': 'application/json',
                    'Origin': 'https://127.0.0.1:4001'
                  },
                  protocol: 'http3',
                })
                .then(response => {
                  if (!response.ok) {
                    throw new Error('Network response was not ok');
                  }
                  return response.json();
                })
                .then(data => {
                document.getElementById("demo").innerHTML =data.protocol;
                  console.log(data);
                })
                .catch(error => {
                  console.error('There was a problem with the fetch operation:', error);
                });

                </script>
                </body>
                </html>

                """);
            return;
        }

        resp.StatusCode = 404;
        await resp.WriteAsJsonAsync(new { context.Request.Protocol });
        await resp.CompleteAsync().ConfigureAwait(false);
    }
}
```

当然 我们还得注入 `HttpListenHandler`

``` csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using VKProxy.Core.Hosting;

var app = Host.CreateDefaultBuilder(args)
    .UseVKProxyCore()  // 不必注入 VKProxy 完全功能， 只需网络处理的核心部分
    .ConfigureServices(i =>
    {
        i.AddSingleton<IListenHandler, HttpListenHandler>();  // 这里注入
    })
    .Build();

await app.RunAsync();
```

## TCP

那么tcp又怎么样完成呢？

首先 我们实现一个 tcp 的`IListenHandler`

``` csharp
internal class TcpListenHandler : ListenHandlerBase
{
    private readonly List<EndPointOptions> endPointOptions = new List<EndPointOptions>();
    private readonly ILogger<TcpListenHandler> logger;
    private readonly IConnectionFactory connectionFactory;

    public TcpListenHandler(ILogger<TcpListenHandler> logger, IConnectionFactory connectionFactory)
    {
        this.logger = logger;
        this.connectionFactory = connectionFactory;
    }

    public override Task InitAsync(CancellationToken cancellationToken) // 这里做一点点差异， 假设 监听地址是从配置拿去，就可以在此方法处理
    {
        endPointOptions.Add(new EndPointOptions()
        {
            EndPoint = IPEndPoint.Parse("127.0.0.1:5000"),
            Key = "tcp"
        });
        return Task.CompletedTask;
    }

    public override async Task BindAsync(ITransportManager transportManager, CancellationToken cancellationToken) // 利用 transportManager 监听
    {
        foreach (var item in endPointOptions)
        {
            try
            {
                await transportManager.BindAsync(item, Proxy, cancellationToken);
                logger.LogInformation($"listen {item.EndPoint}");
            }
            catch (Exception ex)
            {
                logger.LogError(ex.Message, ex);
            }
        }
    }

    private async Task Proxy(ConnectionContext connection) // 为了简单就实现一个简单的 tcp 转发
    {
        logger.LogInformation($"begin tcp {DateTime.Now} {connection.LocalEndPoint.ToString()} ");
        var upstream = await connectionFactory.ConnectAsync(new IPEndPoint(IPAddress.Parse("14.215.177.38"), 80));
        var task1 = connection.Transport.Input.CopyToAsync(upstream.Transport.Output);
        var task2 = upstream.Transport.Input.CopyToAsync(connection.Transport.Output);
        await Task.WhenAny(task1, task2);
        upstream.Abort();
        connection.Abort();
        logger.LogInformation($"end tcp {DateTime.Now} {connection.LocalEndPoint.ToString()} ");
    }
}
```

当然 我们还得注入 `TcpListenHandler`

``` csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using VKProxy.Core.Hosting;

var app = Host.CreateDefaultBuilder(args)
    .UseVKProxyCore()  // 不必注入 VKProxy 完全功能， 只需网络处理的核心部分
    .ConfigureServices(i =>
    {
        i.AddSingleton<IListenHandler, TcpListenHandler>();  // 这里注入
    })
    .Build();

await app.RunAsync();
```


## UDP

那么udp又怎么样完成呢？

首先 我们实现一个 udp 的`IListenHandler`

``` csharp
internal class UdpListenHandler : ListenHandlerBase
{
    private readonly ILogger<UdpListenHandler> logger;
    private readonly IUdpConnectionFactory udp;
    private readonly IPEndPoint proxyServer = new(IPAddress.Parse("127.0.0.1"), 11000);

    public UdpListenHandler(ILogger<UdpListenHandler> logger, IUdpConnectionFactory udp)
    {
        this.logger = logger;
        this.udp = udp;
    }

    public override async Task BindAsync(ITransportManager transportManager, CancellationToken cancellationToken) // 监听
    {
        var ip = new EndPointOptions()
        {
            EndPoint = UdpEndPoint.Parse("127.0.0.1:5000"),
            Key = "tcp"
        };
        await transportManager.BindAsync(ip, Proxy, cancellationToken);
        logger.LogInformation($"listen {ip.EndPoint}");
    }

    private async Task Proxy(ConnectionContext connection) // 同理 一个简单的 udp 转发
    {
        if (connection is UdpConnectionContext context)
        {
            Console.WriteLine($"{context.LocalEndPoint} received {context.ReceivedBytesCount} from {context.RemoteEndPoint}");
            var socket = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
            await udp.SendToAsync(socket, proxyServer, context.ReceivedBytes, CancellationToken.None);
            var r = await udp.ReceiveAsync(socket, CancellationToken.None);
            await udp.SendToAsync(context.Socket, context.RemoteEndPoint, r.GetReceivedBytes(), CancellationToken.None);
        }
    }
}
```

当然 我们还得注入 `UdpListenHandler`

``` csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using VKProxy.Core.Hosting;

var app = Host.CreateDefaultBuilder(args)
    .UseVKProxyCore()  // 不必注入 VKProxy 完全功能， 只需网络处理的核心部分
    .ConfigureServices(i =>
    {
        i.AddSingleton<IListenHandler, UdpListenHandler>();  // 这里注入
    })
    .Build();

await app.RunAsync();
```


## 如何释放 Kestrel 能力的

众所周知 Kestrel 是 Aspnetcore 为了跨平台而实现的web server，只提供 http 1/2/3 的 L7层的能力

但看过源码的同学都知道，其实其本身从L4层(socket)实现的Http协议处理，只是[OnBind](https://github.com/dotnet/aspnetcore/blob/main/src/Servers/Kestrel/Core/src/Internal/KestrelServerImpl.cs#L137)只有http相关实现以及没有提供相关公开扩展的api，所以限制了其能力

但是既然代码是开源的，并且我们也知道dotnet有虽然麻烦但是能跨越访问限制的能力（Reflection），所以它是不能阻挡我们的魔爪

（ps 
     1. 不过这样绕过限制可能会在[Native AOT](https://learn.microsoft.com/zh-cn/dotnet/core/deploying/native-aot/?tabs=windows%2Cnet8)相关场景存在问题，目前暂时没有做具体相关测试
     2. 在不同版本Kestrel 可能会存在api变动，目前为了省事，不适配各版本差异，暂时以net9.0为准，net10正式发布后迁移升级到net10，此后不再适配net9.0之前版本
 ）

### 适配Kestrel 的核心点
核心重点在暴露[TransportManager](https://github.com/dotnet/aspnetcore/blob/main/src/Servers/Kestrel/Core/src/Internal/Infrastructure/TransportManager.cs) api， 这样大家就有了L4层的处理能力

TransportManagerAdapter 实现
```csharp
public class TransportManagerAdapter : ITransportManager, IHeartbeat
{
    private static MethodInfo StopAsyncMethod;
    private static MethodInfo StopEndpointsAsyncMethod;
    private static MethodInfo MultiplexedBindAsyncMethod;
    private static MethodInfo BindAsyncMethod;
    private static MethodInfo StartHeartbeatMethod;
    private object transportManager;
    private object heartbeat;
    private object serviceContext;
    private object metrics;
    private int multiplexedTransportCount;
    private int transportCount;
    internal readonly IServiceProvider serviceProvider;

    IServiceProvider ITransportManager.ServiceProvider => serviceProvider;

    public TransportManagerAdapter(IServiceProvider serviceProvider, IEnumerable<IConnectionListenerFactory> transportFactories, IEnumerable<IMultiplexedConnectionListenerFactory> multiplexedConnectionListenerFactories)
    {
        (transportManager, heartbeat, serviceContext, metrics) = CreateTransportManager(serviceProvider);
        multiplexedTransportCount = multiplexedConnectionListenerFactories.Count();
        transportCount = transportFactories.Count();
        this.serviceProvider = serviceProvider;
    }

    private static (object, object, object, object) CreateTransportManager(IServiceProvider serviceProvider)
    {
        foreach (var item in KestrelExtensions.TransportManagerType.GetTypeInfo().DeclaredMethods)
        {
            if (item.Name == "StopAsync")
            {
                StopAsyncMethod = item;
            }
            else if (item.Name == "StopEndpointsAsync")
            {
                StopEndpointsAsyncMethod = item;
            }
            else if (item.Name == "BindAsync")
            {
                if (item.GetParameters().Any(i => i.ParameterType == typeof(ConnectionDelegate)))
                {
                    BindAsyncMethod = item;
                }
                else
                {
                    MultiplexedBindAsyncMethod = item;
                }
            }
        }

        var s = CreateServiceContext(serviceProvider);
        var r = Activator.CreateInstance(KestrelExtensions.TransportManagerType,
                    Enumerable.Reverse(serviceProvider.GetServices<IConnectionListenerFactory>()).ToList(),
                    Enumerable.Reverse(serviceProvider.GetServices<IMultiplexedConnectionListenerFactory>()).ToList(),
                    CreateHttpsConfigurationService(serviceProvider),
                    s.context
                    );
        return (r, s.heartbeat, s.context, s.metrics);

        static object CreateHttpsConfigurationService(IServiceProvider serviceProvider)
        {
            var CreateLogger = typeof(LoggerFactoryExtensions).GetTypeInfo().DeclaredMethods.First(i => i.Name == "CreateLogger" && i.ContainsGenericParameters);
            var r = Activator.CreateInstance(KestrelExtensions.HttpsConfigurationServiceType);
            var m = KestrelExtensions.HttpsConfigurationServiceType.GetMethod("Initialize");
            var log = serviceProvider.GetRequiredService<ILoggerFactory>();
            var l = CreateLogger.MakeGenericMethod(KestrelExtensions.HttpsConnectionMiddlewareType).Invoke(null, new object[] { log });
            m.Invoke(r, new object[] { serviceProvider.GetRequiredService<IHostEnvironment>(), log.CreateLogger<KestrelServer>(), l });
            return r;
        }

        static (object context, object heartbeat, object metrics) CreateServiceContext(IServiceProvider serviceProvider)
        {
            var m = CreateKestrelMetrics();
            var KestrelCreateServiceContext = KestrelExtensions.KestrelServerImplType.GetMethod("CreateServiceContext", System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
            var r = KestrelCreateServiceContext.Invoke(null, new object[]
            {
                serviceProvider.GetRequiredService<IOptions<KestrelServerOptions>>(),
                serviceProvider.GetRequiredService<ILoggerFactory>(),
                null,
                m
            });
            var h = KestrelExtensions.ServiceContextType.GetTypeInfo().DeclaredProperties.First(i => i.Name == "Heartbeat");
            StartHeartbeatMethod = KestrelExtensions.HeartbeatType.GetTypeInfo().DeclaredMethods.First(i => i.Name == "Start");
            return (r, h.GetGetMethod().Invoke(r, null), m);
        }

        static object CreateKestrelMetrics()
        {
            return Activator.CreateInstance(KestrelExtensions.KestrelMetricsType, Activator.CreateInstance(KestrelExtensions.DummyMeterFactoryType));
        }
    }

    public Task<EndPoint> BindAsync(EndPointOptions endpointConfig, ConnectionDelegate connectionDelegate, CancellationToken cancellationToken)
    {
        return BindAsyncMethod.Invoke(transportManager, new object[] { endpointConfig.EndPoint, connectionDelegate, endpointConfig.Init(), cancellationToken }) as Task<EndPoint>;
    }

    public Task<EndPoint> BindAsync(EndPointOptions endpointConfig, MultiplexedConnectionDelegate multiplexedConnectionDelegate, CancellationToken cancellationToken)
    {
        return MultiplexedBindAsyncMethod.Invoke(transportManager, new object[] { endpointConfig.EndPoint, multiplexedConnectionDelegate, endpointConfig.GetListenOptions(), cancellationToken }) as Task<EndPoint>;
    }

    public Task StopEndpointsAsync(List<EndPointOptions> endpointsToStop, CancellationToken cancellationToken)
    {
        return StopEndpointsAsyncMethod.Invoke(transportManager, new object[] { EndPointOptions.Init(endpointsToStop), cancellationToken }) as Task;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        return StopAsyncMethod.Invoke(transportManager, new object[] { cancellationToken }) as Task;
    }

    public void StartHeartbeat()
    {
        if (heartbeat != null)
        {
            StartHeartbeatMethod.Invoke(heartbeat, null);
        }
    }

    public void StopHeartbeat()
    {
        if (heartbeat is IDisposable disposable)
        {
            disposable.Dispose();
        }
    }

    public IConnectionBuilder UseHttpServer(IConnectionBuilder builder, IHttpApplication<HttpApplication.Context> application, HttpProtocols protocols, bool addAltSvcHeader)
    {
        KestrelExtensions.UseHttpServerMethod.Invoke(null, new object[] { builder, serviceContext, application, protocols, addAltSvcHeader });
        return builder;
    }

    public IMultiplexedConnectionBuilder UseHttp3Server(IMultiplexedConnectionBuilder builder, IHttpApplication<HttpApplication.Context> application, HttpProtocols protocols, bool addAltSvcHeader)
    {
        KestrelExtensions.UseHttp3ServerMethod.Invoke(null, new object[] { builder, serviceContext, application, protocols, addAltSvcHeader });
        return builder;
    }

    public ConnectionDelegate UseHttps(ConnectionDelegate next, HttpsConnectionAdapterOptions tlsCallbackOptions, HttpProtocols protocols)
    {
        if (tlsCallbackOptions == null)
            return next;
        var o = KestrelExtensions.HttpsConnectionMiddlewareInitMethod.Invoke(new object[] { next, tlsCallbackOptions, protocols, serviceProvider.GetRequiredService<ILoggerFactory>(), metrics });
        return KestrelExtensions.HttpsConnectionMiddlewareOnConnectionAsyncMethod.CreateDelegate<ConnectionDelegate>(o);
    }

    public async Task BindHttpApplicationAsync(EndPointOptions options, IHttpApplication<HttpApplication.Context> application, CancellationToken cancellationToken, HttpProtocols protocols = HttpProtocols.Http1AndHttp2AndHttp3, bool addAltSvcHeader = true, Action<IConnectionBuilder> config = null
        , Action<IMultiplexedConnectionBuilder> configMultiplexed = null, HttpsConnectionAdapterOptions callbackOptions = null)
    {
        var hasHttp1 = protocols.HasFlag(HttpProtocols.Http1);
        var hasHttp2 = protocols.HasFlag(HttpProtocols.Http2);
        var hasHttp3 = protocols.HasFlag(HttpProtocols.Http3);
        var hasTls = callbackOptions is not null;

        if (hasTls)
        {
            if (hasHttp3)
            {
                options.GetListenOptions().Protocols = protocols;
                options.SetHttpsOptions(callbackOptions);
            }
            //callbackOptions.SetHttpProtocols(protocols);
            //if (hasHttp3)
            //{
            //    HttpsConnectionAdapterOptions
            //    options.SetHttpsCallbackOptions(callbackOptions);
            //}
        }
        else
        {
            // Http/1 without TLS, no-op HTTP/2 and 3.
            if (hasHttp1)
            {
                hasHttp2 = false;
                hasHttp3 = false;
            }
            // Http/3 requires TLS. Note we only let it fall back to HTTP/1, not HTTP/2
            else if (hasHttp3)
            {
                throw new InvalidOperationException("HTTP/3 requires HTTPS.");
            }
        }

        // Quic isn't registered if it's not supported, throw if we can't fall back to 1 or 2
        if (hasHttp3 && multiplexedTransportCount == 0 && !(hasHttp1 || hasHttp2))
        {
            throw new InvalidOperationException("Unable to bind an HTTP/3 endpoint. This could be because QUIC has not been configured using UseQuic, or the platform doesn't support QUIC or HTTP/3.");
        }

        addAltSvcHeader = addAltSvcHeader && multiplexedTransportCount > 0;

        // Add the HTTP middleware as the terminal connection middleware
        if (hasHttp1 || hasHttp2
            || protocols == HttpProtocols.None)
        {
            if (transportCount == 0)
            {
                throw new InvalidOperationException($"Cannot start HTTP/1.x or HTTP/2 server if no {nameof(IConnectionListenerFactory)} is registered.");
            }

            var builder = new ConnectionBuilder(serviceProvider);
            config?.Invoke(builder);
            UseHttpServer(builder, application, protocols, addAltSvcHeader);
            var connectionDelegate = UseHttps(builder.Build(), callbackOptions, protocols);

            options.EndPoint = await BindAsync(options, connectionDelegate, cancellationToken).ConfigureAwait(false);
        }

        if (hasHttp3 && multiplexedTransportCount > 0)
        {
            var builder = new MultiplexedConnectionBuilder(serviceProvider);
            configMultiplexed?.Invoke(builder);
            UseHttp3Server(builder, application, protocols, addAltSvcHeader);
            var multiplexedConnectionDelegate = builder.Build();

            options.EndPoint = await BindAsync(options, multiplexedConnectionDelegate, cancellationToken).ConfigureAwait(false);
        }
    }
}
```

其次通过重写 `VKServer` 从而去除 OnBind 方法的影响，达到大家可以使用 `ITransportManager` 做任意 L4/L7的处理

```csharp
public class VKServer : IServer
{
    private readonly ITransportManager transportManager;
    private readonly IHeartbeat heartbeat;
    private readonly IListenHandler listenHandler;
    private readonly GeneralLogger logger;
    private bool _hasStarted;
    private int _stopping;
    private readonly SemaphoreSlim _bindSemaphore = new SemaphoreSlim(initialCount: 1);
    private readonly CancellationTokenSource _stopCts = new CancellationTokenSource();
    private readonly TaskCompletionSource _stoppedTcs = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
    private IDisposable? _configChangedRegistration;

    public VKServer(ITransportManager transportManager, IHeartbeat heartbeat, IListenHandler listenHandler, GeneralLogger logger)
    {
        this.transportManager = transportManager;
        this.heartbeat = heartbeat;
        this.listenHandler = listenHandler;
        this.logger = logger;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        try
        {
            if (_hasStarted)
            {
                throw new InvalidOperationException("Server already started");
            }
            _hasStarted = true;
            await listenHandler.InitAsync(cancellationToken);
            heartbeat.StartHeartbeat();
            await BindAsync(cancellationToken).ConfigureAwait(false);
        }
        catch
        {
            Dispose();
            throw;
        }
    }

    private async Task BindAsync(CancellationToken cancellationToken)
    {
        await _bindSemaphore.WaitAsync(cancellationToken).ConfigureAwait(false);

        try
        {
            if (_stopping == 1)
            {
                throw new InvalidOperationException("Server has already been stopped.");
            }

            IChangeToken? reloadToken = listenHandler.GetReloadToken();
            await listenHandler.BindAsync(transportManager, _stopCts.Token).ConfigureAwait(false);
            _configChangedRegistration = reloadToken?.RegisterChangeCallback(TriggerRebind, this);
        }
        finally
        {
            _bindSemaphore.Release();
        }
    }

    private void TriggerRebind(object? state)
    {
        if (state is VKServer server)
        {
            _ = server.RebindAsync();
        }
    }

    private async Task RebindAsync()
    {
        await _bindSemaphore.WaitAsync();

        IChangeToken? reloadToken = null;
        try
        {
            if (_stopping == 1)
            {
                return;
            }

            reloadToken = listenHandler.GetReloadToken();
            await listenHandler.RebindAsync(transportManager, _stopCts.Token).ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            logger.UnexpectedException("Unable to reload configuration", ex);
        }
        finally
        {
            _configChangedRegistration = reloadToken?.RegisterChangeCallback(TriggerRebind, this);
            _bindSemaphore.Release();
        }
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        if (Interlocked.Exchange(ref _stopping, 1) == 1)
        {
            await _stoppedTcs.Task.ConfigureAwait(false);
            return;
        }

        heartbeat.StopHeartbeat();

        _stopCts.Cancel();

        await _bindSemaphore.WaitAsync().ConfigureAwait(false);

        try
        {
            await listenHandler.StopAsync(transportManager, cancellationToken).ConfigureAwait(false);
            await transportManager.StopAsync(cancellationToken).ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            _stoppedTcs.TrySetException(ex);
            throw;
        }
        finally
        {
            _configChangedRegistration?.Dispose();
            _stopCts.Dispose();
            _bindSemaphore.Release();
        }

        _stoppedTcs.TrySetResult();
    }

    public void Dispose()
    {
        StopAsync(new CancellationToken(canceled: true)).GetAwaiter().GetResult();
    }
}
```