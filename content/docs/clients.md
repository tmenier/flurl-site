## Managing Clients

Flurl.Http is built on top of the `System.Net.Http` stack. If you're familiar with `HttpClient`, then you've probably heard this advice: don't create a new client for every request; reuse them, or [face the consequences](https://www.aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/). A `FlurlClient` wraps a single `HttpClient` and is bound to the same lifetime, hence the advice is similar.

### Clientless Usage

If you don't want to manage `FlurlClient` instances explicitly, you don't need to; Flurl will do this for you. In fact, the client is noteably absent in most examples on this site:

```cs
var result = await "https://some-api.com".GetJsonAsync<T>();
```

But much like the term "serverless", Flurl's "clientless" pattern doesn't mean there's literally no client - it just means you don't need to deal with it explicitly, and you can trust it's being managed smartly. To do this, Flurl caches and reuses client instances using `FlurlHttp.Clients`, which is a global singleton instance of `IFlurlClientCache`. Clients can be pre-configured at startup:

```cs
FlurlHttp.ConfigureClientForUrl("https://some-api.com")
    .WithSettings(settings => ...)
    .WithHeaders(...)
```
_Note: `WithSettings` and other related methods are covered in detail in the [Configuration](configuration.md) section._

With the clientless pattern, all calls to the same host (or more accurately, same combination of scheme/host/port) will reuse the same client. Any client-level configurations, such as default headers, will come along for the ride for all calls to that host. You can change this caching strategy if you want. For example, perhaps you want to use a different client for each version of a service, identifiable by a custom request header:

```c#
FlurlHttp.UseClientCachingStrategy(request =>
    // get the default host-based key by invoking the default caching strategy:
    var name = FlurlHttp.BuildClientNameByHost(request);
    // append the service version from a custom request header, if it exists:
    if (request.Headers.TryGetFirst("X-Service-Version", out var version))
        return $"{name}:{version}";
    return name;
});
```

When using a custom caching strategy, be sure `UseClientCachingStrategy` is called _before_ any calls to `ConfigureClientForUrl`.

### Explicit Usage

Many developers, especially those who want to adhere strictly to dependency injection principals, may be turned off by the existence of a global static context for managing clients, arguing that this can lead to tighter coupling and make systems harder to test. For these reasons, Flurl supports an alternative usage pattern where client management is explicit (and almost as easy):

```cs
var client = new FlurlClient("https://some-api.com") // a base URL is optional
    .WithSettings(settings => ...)
    .WithHeaders(...);

var result = await client.Request("path").GetJsonAsync<T>();
```

Here, the `Request` method (which takes zero or more strings as a shortcut to `AppendPathSegments`) creates an `IFlurlRequest` object - the same as gets created by the string extension method in the clientless example. Therefore, all the same fluent, chainable methods are available in either case.

### Using Dependency Injection

To make Flurl fully DI-friendly, one issue remains: we don't want to `new` up that client from inside our service classes; we want to inject something. The recommended approach here is to register `IFlurlClientCache` as a singleton with the container, bound to `FlurlClientCache` and optionally configured from the composition root, and injected into your services. `IFlurlClientCache` supports named clients, which are conceptually very similar to [named clients with IHttpClientFactory](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests#named-clients).

```cs
// at service registration:
services.AddSingleton<IFlurlClientCache>(sp => new FlurlClientCache()
    .Add("MyCli", "https://some-api.com"));

// in service class:
public class MyService : IMyService
{
    private readonly IFlurlClient _flurlCli;

    public MyService(IFlurlClientCache clients) {
        _flurlCli = clients.Get("MyCli");
    }
}
```

Much like the clientless pattern, all clients are managed by a single instance of `IFlurlClientCache`. But that singleton is now under the control of your IoC container rather than the static `FlurlHttp` object. Win!

In the above example, `clients.Get` would throw an exception if the named client wasn't already created (at service registration in this example). But pre-creating clients at startup is not strictly required - `GetOrAdd` could have been used instead:

```cs
_flurlCli = clients.GetOrAdd("MyCli", "https://some-api.com");
```

Both `Add` and `GetOrAdd` support an optional third parameter - an `Action<IFlurlClientBuilder>` that allows you to (lazily) configure the client:

```cs
.Add("MyCli", "https://some-api.com", builder => builder
    .WithSettings(settings => ...)
    .WithHeaders(...)
```
