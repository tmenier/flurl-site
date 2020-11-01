## Extensibility

Since most of Flurl's functionality is provided through extension methods, it is very easy to extend using the same patterns that Flurl itself uses.

### Extending the URL builder

Chainable URL builder methods generally come in sets of 3 overloads: one extending `Flurl.Url`, one extending `System.Uri`, and one extending `String`. All of these should return the modified `Flurl.Url` object:

```c#
public static Url DoMyThing(this Url url) {
    // do something interesting with url
    return url;
}

// keep these overloads DRY by constructing a Url and deferring to the above method
public static Url DoMyThing(this Uri uri) => new Url(uri).DoMyThing(); 
public static Url DoMyThing(this string url) => new Url(url).DoMyThing();
```

### Extending Flurl.Http

Chainable Flurl.Http extension methods generally come in sets of 4, extending `Flurl.Url`, `System.Uri`, `String`, and `IFlurlRequest`. All should return the current `IFlurlRequest` to allow further chaining.

```c#
public static IFlurlRequest DoMyThing(this IFlurlRequest req) {
    // do something interesting with req.Settings, req.Headers, req.Url, etc.
    return req;
}

// keep these overloads DRY by constructing a Url and deferring to the above method
public static IFlurlRequest DoMyThing(this Url url) => new FlurlRequest(url).DoMyThing();
public static IFlurlRequest DoMyThing(this Uri uri) => new FlurlRequest(uri).DoMyThing();
public static IFlurlRequest DoMyThing(this string url) => new FlurlRequest(url).DoMyThing();
```

Now all of these work:

```c#
result = await "http://api.com"
    .DoMyThing() // string extension
    .GetAsync();

result = "http://api.com"
    .AppendPathSegment("endpoint")
    .DoMyThing() // Url extension
    .GetAsync();

result = "http://api.com"
    .AppendPathSegment("endpoint")
    .WithBasicAuth(u, p)
    .DoMyThing() // IFlurlRequest extension
    .GetAsync();
```

There are cases where you may want yet a fifth overload: an `IFlurlClient` extension. If your extension interacts only with `Settings` or `Headers`, recall that defaults for these exist at the client level, so for completeness it might make sense for your extension to support client-level defaults as well.

