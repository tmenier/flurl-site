## Configuration

Flurl.Http behavior is configurable via a system of hierarchical settings, each level inheriting/overriding the previous in this order:

- `FlurlHttp.GlobalSettings` (static)
- `IFlurlClient.Settings`
- `IFlurlRequest.Settings`
- `HttpTest.Settings` (configured test settings always "win")

Available properties are mostly the same at all 4 levels, with a few exceptions. Here's a complete list along with where they are and are not supported:

Property                | FlurlHttp (global) | HttpTest | IFlurlClient | IFlurlRequest
------------------------|:------------------:|:--------:|:------------:|:-------------:
Timeout                 |         x          |    x     |      x       |       x         
AllowedHttpStatusRange  |         x          |    x     |      x       |       x         
JsonSerializer          |         x          |    x     |      x       |       x         
UrlEncodedSerializer    |         x          |    x     |      x       |       x         
Redirects               |         x          |    x     |      x       |       x         
BeforeCall              |         x          |    x     |      x       |       x         
BeforeCallAsync         |         x          |    x     |      x       |       x         
AfterCall               |         x          |    x     |      x       |       x         
AfterCallAsync          |         x          |    x     |      x       |       x         
OnError                 |         x          |    x     |      x       |       x         
OnErrorAsync            |         x          |    x     |      x       |       x         
OnRedirect              |         x          |    x     |      x       |       x         
OnRedirectAsync         |         x          |    x     |      x       |       x         
HttpClientFactory       |         x          |    x     |      x       |                 
FlurlClientFactory      |         x          |          |              |                 

Note that only the *absence* of explicitly setting a value signals to inherit from up the hierarchy. `null` means `null`, not inherit. (Settings are dictionary-backed, and internally the existence of a key dictates whether to inherit, not a null/default value check.)

### Configuring Settings

Settings properties are all read/write, but if you want to change multiple settings at once (atomically), you should generally do so with one of the `Configure*` methods, which take an `Action<Settings>` lambda.

Configure global defaults:

```c#
// call once at application startup
FlurlHttp.Configure(settings => ...);
```

You can also configure the `FlurlClient` that will be used to call a given URL. It is important to note that the same `FlurlClient` is used for all calls to the same _host_ by default, so this will potentially affect more than just calls to the _specific_ URL provided:

```c#
// call once at application startup
FlurlHttp.ConfigureClient(url, cli => ...);
```

If you're managing `FlurlClient`s explicitly:

```c#
flurlClient.Configure(settings => ...);
```

Fluently configure a single request (via extension method on `string`, `Url`, or `IFlurlRequest`):

```c#
await url.ConfigureRequest(settings => ...).GetAsync();
```

Override any settings from within a [test](../testable-http), regardless what level they're set at in the test subject:

```c#
httpTest.Configure(settings => ...);
```

If needed, you can revert settings at any level back to their default or inherited values:

```c#
flurlClient.Settings.ResetDefaults();
FlurlHttp.GlobalSettings.ResetDefaults();
```

Let's take a look at some specific settings.

### HttpClientFactory

(*Not to be confused with .NET Core's [IHttpClientFactory](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.ihttpclientfactory); they are very different things (and Flurl's came first, in case you were wondering ;))*

For advanced scenarios, you can customize the way Flurl.Http constructs `HttpClient` and `HttpMessageHandler` instances. Although it is only required that your custom factory implements `Flurl.Http.Configuration.IHttpClientFactory`, it is recommended to inherit from `DefaultHttpClientFactory` and extend only as needed.

```c#
public class MyCustomHttpClientFactory : DefaultHttpClientFactory
{
    // override to customize how HttpClient is created/configured
    public override HttpClient CreateHttpClient(HttpMessageHandler handler);

    // override to customize how HttpMessageHandler is created/configured
    public override HttpMessageHandler CreateMessageHandler();
}
```

Register your factory globally:

```c#
FlurlHttp.Configure(settings => {
    settings.HttpClientFactory = new MyCustomHttpClientFactory();
});
```

Or (less common) on an individual `FlurlClient`:

```c#
var cli = new FlurlClient(BASE_URL).Configure(settings => {
    settings.HttpClientFactory = new MyCustomHttpClientFactory();
});
```

*A few words of caution:*

1. Overriding `CreateMessageHandler` can be very useful for configuring things like proxies and client certificates, but some features that Flurl has re-implemented, such as cookies and redirects, require that the related settings on `HttpClientHandler` remain _disabled_ in order to function properly. It's a best practice to call `base.CreateMessageHandler()` and configure/return that, rather than creating a new one yourself. If in doubt, [have a look at what Flurl does by default](https://github.com/tmenier/Flurl/blob/dev/src/Flurl.Http/Configuration/DefaultHttpClientFactory.cs) and avoid straying farther from that implementation than necessary.

2. A custom `HttpClientFactory` should be concerned only with _creating_ these objects, not caching/reusing them. That's a concern of `FlurlClientFactory`.

### FlurlClientFactory

`IFlurlClientFactory` defines one method, `Get(Url)`, which is responsible for providing the `IFlurlClient` instance that should be used to call that `Url`. The default implementation uses a single cached instance of `FlurlClient` per combination of the URL's host/scheme/port for the lifetime of your application.

To change this behavior, you could define your own factory by implementing `IFlurlClientFactory` directly, but inheriting from `FlurlClientFactoryBase` is much easier. It allows you to define a caching _strategy_ by returning a cache key based on the `Url`, without having to implement the cache itself.

```c#
public abstract class FlurlClientFactoryBase : IFlurlClientFactory
{
    // override to customize how FlurlClient instances are cached/reused
    protected abstract string GetCacheKey(Url url);

    // override to customize how FlurlClient is created/configured (called only as needed)
    protected virtual IFlurlClient Create(Url url);
}
```

Although the `FlurlClientFactory` configuration setting is only available at the global level, `IFlurlClientFactory` is also [useful](../client-lifetime) in conjunction dependency injection patterns.

### Serializers

Both `JsonSerializer` and `UrlEncodedSerializer` implement `ISerializer`, a simple interface for serializing objects to and from strings.

```C#
public interface ISerializer
{
    string Serialize(object obj);
    T Deserialize<T>(string s);
    T Deserialize<T>(Stream stream);
}
```

Both have a default implementation registered globally, and replacing them is possible but not common. The default `JsonSerializer` implementation is `NewtonsoftJsonSerializer` that, as you probably guessed, uses the ever popular [Json.NET](https://www.newtonsoft.com/json) library. Although it's unlikely that you'd want to _replace_ this implementation, note that its constructor takes a `Newtonsoft.Json.JsonSerializerSettings` argument, which is a nice hook for tapping into the many [custom serialization settings](https://www.newtonsoft.com/json/help/html/SerializationSettings.htm) that library offers:

```c#
FlurlHttp.Configure(settings => {
    var jsonSettings = new JsonSerializerSettings
    {
        NullValueHandling = NullValueHandling.Ignore,
        ObjectCreationHandling = ObjectCreationHandling.Replace
    };
    settings.JsonSerializer = new NewtonsoftJsonSerializer(jsonSettings);
});
```

### Event Handlers

Keeping cross-cutting concerns like logging and error handling separated from your normal logic flow often results in cleaner code. Flurl.Http provides an event model for these scenarios. `BeforeCall`, `AfterCall`, `OnError`, `OnRedirect`, and their `*Async` equivalents are typically defined at the global or client level, but can be defined per request if it makes sense. These settings each take an `Action<HttpCall>` delegate (`Func<HttpCall, Task>` for the async versions). `FlurlCall` provides rich details about the call that you can act upon:

```c#
public class FlurlCall
{
    public IFlurlRequest Request { get; set; }
    public HttpRequestMessage HttpRequestMessage { get; set; }
    public IFlurlResponse Response { get; set; }
    public HttpResponseMessage HttpResponseMessage { get; set; }
    public string RequestBody { get; }
    public FlurlRedirect Redirect { get; set; }
    public FlurlCall RedirectedFrom { get; set; }
    public Exception Exception { get; set; }
    public bool ExceptionHandled { get; set; }
    public DateTime StartedUtc { get; set; }
    public DateTime? EndedUtc { get; set; }
    public TimeSpan? Duration { get; }
    public bool Completed { get; }
    public bool Succeeded { get; }
}
```

Not surprisingly, response-related properties will be `null` in `BeforeCall`. `AfterCall` fires after both successful and failed requests. Setting `ExceptionHandled` to `true` in `OnError` prevents exceptions from bubbling up. Here's an example of registering a global async error handler:

```c#
private async Task HandleFlurlErrorAsync(HttpCall call) {
    await LogErrorAsync(call.Exception.Message);
    call.ExceptionHandled = true;
}

FlurlHttp.Configure(settings => settings.OnErrorAsync = HandleFlurlErrorAsync);
```

### Redirects

Like `HttpClient`, Flurl automatically follows 3XX redirects by default. But the settings and hooks exposed by Flurl offer a greater level of configurability.

```c#
FlurlHttp.Configure(settings => {
    settings.Redirects.Enabled = true; // default true
    settings.Redirects.AllowSecureToInsecure = true; // default false
    settings.Redirects.ForwardAuthorizationHeader = true; // default false
    settings.Redirects.MaxAutoRedirects = 5; // default 10 (consecutive)
});
```

You can also configure redirect behavior on a per-call basis using an event handler:

```c#
flurlClient.OnRedirect(call => {
    if (call.Redirect.Count > 5) {
        call.Redirect.Follow = false;
    }
    else {
        log.WriteInfo($"redirecting from {call.Request.Url} to {call.Redirect.Url}");
        call.Redirect.ChangeVerbToGet = (call.Response.Status == 301);
        call.Redirect.Follow = true;
    }
});
```

If you just need to enable/disable auto-redirect for a single call, you can do it inline:

```c#
await url.WithAutoRedirect(false).GetAsync();
```

