## Fluent HTTP

*NOTE: Everything beyond URL building and parsing requires installing [Flurl.Http](https://www.nuget.org/packages/Flurl.Http/) rather than the base [Flurl](https://www.nuget.org/packages/Flurl/) package.*

A pretty common way to think about interacting with an HTTP service is "I want to build a URL and then call it." Flurl.Http allows you to express that pretty concisely:

```c#
using Flurl;
using Flurl.Http;

var result = await baseUrl.AppendPathSegment("endpoint").GetAsync();
```

The above code sends an HTTP `GET` request and returns an `IFlurlResponse`, from which you can get properties such as `StatusCode`, `Headers`, and the body content via methods such as `GetStringAsync` and `GetJsonAsync<T>`.

But often you just want to jump straight to the body, and Flurl provides a variety of shortcuts to do that:

```c#
T poco = await "http://api.foo.com".GetJsonAsync<T>();
string text = await "http://site.com/readme.txt".GetStringAsync();
byte[] bytes = await "http://site.com/image.jpg".GetBytesAsync();
Stream stream = await "http://site.com/music.mp3".GetStreamAsync();
```

Download a file with ease:

```c#
// filename is optional here; it will default to the remote file name
var path = await "http://files.foo.com/image.jpg"
    .DownloadFileAsync("c:\\downloads", filename);
```

Other "read" verbs:

```c#
var headResponse = await "http://api.foo.com".HeadAsync();
var optionsResponse = await "http://api.foo.com".OptionsAsync();
```

Then there's the "write" verbs:

```c#
await "http://api.foo.com".PostJsonAsync(new { a = 1, b = 2 });
await "http://api.foo.com/1".PatchJsonAsync(new { c = 3 });
await "http://api.foo.com/2".PutStringAsync("hello");
```

All of the methods above return a `Task<IFlurlResponse>`. You may of course expect some data to be returned in the response body:

```c#
T poco = await url.PostAsync(content).ReceiveJson<T>();
string s = await url.PatchJsonAsync(partial).ReceiveString();
```

Weird verbs or content? Use one of the lower-level methods:

```c#
await url.PostAsync(content); // a System.Net.Http.HttpContent object
await url.SendJsonAsync(HttpMethod.Trace, data);
await url.SendAsync(
    new HttpMethod("CONNECT"),
    httpContent, // optional
    cancellationToken,  // optional
    HttpCompletionOption.ResponseHeaderRead);  // optional
```

Set request headers:

```c#
// one:
await url.WithHeader("Accept", "text/plain").GetJsonAsync();
// multiple:
await url.WithHeaders(new { Accept = "text/plain", User_Agent = "Flurl" }).GetJsonAsync();
```

In the second example above, `User_Agent` will automatically render as `User-Agent` in the header name. (Hyphens are very common in header names but not allowed in C# identifiers; underscores, just the opposite.)

Specify a timeout:

```c#
await url.WithTimeout(10).DownloadFileAsync(); // 10 seconds
await url.WithTimeout(TimeSpan.FromMinutes(2)).DownloadFileAsync();
```

Cancel a request:
```c#
var cts = new CancellationTokenSource();
var task = url.GetAsync(cts.Token);
...
cts.Cancel();
```

Authenticate using [Basic authentication](https://en.wikipedia.org/wiki/Basic_access_authentication):

```c#
await url.WithBasicAuth("username", "password").GetJsonAsync();
```

Or an [OAuth 2.0 bearer token](https://tools.ietf.org/html/rfc6750):

```c#
await url.WithOAuthBearerToken("mytoken").GetJsonAsync();
```

Simulate an HTML form post:

```c#
await "http://site.com/login".PostUrlEncodedAsync(new { 
    user = "user", 
    pass = "pass"
});
```

Or a `multipart/form-data` POST:

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

Send some cookies with a request:

```c#
var resp = await "https://cookies.com"
    .WithCookie("name", "value")
    .WithCookies(new { cookie1 = "foo", cookie2 = "bar" })
    .GetAsync();
```

Better yet, grab response cookies from the first request and let Flurl determine when to send them back (per [RFC 6265](https://tools.ietf.org/html/rfc6265)):

```c#
await "https://cookies.com/login".WithCookies(out var jar).PostUrlEncodedAsync(credentials);
await "https://cookies.com/a".WithCookies(jar).GetAsync();
await "https://cookies.com/b".WithCookies(jar).GetAsync();
```

Or avoid all those `WithCookies` calls and use a `CookieSession`:

```c#
using (var session = new CookieSession("https://cookies.com")) {
    // set any initial cookies on session.Cookies
    await session.Request("a").GetAsync();
    await session.Request("b").GetAsync();
    // read cookies at any point using session.Cookies
}
```

A `CookieJar` can also be created/modified explicitly, which may be useful in re-hydrating cookies that were persisted:

```c#
var jar = new CookieJar()
    .AddOrUpdate("cookie1", "foo", "https://cookies.com") // you must specify the origin URL
    .AddOrUpdate("cookie2", "bar", "https://cookies.com");

await "https://cookies.com/a".WithCookies(jar).GetAsync();
```

`CookieJar` is Flurl's equivalent of `CookieContainer` from the `HttpClient` stack, but with one major advantage: it is not bound to an `HttpMessageHandler`, hence you can simulate multiple cookie "sessions" on a single `HttClient/Handler` instance.