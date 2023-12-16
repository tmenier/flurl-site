## Upgrading from 3.x

This section highlights the most significant breaking changes in Flurl.Http 4.0 and how to adapt your existing code when upgrading.

### New Default JSON Serializer

Prior to 4.0, JSON serialization was backed by [Newtonsoft.Json](https://www.newtonsoft.com/json). In 4.0, Newtonsoft was [dropped](https://github.com/tmenier/Flurl/issues/517) in favor of `System.Text.Json` for the default implementation. There are a few things to look out for when you upgrade:

- Any [serialization attributes](https://www.newtonsoft.com/json/help/html/serializationattributes.htm) defined in Newtonsoft will no longer have any effect, and need be replaced by their STJ equivalents.
- Any of Newtonsoft's [configuration settings](https://www.newtonsoft.com/json/help/html/serializationsettings.htm) you may be using also no longer have any effect in Flurl.
- There are subtle differences in the default behaviors of the 2 libraries.

Microsoft provides a [migration guide](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/migrate-from-newtonsoft) that should help you navigate through all of these potential issues. Just be aware that the compiler won't likely catch the sorts of bugs you may encounter, so testing is critical.

_"This all sounds painful, I just want to keep using Newtonsoft!"_

You're in luck, because there's an alternative: install [Flurl.Http.Newtonsoft](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/), a companion package that brings back the 3.x serializer in 4.0 and beyond. Refer to the package [readme](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/#readme-body-tab) for details of its usage.

### No More Dynamics

`dynamic` types have somewhat [fallen out of favor](https://github.com/dotnet/runtime/issues/53195) in the .NET world and are not supported by `System.Text.Json`. Therefore, all non-generic, `dynamic`-returning methods like `GetJsonAsync()` were [dropped](https://github.com/tmenier/Flurl/issues/699) in 4.0. If you use these methods today, one way forward is to create full-blown classes to replace your `dynamic`s and use generic methods like `GetJsonAsync<T>()` instead.

But once again, the alternative is installing [Flurl.Http.Newtonsoft](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/). Since direct `dynamic` support still exists in Newtonsoft, those `dynamic`-returning Flurl methods were ported over to that library.

### No More Factories

In Flurl 3.x, you were able to fine-tune behavior by creating and registering custom implementations of `IFlurlClientFactory` and `IHttpClientFactory`. Both have been removed in favor of simpler, more fluent ways to achieve the same results, such as configuring the inner `HttpMessageHandler` and changing the client caching strategy when the clientless pattern is used. Refer to the matrix below for details.

### Configuration Changes

Flurl's configuration system got a major [overhaul](https://github.com/tmenier/Flurl/issues/770) in 4.0, with the goals of making it more fluent, more [DI-friendly](clients.md#using-dependency-injection), and a bit more familiar to those accustomed to using [.NET's IHttpClientFactory](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests). This matrix should help you adapt your existing code:

| How do I... | 3.x | 4.0 ||
|--|--|--|--|
| Configure client or request settings | `cliOrRequest.Configure` | `cliOrRequest.WithSettings` | [more info](configuration.md#settings) |
| Configure client used to call a specific URL | `FlurlHttp.ConfigureClient` | `FlurlHttp.ConfigureClientForUrl` | [more info](clients.md#clientless-usage) |
| Configure "global" defaults | `FlurlHttp.Configure` | `FlurlHttp.Clients.WithDefaults` | [more info](configuration.md#settings) |
| Handle an event | `cliOrRequest.Settings   .BeforeCall[Async] = call => ...` | `cliOrRequest.BeforeCall(call => ...)` | [more info](configuration.md#event-handlers) |
| Configure inner-most HttpMessageHandler | Inherit from `DefaultHttpClientFactory`, override `CreateMessageHandler`, register factory | `IFlurlClientBuilder.ConfigureInnerHandler` | [more info](configuration.md#message-handlers) |
| Add a DelegatingHandler | Inherit from `DefaultHttpClientFactory`, override `CreateMessageHandler`, register factory | `IFlurlClientBuilder.AddMiddleware` | [more info](configuration.md#message-handlers) |
| Change the client caching strategy | Inherit from `DefaultFlurlClientFactory`, override `GetCacheKey`, register factory | `FlurlHttp.UseClientCachingStrategy` | [more info](clients.md#clientless-usage) |

### More Changes

Although this page is not meant to be an exhaustive list of changes in 4.0, the [release notes](https://github.com/tmenier/Flurl/releases/tag/Flurl.Http.4.0.0) provide such a list. If you encounter a pain point while upgrading that you feel should be better covered here, you're encouraged to edit this page directly using the link below, or [open an issue in the docs repo](https://github.com/tmenier/flurl-site/issues/new) to make a more general suggestion.