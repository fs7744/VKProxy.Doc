# 如何扩展HTTP转换器

在 [如何为HTTP配置请求和响应转换](/VKProxy.Doc/docs/transforms) 已经列举了所有内置支持的HTTP转换器

还可以以两种接口形式实施扩展

- `ITransformFactory`

    多个`ITransformFactory`只会有最先匹配的一个执行，其他都会忽略， 适合配置规则匹配的形式

- `ITransformProvider`

    每一个`ITransformProvider`都会运行，适合全局默认规则


## ITransformFactory

多个`ITransformFactory`只会有最先匹配的一个执行，其他都会忽略， 适合配置规则匹配的形式

举例

```csharp
public class TestAddResponseHeader : ResponseTransform
{
    private string v;

    public TestAddResponseHeader(string v)
    {
        this.v = v;
    }

    public override ValueTask ApplyAsync(ResponseTransformContext context)
    {
        SetHeader(context, $"x-{v}", v);

        return ValueTask.CompletedTask;
    }
}

internal class TestTransformFactory : ITransformFactory
{
    public bool Build(TransformBuilderContext context, IReadOnlyDictionary<string, string> transformValues)
    {
        if (transformValues.TryGetValue("myHeader", out var v))  // 配置要满足有 myHeader
        {
            context.ResponseTransforms.Add(new TestAddResponseHeader(v));

            return true;
        }
        return false;
    }
}
```


然后注入DI

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<ITransformFactory, TestTransformFactory>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

大家在配置时还要指定一下


``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { "myHeader": "gogo" }
        ]
      }
    }
  }
}
```

## ITransformProvider

每一个`ITransformProvider`都会运行，适合全局默认规则

举例

```csharp
internal class TestITransformProvider : ITransformProvider
{
    public void Apply(TransformBuilderContext context) // 全部都会运行
    {
        context.ResponseTransforms.Add(new TestAddResponseHeader("all"));
    }
}
```

然后注入DI

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<ITransformProvider, TestITransformProvider>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

无需特别配置