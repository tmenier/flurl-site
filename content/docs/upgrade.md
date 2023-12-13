## Upgrading from 3.x

This section highlights the features that were removed or significantly overhauled in Flurl.Http 4.0, and how to adapt existing code when upgrading.

### New Default JSON Serializer

Prior to 4.0, JSON serialization was backed by [Newtonsoft.Json](https://www.newtonsoft.com/json). In 4.0, Newtonsoft was [dropped](https://github.com/tmenier/Flurl/issues/517) in favor of `System.Text.Json` for the default implementation. There are a few things to look out for when you upgrade:

- Any [serialization attributes](https://www.newtonsoft.com/json/help/html/serializationattributes.htm) defined in Newtonsoft will no longer have any effect, and need be replaced by their STJ equivalents.
- Any of Newtonsoft's [configuration settings](https://www.newtonsoft.com/json/help/html/serializationsettings.htm) you may be using also no longer have any effect in Flurl.
- There are subtle differences in the default behaviors of the 2 libraries.

Microsoft provides a [migration guide](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/migrate-from-newtonsoft) that should help you navigate through all of these potential issues. Just be aware that the compiler won't likely catch the sorts of bugs you may encounter, so testing is critical.

_"This all sounds painful, I just want to keep using Newtonsoft!"_

You're in luck, because there's an alternative: install [Flurl.Http.Newtonsoft](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/), a companion package that brings back the original serializer in 4.0 and beyond. The package readme contains all details of its usage.

### No More Dynamics

`dynamic` types have somewhat [fallen out of favor](https://github.com/dotnet/runtime/issues/53195) in the .NET world and are not supported by `System.Text.Json`. Therefore, all non-generic, `dynamic`-returning methods like `GetJsonAsync()` were dropped in 4.0. If you used these methods, one way forward is to create full-blown classes to replace your `dynamic`s and use generic methods like `GetJsonAsync<T>()` instead.

But once again, the alternative is installing [Flurl.Http.Newtonsoft](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/). Since direct `dynamic` support still exists in Newtonsoft, those `dynamic`-returning Flurl methods were ported over to that library.

### No More Factories

In Flurl 3.x, you were able to fine-tune behavior by creating and registering implementations of `IFlurlClientFactory` and `IHttpClientFactory`.

The main use case for creating a custom `IFlurlClientFactory` was to define a custom strategy for caching clients when the clientless pattern is used; more specifically, overriding `DefaultFlurlClientFactory.GetCacheKey`. The direct replacement is to simply call `FlurlHttp.UseClientCachingStrategy(Func<IFlurlRequest, string>)`. An advantage of the new way is that you can use other properties of the request besides the URL, such as headers, to define a cache key. An example of that is shown [here](clients.md#clientless-usage).

`IHttpClientFactory` (not to be confused with .NET's [more-famous interface](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests) of the same name) was particularly useful as a hook to configure the underlying `HttpMessageHandler`. Uses ranged from [configuring proxies](https://stackoverflow.com/q/50649348/62600) to inserting [DelegatingHanders](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.delegatinghandler). This factory has also been removed, and the way forward is configuring these things fluently:

```cs
flurlClientBuilder
    .ConfigureHttpClient(hc => ...)
    .ConfigureInnerHandler(hch => ...)
    .AddMiddleware(() => ...)
```

How you get an `IFlurlClientBuilder` instance depends on whether you're using the clientless pattern or DI, whether you're configuring one client or all of them, etc. All of these things are described in detail in the The [Configuration](configuration.md) section.

### Configuring Clients and Requests

You may notice that calling `.Configure(settings => ...)` on a `FlurlClient` or `FlurlRequest` no longer compiles. This one's an easy fix: the method was renamed to `WithSettings` for better clarity.

### Global Configuration

The `FlurlHttp` static class that's been around since earlier versions still exists in 4.0, but its role has been reduced to managing a behind-the-scenes `IFlurlClientCache` used in support of the [clientless pattern](clients.md#clientless-usage). While the notion of a "global" settings object technically no longer exists, it has effectively been replaced by `FlurlHttp.Clients.WithDefaults` for the clientless pattern, and more generally (including when [using DI](clients.md#using-dependency-injection)) by `IFlurlClientCache.WithDefaults`. The [Configuration](configuration.md) section describes these patterns in more detail.