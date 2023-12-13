## Fluent HTTP

*NOTE: Everything beyond URL building and parsing requires installing [Flurl.Http](https://www.nuget.org/packages/Flurl.Http/) rather than the base [Flurl](https://www.nuget.org/packages/Flurl/) package.*

### Basic Usage

A pretty common way to think about interacting with an HTTP service is "I want to build a URL and then call it." Flurl.Http allows you to express that pretty concisely:

```cs
using Flurl;
using Flurl.Http;

var result = await "https://some-api.com"
    .AppendPathSegment("endpoint") 
    .GetStringAsync();
```

Get things other than strings:

```cs
T poco = await "http://api.foo.com".GetJsonAsync<T>();
byte[] bytes = await "http://site.com/image.jpg".GetBytesAsync();
Stream stream = await "http://site.com/music.mp3".GetStreamAsync();
```

The above examples all send an HTTP `GET` request. All other common verbs are also supported:

```cs
var result = await "http://api.foo.com".PostJsonAsync(requestObj).ReceiveJson<T>();
var resultStr = await "http://api.foo.com/1".PatchJsonAsync(requestObj).ReceiveString();
var resultStr2 = await "http://api.foo.com/2".PutStringAsync("hello").ReceiveString();
var resp = await "http://api.foo.com".OptionsAsync();
await "http://api.foo.com".HeadAsync();
```

### Responses and Error Handling

Most of the examples above demonstrate getting some representation of the response _body_ only. So how do you handle a failure response, such as a 400, whose body may take a very different shape? Flurl differs from `HttpClient` in that it throws on any 4xx or greater response status by default, and the `FlurlHttpException` thrown allows you to inspect the body separately:

```cs
try {
    var result = await url.PostJsonAsync(requestObj).ReceiveJson<T>();
}
catch (FlurlHttpException ex) {
    var err = await ex.GetResponseJsonAsync<TError>(); // or GetResponseStringAsync(), etc.
    logger.Write($"Error returned from {ex.Call.Request.Url}: {err.SomeDetails}");
}
```

Of course, you may want to inspect the response status independently of error-handling, or inspect other response properties such as headers. Using `GetAsync`, or just excluding the `ReceiveXXX`, from non-GET calls, returns an `IFlurlResponse`:

```cs
var resp1 = await "http://api.foo.com".GetAsync();
var resp2 = await "http://api.foo.com".PostJsonAsync(requestObj);
```

From which you can read other information before consuming the body:

```cs
int status = resp.StatusCode;
string headerVal = resp.Headers.FirstOrDefault("my-header");
T body = await resp.GetJsonAsync<T>();
```

If you prefer inspecting status codes this way exclusively, you can disable the exception-throwing behavior for specific statuses or all of them:

```cs
var resp2 = await "http://api.foo.com".AllowHttpStatus(400, 401).GetAsync();
var resp3 = await "http://api.foo.com".AllowHttpStatus("400-403,5xx").GetAsync();
var resp1 = await "http://api.foo.com".AllowAnyHttpStatus().GetAsync();
```

In the last example, `x`, `X`, and `*` are all valid wildcards. Note that you do not need to include 2xx or 3xx codes explicitly; those are considered "success" statuses and never throw.

### Simulating a Browser

Flurl includes first-class support for things like form posts and cookies, which are more typical with web browsers than REST APIs.

Simulate an HTML form post:

```cs
await "http://site.com/login".PostUrlEncodedAsync(new { 
    user = "user", 
    pass = "pass"
});
```

Or a multipart form POST (typically associated with file uploads):

``` c#
var resp = await "http://api.com".PostMultipartAsync(mp => mp
    .AddString("name", "hello!")                // individual string
    .AddStringParts(new {a = 1, b = 2})         // multiple strings
    .AddFile("file1", path1)                    // local file path
    .AddFile("file2", stream, "foo.txt")        // file stream
    .AddJson("json", new { foo = "x" })         // json
    .AddUrlEncoded("urlEnc", new { bar = "y" }) // URL-encoded                      
    .Add(content));                             // any HttpContent
```

Download a file:

```cs
// filename is optional here; it will default to the remote file name
var path = await "http://files.foo.com/image.jpg"
    .DownloadFileAsync("c:\\downloads", filename);
```

Send some cookies with a request:

```cs
var resp = await "https://cookies.com"
    .WithCookie("name", "value")
    .WithCookies(new { cookie1 = "foo", cookie2 = "bar" })
    .GetAsync();
```

Better yet, grab response cookies from the first request and let Flurl determine when to send them back (per [RFC 6265](https://tools.ietf.org/html/rfc6265)):

```cs
await "https://cookies.com/login".WithCookies(out var jar).PostUrlEncodedAsync(credentials);
await "https://cookies.com/a".WithCookies(jar).GetAsync();
await "https://cookies.com/b".WithCookies(jar).GetAsync();
```

Or avoid all those `WithCookies` calls and use a `CookieSession`:

```cs
using var session = new CookieSession("https://cookies.com");
// set any initial cookies on session.Cookies
await session.Request("a").GetAsync();
await session.Request("b").GetAsync();
// read cookies at any point using session.Cookies
```

In the above examples, `jar` and `session.Cookies` are instances of `CookieJar`. This is Flurl's equivalent of `CookieContainer` from the `HttpClient` stack, but with one major advantage: it is not bound to an `HttpMessageHandler`, hence you can simulate multiple cookie "sessions" on a single `HttClient/Handler` instance. It can also be easily persisted and reloaded between sessions:

```cs
// string-based persistence:
var saved = jar.ToString();
var jar2 = CookieJar.LoadFromString(saved);

// file-based persistence:
using var writer = new StreamWriter("path/to/file");
jar.WriteTo(writer);

using var reader = new StreamReader("path/to/file");
var jar2 = CookieJar.LoadFrom(reader);
```

### Additional Use Cases

Set request headers:

```cs
// one:
await url.WithHeader("Accept", "text/plain").GetJsonAsync();
// multiple:
await url.WithHeaders(new { Accept = "text/plain", User_Agent = "Flurl" }).GetJsonAsync();
```

In the second example above, `User_Agent` will automatically render as `User-Agent` in the header name. Hyphens are very common in header names but not allowed in C# identifiers; underscores, just the opposite.

Authenticate using [Basic authentication](https://en.wikipedia.org/wiki/Basic_access_authentication):

```cs
await url.WithBasicAuth("username", "password").GetJsonAsync();
```

Or an [OAuth 2.0 bearer token](https://tools.ietf.org/html/rfc6750):

```cs
await url.WithOAuthBearerToken("mytoken").GetJsonAsync();
```

Specify a timeout:

```cs
await url.WithTimeout(10).GetAsync(); // 10 seconds
await url.WithTimeout(TimeSpan.FromMinutes(2)).GetAsync();
```

Handle a timeout error:

```cs
try {
    var result = await url.GetStringAsync();
}
catch (FlurlHttpTimeoutException) {
    // handle timeouts
}
catch (FlurlHttpException) {
    // handle error responses
}
```

`FlurlHttpTimeoutException` inherits from `FlurlHttpException` and hence could be handled from the same `catch` block, but the above example demonstrates handling it as a special case.

Weird verbs or content? Use one of the lower-level methods:

```cs
await url.PostAsync(content); // content is a System.Net.Http.HttpContent
await url.SendJsonAsync(HttpMethod.Trace, data);
await url.SendAsync(
    new HttpMethod("CONNECT"),
    httpContent, // optional
    cancellationToken,  // optional
    HttpCompletionOption.ResponseHeaderRead);  // optional
```

Cancel a request:
```cs
var cts = new CancellationTokenSource();
var task = url.GetAsync(cts.Token);
...
cts.Cancel();
```
