# 日志采样

比如 有时候同学们遇到一些问题也希望通过log检查问题，有些时候这些log在产线又会被关闭

.NET 提供[日志采样功能](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/log-sampling?tabs=dotnet-cli)，使你能够控制应用程序发出的日志量，而不会丢失重要信息。 可以使用以下采样策略：

基于跟踪的采样：基于当前跟踪的采样决策采样日志。
随机概率采样：基于配置的概率规则对日志进行采样。

为了方便大家使用 VKProxy也提供相关的功能配置

日志采样通过更精细地控制应用程序发出的日志来扩展 筛选功能 。 与其简单地启用或禁用日志，您可以通过配置采样比例来仅发出部分日志。

例如，虽然筛选通常使用概率（ 0 不发出日志）或 1 （发出所有日志），但采样允许你选择介于两者之间的任何值，例如 0.1 发出 10% 个日志，或 0.25 发出 25% 个。

## 配置随机概率采样

随机概率采样允许你根据配置的概率规则对日志进行采样。 可以定义以下特定规则：

- 日志类别
- 日志级别
- 事件编号

可以在 `appsettings.json` 配置文件中修改

如

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  },

  "RandomProbabilisticSampler": {
    "Rules": [
      {
        "CategoryName": "Microsoft.AspNetCore.*",
        "Probability": 0.25,
        "LogLevel": "Information"
      },
      {
        "CategoryName": "System.*",
        "Probability": 0.1
      },
      {
        "EventId": 1001,
        "Probability": 0.05
      }
    ]
  }
}
```

运行可以用

# [dotnet tool](#tab/tool)

``` shell
vkproxy -c D:\code\test\proxy\config.json --sampler random
```

---

## 配置基于跟踪的采样

基于跟踪的采样可确保日志被与底层Activity一致地采样。 如果要在跟踪和日志之间保持相关性，这非常有用。 可以启用跟踪采样

当启用基于跟踪的采样时，仅当基础 Activity 被采样时才会记录日志。 采样决策来自当前 Recorded 值。

运行可以用

# [dotnet tool](#tab/tool)

``` shell
vkproxy -c D:\code\test\proxy\config.json --sampler trace
```

---

