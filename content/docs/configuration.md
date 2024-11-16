## Configuration

Flurl.Http includes a robust set of options and techniques for configuring its behavior at various levels.

### Settings

Flurl is configurable primarily through a `Settings` property available on `IFlurlClient`, `IFlurlRequest`, `IFlurlClientBuilder`, and `HttpTest`. Here are the available settings:

| Setting                | Type        | Default Value                 |
| ---------------------- | ----------- | ------------------------------|
| Timeout                | TimeSpan?   | 100 seconds                   |
| HttpVersion            | string      | "1.1"                         |
| AllowedHttpStatusRange | string      | null ("2xx,3xx", effectively) |
| Redirects.Enabled                    | bool | true                   |
| Redirects.AllowSecureToInsecure      | bool | false                  |
| Redirects.ForwardHeaders             | bool | false                  |
| Redirects.ForwardAuthorizationHeader | bool | false                  |
| Redirects.MaxAutoRedirects           | int  | 10                     |
| JsonSerializer         | ISerializer | DefaultJsonSerializer         |
| UrlEncodedSerializer   | ISerializer | UrlEncodedSerializer          |

All properties are read/write and can be set directly:

```cs
// set default on the client:
var client = new FlurlClient("https://some-api.com");
client.Settings.Timeout = TimeSpan.FromSeconds(600);
client.Settings.Redirects.Enabled = false;

// override on the request:
var request = client.Request("endpoint");
request.Settings.Timeout = TimeSpan.FromSeconds(1200);
request.Settings.Redirects.Enabled = true;
```

You can also configure them fluently:

```cs
clientOrRequest.WithSettings(settings => {
    settings.Timeout = TimeSpan.FromSeconds(600);
    settings.AllowedHttpStatusRange = "*";
    settings.Redirects.Enabled = false;
})...
```

Or even more fluently in many cases:

```cs
clientOrRequest
    .WithTimeout(600)
    .AllowAnyHttpStatus()
    .WithAutoRedirect(false)
    ...
```

When using clients, requests, and fluent configuration together, you'll want to pay close attention to which objects you're configuring, as it can determine whether subsequent calls with that client are affected:

```cs
await client
    .WithSettings(...) // configures the client, affects all subsequent requests
    .Request("endpoint") // creates and returns a request
    .WithSettings(...) // configures just this request
    .GetAsync();
```

The clientless pattern supports all the same extension methods. Configure and send a request without ever referencing a client explicitly:

```cs
var result = await "https://some-api.com/endpoint"
    .WithSettings(...) // configures just this request
    .WithTimeout(...) // ditto
    .GetJsonAsync<T>();
```

All of the above settings and extension methods are also available on `IFlurlClientBuilder`, which is useful for configuring things at startup:

```cs
// clientless pattern, all clients:
FlurlHttp.Clients.WithDefaults(builder =>
    builder.WithSettings(...));

// clientless pattern, for a specific site/service:
FlurlHttp.ConfigureClientForUrl("https://some-api.com")
    .WtihSettings(...);

// DI pattern:
services.AddSingleton<IFlurlClientCache>(_ => new FlurlClientCache()
    // all clients:
    .WithDefaults(builder =>
        builder.WithSettings(...))
    // specific named client:
    .Add("MyClient", "https://some-api.com", builder =>
        builder.WithSettings(...))
```

The fourth and final object that supports `Settings` (and all the related fluent goodness) is `HttpTest`, and it takes precedence over everything:

```cs
using var test = new HttpTest.AllowAnyHttpStatus();
await sut.DoThingAsync(); // no matter how things are configured in the SUT,
                          // test settings always win 
```

### Serializers

The `JsonSerializer` and `UrlEncodedSerializer` settings deserve special attention. As you might expect, they control the details of (de)serializing JSON requests and responses, and URL-encoded form posts, respectively. Both implement `ISerializer`, which defines 3 methods:

```cs
string Serialize(object obj);
T Deserialize<T>(string s);
T Deserialize<T>(Stream stream);
```

Flurl provides default implementations for both. It's very unlikely you'll need to use a different `UrlEncodedSerializer`, but there's a couple reasons you may want to swap out the `JsonSerializer`:

1. You prefer the [Newtonsoft](https://www.newtonsoft.com/json)-based version from 3.x and earlier. (4.0 replaced this with a [System.Text.Json](https://learn.microsoft.com/en-us/dotnet/api/system.text.json)-based version.) This is available with the the [Flurl.Http.Newtonsoft](https://www.nuget.org/packages/Flurl.Http.Newtonsoft/) companion package.

2. You want to use the default implementation, but with custom [JsonSerializerOptions](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializeroptions). This can be accomplished by providing your own instance:

```cs
clientOrRequest.Settings.JsonSerializer = new DefaultJsonSerializer(new JsonSerializerOptions {
    PropertyNameCaseInsensitive = true,
    IgnoreReadOnlyProperties = true
});
```
### Event Handlers

Keeping cross-cutting concerns like logging and error handling separated from your normal logic flow often results in cleaner code. Flurl.Http defines 4 events - `BeforeCall`, `AfterCall`, `OnError`, and `OnRedirect` - and an `EventHandlers` collection on `IFlurlClient`, `IFlurlRequest`, and `IFlurlClientBuilder`. (Unlike `Settings`, event handlers are not available on `HttpTest`.)

Similar to `Settings`, everything with an `EventHandlers` property brings with it some fluent shortcuts:

```cs
clientOrRequest
    .BeforeCall(call => DoSomething(call)) // attach a synchronous handler
    .OnError(call => LogErrorAsync(call))  // attach an async handler
```

In the example above, `call` is an instance of `FlurlCall`, which contains a robust set of information and options related to all aspects of the request and response:

```cs
IFlurlRequest Request
HttpRequestMessage HttpRequestMessage
string RequestBody
IFlurlResponse Response
HttpResponseMessage HttpResponseMessage
FlurlRedirect Redirect
Exception Exception
bool ExceptionHandled
DateTime StartedUtc
DateTime? EndedUtc
TimeSpan? Duration
bool Completed
bool Succeeded
```

`OnError` fires before `AfterCall`, and gives you an opportunity to decide whether to allow the exception to bubble up:

```cs
clientOrRequest.OnError(async call => {
    await LogTheErrorAsync(call.Exception);
    call.ExceptionHandled = true; // otherwise, the exeption will bubble up
});
```

`OnRedirect` allows precision handling of 3xx responses:

```cs
clientOrRequest.OnRedirect(call => {
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

At a lower level, event handlers are objects that implement `IFlurlEventHandler`, which defines 2 methods:

```cs
void Handle(FlurlEventType eventType, FlurlCall call);
Task HandleAsync(FlurlEventType eventType, FlurlCall call);
```

Typically you'll only need to implement one or other, so Flurl provides a default implementation, `FlurlEventHanler`, that makes a handy base class - both methods are virtual no-ops, so just override the one you need. Handlers are assignable like this:

```cs
clientOrRequest.EventHandlers.Add((FlurlEventType.BeforeCall, new MyEventHandler()));
```

Note that `EventHanlers` items are of type `Tuple<EventType, IFlurlEventHandler>`. Keeping handlers decoupled from even types means a given handler could be reused for different event types.

One reason you might favor the object-based approach over the simpler lambda-based one described earlier is if you are using DI and your handlers need certain dependencies injected:

```cs
public interface IFlurlErrorLogger : IFlurlEventHandler { }

public class FlurlErrorLogger : FlurlEventHandler, IFlurlErrorLogger
{
    private readonly ILogger _logger;

    public FlurlErrorLogger(ILogger logger) {
        _logger = logger;
    }
}
```

Here's how this might be wired up using Microsoft's DI framework:

```cs
// register ILogger:
services.AddLogging();
// register service that implements IFlurlEventHander and has dependency on ILogger
services.AddSingleton<IFlurlErrorLogger, FlurlErrorLogger>();

// register event hanlder with Flurl, using IServiceProvider to wire up dependencies:
services.AddSingleton<IFlurlClientCache>(sp => new FlurlClientCache()
    .WithDefaults(builder =>
        builder.EventHandlers.Add((FlurlEventType.OnError, sp.GetService<IFlurlErrorLogger>()))
```

### Message Handlers

Flurl.Http is built on top of `HttpClient`, which uses `HttpClientHandler` (by default) for most of its heavy lifting.`IFlurlClientBuilder` exposes methods for configuring both:

```cs
// clientless pattern:
FlurlHttp.Clients.WithDefaults(builder => builder
    .ConfigureHttpClient(hc => ...)
    .ConfigureInnerHandler(hch => {
        hch.Proxy = new WebProxy("https://my-proxy.com");
        hch.UseProxy = true;
    }));

// DI pattern: 
services.AddSingleton<IFlurlClientCache>(_ => new FlurlClientCache()
    .WithDefaults(builder => builder.
        .ConfigureHttpClient(hc => ...)
        .ConfigureInnerHandler(hch => ...)));
```

A few word of caution:

1. Flurl disables `AllowAutoRedirect` and `UseCookies` in order to re-implement its own notion of these features. If you set either to true via `ConfigureInnerHander`, it may break these features in Flurl.
2. Flurl also implements its own notions of `DefaultRequestHeaders` and `Timeout`, likely leaving few reasons that you'd ever need to configure the `HttpClient`.

Perhaps you've read about [SocketsHttpHandler](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.socketshttphandler) and are wondering how to use it in Flurl. You may be surprised to learn that you probably already are. As mentioned, `FlurlClient` delegates most of its work to `HttpClient`, which in turn delegates most of its work to `HttpClientHandler`. But what's lesser known is that, since .NET Core 2.1, `HttpClientHandler` delegates virtually all of its work to `SocketsHttpHandler` on all platforms that support it, which is basically everything except browser-based (i.e. Blazor) platforms. ([Browse the source](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Net.Http/src/System/Net/Http/HttpClientHandler.cs) if you need convincing.)

```txt
FlurlClient →
    HttpClient →
        HttpClientHandler →
            SocketsHttpHandler (on all supported platforms)
```

Still, there is a reason that you may want to bypass `HttpClientHandler` and use `SocketsHttpHander` directly: some of its [configurability](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.socketshttphandler#properties) is not available on `HttpClientHandler`. So long as you don't need to support Blazor, you can do this:

```cs
// clientless pattern:
FlurlHttp.Clients.WithDefaults(builder =>
    builder.UseSocketsHttpHandler(shh => {
        shh.PooledConnectionLifetime = TimeSpan.FromMinutes(10);
        ...
    }));

// DI pattern: 
services.AddSingleton<IFlurlClientCache>(_ => new FlurlClientCache()
    .WithDefaults(builder => builder.UseSocketsHttpHandler(shh => {
        ...
    })));
```

_Note: Calling both `ConfigureInnerHandler` and `UseSocketsHttpHandler` on the same builder will result in a runtime exception. Use one or the other, never both._

The final type of message handler Flurl supports directly is [DelegatingHandler](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.delegatinghandler), a chainable handler type commonly referred to as middleware and often implemented by third-party libraries.

```cs
// clientless pattern:
FlurlHttp.Clients.WithDefaults(builder => builder
    .AddMiddleare(new MyDelegatingHandler()));

// DI pattern: 
services.AddSingleton<IFlurlClientCache>(sp => new FlurlClientCache()
    .WithDefaults(builder => builder
        .AddMiddleware(sp.GetService<IMyDelegatingHandler>())
```

This example uses [Polly](https://github.com/App-vNext/Polly), a popular resiliency library, along with configuring Flurl in a DI scenario:

```cs
using Microsoft.Extensions.Http;

var policy = Policy
    .Handle<HttpRequestException>()
    ...

services.AddSingleton<IFlurlClientCache>(_ => new FlurlClientCache()
    .WithDefaults(builder => builder
        .AddMiddleware(() => new PolicyHttpMessageHandler(policy))));
```

The above example requires installing [Microsoft.Extensions.Http.Polly](https://www.nuget.org/packages/Microsoft.Extensions.Http.Polly/).
