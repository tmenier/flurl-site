## Error Handling

Unlike `HttpClient`, Flurl.Http throws on any non-2XX HTTP status by default. Here's the reasoning:

1. Non-2XX conditions tend to be "exceptional", that is, they're not expected them under "normal" circumstances and logic flow, hence they fit the `try`/`catch` paradigm.

2. Especially in JSON APIs, error response bodies tend to take a different shape than regular responses, and if you're using shortcuts like `url.GetJsonAsync<RegularShape>()`, Flurl's `try`/`catch` pattern provides a way to deserialize to something different in the `catch` block.

```c#
try {
    var result = await url.PostJsonAsync(poco).ReceiveJson<T>();
}
catch (FlurlHttpException ex) {
    var error = await ex.GetResponseJsonAsync<TError>();
    logger.Write($"Error returned from {ex.Call.Request.Url}: {error.SomeDetails}");
}
```

The `Call` property above is an instance of the same [FlurlCall object](configuration/#event-handlers) used by event handlers, providing a wealth of details about the call. For simple logging and debugging, `FlurlHttpException.Message` gives you a handy summary of the error, including the URL, HTTP verb, and status code received.

`FlurlHttpException` also gives you a few shortcuts for deserializing the body:
 
```c#
Task<string> GetResponseStringAsync();
Task<T> GetResponseJsonAsync<T>();
Task<dynamic> GetResponseJsonAsync();
```

These are all short-hand for equivalent methods on `FlurlHttpException.Call.Response`, so you can go that route if you need something different, such as a stream.

### Timeouts

Flurl.Http defines a special exception type for timeouts: `FlurlHttpTimeoutException`. This type inherits from `FlurlHttpException`, and hence will get caught in a `catch (FlurlHttpException)` block. But you may want to handle timeouts differently:

```c#
catch (FlurlHttpTimeoutException) {
    // handle timeout
}
catch (FlurlHttpException) {
    // handle error response
}
```

`FlurlHttpTimeoutException` has no additional properties other than those in `FlurlHttpException`, but because a timeout implies that no response was received, all response-related properties will always be `null`.

The default timeout is 100 seconds (same as `HttpClient`), but this can be [configured](configuration) at any settings level, or inline per request:

```c#
await url.WithTimeout(200).GetAsync(); // 200 seconds
await url.WithTimeout(TimeSpan.FromMinutes(10)).GetAsync();
```

### Allowing Non-2XX Responses

If you don't like the default throwing behavior, you can change it at [any settings level](configuration) via `Settings.AllowedHttpStatusRange`. This is a string based setting that accepts wildcards, so if you never want to throw, set it to `*`.

You can also allow non-2XX at the request level:

```c#
url.AllowHttpStatus("400-404,6xx").GetAsync();
url.AllowAnyHttpStatus().GetAsync();
```

The pattern in the first example is fairly self-explanatory. Allowed characters include digits, commas for separators, hyphens for ranges, and wildcards `x` or `X` or `*`. These syntax rules are the same for `Settings.AllowedHttpStatusRange`, but there's one subtle behavioral difference: the request-level methods above are _additive_, so for example you don't have to include `2XX` in the request-level pattern if it's already allowed per the settings.

### Inspecting the Response Before Deserializing

If you prefer to handle non-2XX as part of the normal control flow, that's easy enough:

```c#
var response = await url
    .AllowAnyHttpStatus()
    .GetAsync();

if (result.StatusCode < 300) {
    var result = await response.GetJsonAsync<T>();
    Console.WriteLine($"Success! {result}")
}
else if (result.StatusCode < 500) {
    var error = await response.GetJsonAsync<UserErrorData>();
    Console.WriteLine($"You did something wrong! {error}")
}
else {
    var error = await response.GetJsonAsync<ServerErrorData>();
    Console.WriteLine($"We did something wrong! {error}")
}
```

Here `response` is an instance of `IFlurlResponse`, which wraps (and exposes) a `System.Net.Http.HttpResponseMessage`. In addition to `StatusCode`, you can inspect `Headers`, `Cookies`, and get the body content in a variety of ways:

```c#
Task<T> GetJsonAsync<T>();
Task<dynamic> GetJsonAsync();
Task<IList<dynamic>> GetJsonListAsync();
Task<string> GetStringAsync();
Task<Stream> GetStreamAsync();
Task<byte[]> GetBytesAsync();
```
