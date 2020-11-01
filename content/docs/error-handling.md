## Error Handling

By default, Flurl.Http throws a `FlurlHttpException` on any non-2XX HTTP status.

```c#
try {
    await url.PostJsonAsync(poco);
}
catch (FlurlHttpTimeoutException) {
    // FlurlHttpTimeoutException derives from FlurlHttpException; catch here only
    // if you want to handle timeouts as a special case
    LogError("Request timed out.");
}
catch (FlurlHttpException ex) {
    // ex.Message contains rich details, inclulding the URL, verb, response status,
    // and request and response bodies (if available)
    LogError(ex.Message);
}
```

To inspect individual details of the call, or to provide a custom error message, you do not need to parse the `Message` property. `FlurlHttpException` also contains a `Call` property, which is an instance of the same [FlurlCall object](configuration/#event-handlers) used by event handlers, providing a wealth of details about the call.

`FlurlHttpException` also provides shortcuts for deserializing JSON responses:
 
```c#
catch (FlurlHttpException ex) {
    // For error responses that take a known shape
    TError e = await ex.GetResponseJsonAsync<TError>();

    // For error responses that take an unknown shape
    dynamic d = await ex.GetResponseJsonAsync();
}
```

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

You can allow additional HTTP statuses (i.e. prevent throwing) fluently per request:

```c#
url.AllowHttpStatus(HttpStatusCode.NotFound, HttpStatusCode.Conflict).GetAsync();
url.AllowHttpStatus("400-404,6xx").GetAsync();
url.AllowAnyHttpStatus().GetAsync();
```

The pattern in the second example is fairly self-explanatory; allowed characters include digits, commas for separators, hyphens for ranges, and wildcards `x` or `X` or `*`.

You can also override the default behavior at [any settings level](configuration) via `Settings.AllowedHttpStatusRange`, which also takes a string pattern. Note that unlike the fluent methods (which are additive), you must include `2XX` in this setting if you want to allow all statuses in that range.

### Inspecting the Response Before Deserializing

Non-2XX responses tend to be "exceptional", and the `try`/`catch` pattern above is a clean approach to dealing with them. However, if you prefer to check the HTTP status and act as part of the normal control flow, that's easy enough:

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
