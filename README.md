# JustEat.StatsD

[![NuGet version](https://buildstats.info/nuget/JustEat.StatsD?includePreReleases=false)](http://www.nuget.org/packages/JustEat.StatsD)

| | Linux | Windows |
|:-:|:-:|:-:|
| **Build Status** | [![Build status](https://img.shields.io/travis/justeat/JustEat.StatsD/master.svg)](https://travis-ci.org/justeat/JustEat.StatsD) | [![Build status](https://img.shields.io/appveyor/ci/justeattech/justeat-statsd/master.svg)](https://ci.appveyor.com/project/justeattech/justeat-statsd) |
| **Build History** | [![Build history](https://buildstats.info/travisci/chart/justeat/JustEat.StatsD?branch=master&includeBuildsFromPullRequest=false)](https://travis-ci.org/justeat/JustEat.StatsD) |  [![Build history](https://buildstats.info/appveyor/chart/justeattech/justeat-statsd?branch=master&includeBuildsFromPullRequest=false)](https://ci.appveyor.com/project/justeattech/justeat-statsd) |

## The point

### TL;DR

We use this within our components to publish [statsd](http://github.com/etsy/statsd) metrics from .NET code. We've been using this in most of our things, since around 2013.

### Features

* statsd metrics formatter
* UDP client handling

#### Publishing statistics

`IStatsDPublisher` is the interface that you will use in most circumstances. With this you can `Increment` or `Decrement` an event, and send values for a `Gauge` or `Timing`.

The concrete class that implements `IStatsDPublisher` is `StatsDPublisher`. The constructor takes an instance of `StatsDConfiguration`. For the configuration's values, you will always need the statsd server host name or IP address. Optionally, you can change the standard port (8125). You can also prepend a prefix to all stats. These values often come from configuration as the host name and/or prefix may vary between test and production environments.

## Setting up StatsD

### Simple example of setting up a StatsDPublisher

An example of a very simple StatsD publisher configuration, using the default values for most things.

#### .NET Core

Using `IServiceCollection` and the built-in DI container:

```csharp
// Registration
services.AddStatsD("metrics_server.mycompany.com");
services.AddSingleton<MyService>();

// Consumption
public class MyService
{
    public MyService(IStatsDPublisher statsPublisher)
    {
        // Use the publisher
    }
}

```

#### Simplest example

No service registration or IoC. Works for both .NET 4.5.1 and .NET Core.

```csharp
var statsDConfig = new StatsDConfiguration { Host = "metrics_server.mycompany.com" };
IStatsDPublisher statsDPublisher = new StatsDPublisher(statsDConfig);
```

### IoC Examples

#### .NET Core

An example of registering StatsD dependencies using `IServiceCollection`:

```csharp
services.AddStatsD(
    (provider) =>
    {
        var options = provider.GetRequiredService<MyOptions>().StatsD;

        return new StatsDConfiguration()
        {
            Host = options.HostName,
            Port = options.Port,
            Prefix = options.Prefix,
            OnError = ex => LogError(ex)
        };
    });
```

#### .NET Core (AWS Lambda functions)

An example of registering StatsD dependencies using `IServiceCollection` when using an AWS Lambda function:

```csharp
// Simple
services.AddSingleton<IStatsDTransport, IpTransport>();
services.AddStatsD(Environment.GetEnvironmentVariable("MY_STATSD_ENDPOINT"));

// Advanced
services.AddSingleton<IStatsDTransport, IpTransport>();
services.AddStatsD(
    (provider) =>
    {
        var options = provider.GetRequiredService<MyOptions>().StatsD;

        return new StatsDConfiguration()
        {
            Host = options.HostName,
            Port = options.Port,
            Prefix = options.Prefix,
            OnError = ex => LogError(ex)
        };
    });
```

#### .NET 4.5.1

An example of IoC in Ninject for StatsD publisher with values for all options, read from configuration:

```csharp

// read config values from app settings
string statsdHostName =  ConfigurationManager.AppSettings["statsd.hostname"];
int statsdPort = int.Parse(ConfigurationManager.AppSettings["statsd.port"]);
string statsdPrefix =  ConfigurationManager.AppSettings["statsd.prefix"];

// assemble a StatsDConfiguration object
// since the configuration doesn't change for the lifetime of the app,
// it can be a preconfigured singleton instance
var statsDConfig = new StatsDConfiguration
{
  Host = statsdHostName,
  Port = statsdPort,
  Prefix = statsdPrefix,
  OnError = ex => LogError(ex)
};

// register with NInject
Bind<StatsDConfiguration>().ToConstant(statsDConfig>);
Bind<IStatsDPublisher>().To<StatsDPublisher>();
```

## StatsDConfiguration fields

| Name              | Type                    | Default                        | Comments                                                                                                |
|-------------------|-------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------|
| Host              | `string`                |                                | The host name or IP address of the StatsD server. There is no default, this must be set.                |
| Port              | `int`                   | `8125`                         | The StatsD port.                                                                                        |
| DnsLookupInterval | `TimeSpan?`             | `5 minutes`                    | Length of time to cache the host name to IP address lookup. Only used when "Host" contains a host name. |
| Prefix            | `string`                | `string.Empty`                 | Prepend a prefix to all stats.                                                                          |
| OnError           | `Func<Exception, bool>` | `null`                         | Function to receive notification of any exceptions.                                                     |

`OnError` is a function to receive notification of any errors that occur when trying to publish a metric. This function should return:
 *  **True** if the exception was handled and no further action is needed
 *  **False** if the exception should be thrown

The default behaviour is to ignore the error.


#### Example of using the interface

Given an existing instance of `IStatsDPublisher` called `stats` you can do for e.g.:

```csharp
stats.Increment("DoSomething.Attempt");
var stopWatch = Stopwatch.StartNew();
var success = DoSomething();

stopWatch.Stop();

var statName = "DoSomething." + success ? "Success" : "Failure";
stats.Timing(statName, stopWatch.Elapsed);
```

#### Simple timers

This syntax for timers less typing in simple cases, where you always want to time the operation, and always with the same stat name. Given an existing instance of `IStatsDPublisher` you can do:

```csharp
//  timing a block of code in a using statement:
using (stats.StartTimer("someStat"))
{
    DoSomething();
}

// also works with async
using (stats.StartTimer("someStat"))
{
    await DoSomethingAsync();
}
```

The `StartTimer` returns an `IDisposableTimer` that wraps a stopwatch and implements `IDisposable`.
The stopwatch is automatically stopped and the metric sent when it falls out of scope and is disposed on the closing `}` of the `using` statement.

##### Changing the name of simple timers

Sometimes the decision of which stat to send should not be taken before the operation completes. e.g. When you are timing http operations and want different status codes to be logged under different stats.

The `IDisposableTimer` has a `StatName` property to set or change the name of the stat. To use this you need a reference to the timer, e.g. `using (var timer = stats.StartTimer("statName"))` instead of `using (stats.StartTimer("statName"))`

The stat name must be set to a non-empty string at the end of the `using` block.

```csharp
using (var timer = stats.StartTimer("SomeHttpOperation."))
{
    var response = DoSomeHttpOperation();
    timer.StatName = timer.StatName + (int)response.StatusCode;
    return response;
}
```

##### Functional style

```csharp
//  timing an action without a return value:
stats.Time("someStat", t => DoSomething());

//  timing a function with a return value:
var result = stats.Time("someStat", t => GetSomething());

// and it correctly times async code using the usual syntax:
await stats.Time("someStat", async t => await DoSomethingAsync());
var result = await stats.Time("someStat", async t => await GetSomethingAsync());
```

In all these cases the function or delegate is supplied with a `IDisposableTimer t` so that the stat name can be changed if need be.

##### Credits

The idea of "disposable timers" for using statements is an old one, see for example [this StatsD client](https://github.com/Pereingo/statsd-csharp-client) and [MiniProfiler](https://miniprofiler.com/dotnet/HowTo/ProfileCode).

### How to contribute

See [CONTRIBUTING.md](.github/CONTRIBUTING.md).

### How to release

See [CONTRIBUTING.md](.github/CONTRIBUTING.md#releases).
