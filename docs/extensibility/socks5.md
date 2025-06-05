# 如何利用中间件扩展实现socks5

socks5 代理协议已经有很多文章说明，这里不再赘述，想了解的可以参见[https://zh.wikipedia.org/wiki/SOCKS](https://zh.wikipedia.org/wiki/SOCKS)

这里列举一下核心实现

``` csharp
internal class Socks5Middleware : ITcpProxyMiddleware
{
    private readonly IDictionary<byte, ISocks5Auth> auths;
    private readonly IConnectionFactory tcp;
    private readonly IHostResolver hostResolver;
    private readonly ITransportManager transport;
    private readonly IUdpConnectionFactory udp;

    public Socks5Middleware(IEnumerable<ISocks5Auth> socks5Auths, IConnectionFactory tcp, IHostResolver hostResolver, ITransportManager transport, IUdpConnectionFactory udp)
    {
        this.auths = socks5Auths.ToFrozenDictionary(i => i.AuthType);
        this.tcp = tcp;
        this.hostResolver = hostResolver;
        this.transport = transport;
        this.udp = udp;
    }

    public Task InitAsync(ConnectionContext context, CancellationToken token, TcpDelegate next)
    {
       // 识别是否为 socks5 路由
        var feature = context.Features.Get<IL4ReverseProxyFeature>();
        if (feature is not null)
        {
            var route = feature.Route;
            if (route is not null && route.Metadata is not null
                && route.Metadata.TryGetValue("socks5", out var b) && bool.TryParse(b, out var isSocks5) && isSocks5)
            {
                feature.IsDone = true;
                return Proxy(context, feature, token);
            }
        }
        return next(context, token);
    }

    public Task<ReadOnlyMemory<byte>> OnRequestAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        return next(context, source, token); // 不做任何处理，原样传递
    }

    public Task<ReadOnlyMemory<byte>> OnResponseAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        return next(context, source, token); // 不做任何处理，原样传递
    }

    private async Task Proxy(ConnectionContext context, IL4ReverseProxyFeature feature, CancellationToken token)
    {
        var input = context.Transport.Input;
        var output = context.Transport.Output;
        // 1. socks5 认证
        if (!await Socks5Parser.AuthAsync(input, auths, context, token))
        {
            context.Abort();
        }
        // 2. 获取 socks5 命令请求
        var cmd = await Socks5Parser.GetCmdRequestAsync(input, token);
        IPEndPoint ip = await ResolveIpAsync(context, cmd, token);
        switch (cmd.Cmd)
        {
            case Socks5Cmd.Connect:
            case Socks5Cmd.Bind:
                // 3. 如果为tcp代理，则会在此分支处理，以命令请求中的地址建立tcp链接
                ConnectionContext upstream;
                try
                {
                    upstream = await tcp.ConnectAsync(ip, token);
                }
                catch
                {  // 为了简单，这里异常没有详细分区各种情况
                    await Socks5Parser.ResponeAsync(output, Socks5CmdResponseType.ConnectFail, token);
                    throw;
                }
                // 4. 服务tcp建立成功，通知 client
                await Socks5Parser.ResponeAsync(output, Socks5CmdResponseType.Success, token);
                var task = await Task.WhenAny(
                               context.Transport.Input.CopyToAsync(upstream.Transport.Output, token)
                               , upstream.Transport.Input.CopyToAsync(context.Transport.Output, token));
                if (task.IsCanceled)
                {
                    context.Abort();
                }
                break;

            case Socks5Cmd.UdpAssociate:
                // 3. 如果为udp代理，则会在此分支处理，建立临时 udp 代理服务地址
                var local = context.LocalEndPoint as IPEndPoint;
                var op = new EndPointOptions()
                {
                    EndPoint = new UdpEndPoint(local.Address, 0),
                    Key = Guid.NewGuid().ToString(),
                };
                try
                {
                    var remote = context.RemoteEndPoint;
                    var timeout = feature.Route.Timeout;
                    op.EndPoint = await transport.BindAsync(op, c => ProxyUdp(c as UdpConnectionContext, remote, timeout), token);
                    // 5. tcp 关闭时 需要关闭临时 udp 服务
                    context.ConnectionClosed.Register(state => transport.StopEndpointsAsync(new List<EndPointOptions>() { state as EndPointOptions }, CancellationToken.None).ConfigureAwait(false).GetAwaiter().GetResult(), op);
                }
                catch
                {
                    await Socks5Parser.ResponeAsync(output, Socks5CmdResponseType.ConnectFail, token);
                    throw;
                }
                 // 4. 服务udp建立成功，通知 client 临时udp地址
                await Socks5Parser.ResponeAsync(output, op.EndPoint as IPEndPoint, Socks5CmdResponseType.Success, token);
                break;
        }
    }

    private async Task ProxyUdp(UdpConnectionContext context, EndPoint remote, TimeSpan timeout)
    {
        using var cts = CancellationTokenSourcePool.Default.Rent(timeout);
        var token = cts.Token;
        // 这里用为了简单 同一个临时地址即监听client 也处理 服务端 response，通过端口比较区分， 当然这样存在一定安全问题 
        if (context.RemoteEndPoint.GetHashCode() == remote.GetHashCode())
        {
            var req = Socks5Parser.GetUdpRequest(context.ReceivedBytes);
            IPEndPoint ip = await ResolveIpAsync(req, token);
            // 请求服务，解包原始请求
            await udp.SendToAsync(context.Socket, ip, req.Data, token);
        }
        else
        {
           
            // 服务response，封包
            await Socks5Parser.UdpResponeAsync(udp, context, remote as IPEndPoint, token);
        }
    }

    private async Task<IPEndPoint> ResolveIpAsync(ConnectionContext context, Socks5Common cmd, CancellationToken token)
    {
        IPEndPoint ip = await ResolveIpAsync(cmd, token);
        if (ip is null)
        {
            await Socks5Parser.ResponeAsync(context.Transport.Output, Socks5CmdResponseType.AddressNotAllow, token);
            context.Abort();
        }

        return ip;
    }

    private async Task<IPEndPoint> ResolveIpAsync(Socks5Common cmd, CancellationToken token)
    {
        IPEndPoint ip;
        if (cmd.Domain is not null)
        {
            var ips = await hostResolver.HostResolveAsync(cmd.Domain, token);
            if (ips.Length > 0)
            {
                ip = new IPEndPoint(ips.First(), cmd.Port);
            }
            else
                ip = null;
        }
        else if (cmd.Ip is not null)
        {
            ip = new IPEndPoint(cmd.Ip, cmd.Port);
        }
        else
        {
            ip = null;
        }

        return ip;
    }
}
```