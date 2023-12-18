## Fluent URL Building

Flurl was born as a modest URL builder. While most of the capabilities described on this site require [Flurl.Http](https://www.nuget.org/packages/Flurl.Http), the builder is still available as a [stand-alone package](https://www.nuget.org/packages/Flurl) if it's all you need.

Here's the builder in action:

```cs
using Flurl;

var url = "https://some-api.com"
	.AppendPathSegment("endpoint")
	.SetQueryParams(new {
		api_key = _config.GetValue<string>("MyApiKey"),
		max_results = 20,
		q = "I'll get encoded!"
	})
	.SetFragment("after-hash");

// result:
// https://some-api.com/endpoint?api_key=xxx&max_results=20&q=I%27ll%20get%20encoded%21#after-hash
```

This example (and most on this site) uses a string extension method to implicitly create a `Flurl.Url` object. This object converts back to a string implicitly, meaning you can pass a `Url` object to any method that takes a string without explictly invoking `ToString()`. Combined, these features allow you to manipulate URL strings in a structured manner without ever enlisting the builder object explicitly.

Of course you may still create a `Url` object explicitly if you prefer:

```cs
var url = new Url("http://www.some-api.com").AppendPathSegment(...
```

All string extension methods are also available on `System.Uri`.

The first example demonstrated setting query parameters using object notation, where property names are map to parameter names. There are other approaches available:

```cs
url.SetQueryParam("name", "value"); // one by one
url.SetQueryParams(dictionary); // any IDictionary<string, object>
url.SetQueryParams(kvPairs); // any IEnumerable<KeyValuePair>
url.SetQueryParams(new[] {(name1, value1), (name2, value2), ...}); // any collection of Tuples
url.SetQueryParams(new[] {
	new { name = "foo", value = 1 }, ...}); // virtually anything resembling name/value pairs
```

These alternatives are particularly useful when parameter names are variable or not valid C# identifiers.

`SetQueryParam(s)` overwrites any previously set values of the same name, but you can set multiple values of the same name by passing a collection:

```cs
"https://some-api.com".SetQueryParam("x", new[] { 1, 2, 3 }); // https://some-api.com?x=1&x=2&x=3
```

To add multiple values in multiple steps without overwriting, just use `AppendQueryParam(s)` instead:

```cs
"https://some-api.com"
	.AppendQueryParam("x", 1);
	.AppendQueryParam("x", 2);
	.AppendQueryParams("x", new[] { 3, 4 }); // https://some-api.com?x=1&x=2&x=3&x=4
```

Builder methods and their overloads are highly discoverable, intuitive, and always chainable. A few destructive methods are also included, such as `RemoveQueryParam`, `RemovePathSegment`, and `ResetToRoot`.

### Parsing

In addition to building URLs, `Flurl.Url` is effective at decomposing an existing one:

```cs
var url = new Url("https://user:pass@www.mysite.com:1234/with/path?x=1&y=2#foo");
Assert.Equal("https", url.Scheme);
Assert.Equal("user:pass", url.UserInfo);
Assert.Equal("www.mysite.com", url.Host);
Assert.Equal(1234, url.Port);
Assert.Equal("user:pass@www.mysite.com:1234", url.Authority);
Assert.Equal("https://user:pass@www.mysite.com:1234", url.Root);
Assert.Equal("/with/path", url.Path);
Assert.Equal("x=1&y=2", url.Query);
Assert.Equal("foo", url.Fragment);
```

In addition, `Url.QueryParams` is a special collection type that maintains order and allows duplicate names, but is optimized for the typical case of unique names:

```cs
var url = new Url("https://www.mysite.com?x=1&y=2&y=3");
Assert.Equal("1", url.QueryParams.FirstOrDefault("x"));
Assert.Equal(new[] { "2", "3" }, url.QueryParams.GetAll("y"));
```

Although its parsing capabilities are similar to those of of `System.Uri`, Flurl aims to be more compliant with  [RFC 3986](https://tools.ietf.org/html/rfc3986), and more true to the actual string provided, and therefore differs in the following ways:

- `Uri.Query` includes the `?` character; `Url.Query` does not.
- `Uri.Fragment` includes the `#` character; `Url.Fragment` does not.
- `Uri.AbsolutePath` always includes a leading `/` character; `Url.Path` includes it only if it's actually present in the original string, e.g. for `"http://foo.com"`, `Url.Path` is an empty string.
- `Uri.Authority` does not include user info (i.e. `user:pass@`); `Url.Authority` does.
- `Uri.Port` has a default value if not present; `Url.Port` is nullable and is not defaulted.
- `Uri` will make no attempt to parse a relative URL; `Url` assumes that if the string doesn't start with `{scheme}://`, then it starts with a path and parses it accordingly.

### Mutability

A `Url` is effectively a mutable builder object that implicitly converts to a string. If you need an immutable URL, such as a base URL as a member variable of a class, a common pattern is to type it as a `String`:

```cs
public class MyServiceClass
{
	private readonly string _baseUrl;

	public Task CallServiceAsync(string endpoint, object data) {
		return _baseUrl
			.AppendPathSegment(endpoint)
			.PostAsync(data); // requires Flurl.Http package
	}
}
```

Here the call to `AppendPathSegment` creates a new `Url` object. The result is that `_baseUrl` remains unmodified, and you've added no additional "noise" compared to if you had declared it as a `Url`.

Another way to get around the mutable nature of `Url` when needed is to use the `Clone()` method:

```cs
var url2 = url1.Clone().AppendPathSegment("next");
```

Here you get a new `Url` object based on another, so you can modify it without changing the original.

### Encoding

Flurl takes care of encoding characters in URLs, but it takes a different approach with path segments than it does with query string values. The assumption is that query string values are highly variable (such as from user input), whereas path segments tend to be more "fixed" and may already be encoded, in which case you don't want to double-encode. Here are the rules Flurl follows:

- Query string values are fully URL-encoded.
- For path segments, *reserved* characters such as `/` and `%` are *not* encoded.
- For path segments, *illegal* characters such as spaces are encoded.
- For path segments, the `?` character is encoded, since query strings get special treatment.

In some cases, you might want to set a query parameter that you know to be already encoded. `SetQueryParam` has optional `isEncoded` argument:

```cs
url.SetQueryParam("x", "I%27m%20already%20encoded", true);
```

While the official URL encoding for the space character is `%20`, it's very common to see `+` used in query parameters. You can tell `Url.ToString` to do this with its optional `encodeSpaceAsPlus` argument:

```cs
var url = "http://foo.com".SetQueryParam("x", "hi there");
Assert.Equal("http://foo.com?x=hi%20there", url.ToString());
Assert.Equal("http://foo.com?x=hi+there", url.ToString(true));
```

### Utility Methods

`Url` also contains some handy static methods, such as `Combine`, which is basically a [Path.Combine](http://msdn.microsoft.com/en-us/library/dd991142.aspx) for URLs, ensuring one and only one separator character between parts:

```cs
var url = Url.Combine(
    "http://foo.com/",
    "/too/", "/many/", "/slashes/",
    "too", "few?",
    "x=1", "y=2");
// result: "http://www.foo.com/too/many/slashes/too/few?x=1&y=2"
```

Check if a given string is a well-formed absolute URL:

```cs
if (!Url.IsValid(input))
	throw new Exception("You entered an invalid URL!");
```

And to help you avoid some of the notorious [quirks](https://github.com/tmenier/Flurl/issues/262) associated with the various URL encoding/decoding methods in .NET, Flurl provides some "quirk-free" alternatives:

```cs
Url.Encode(string s, bool encodeSpaceAsPlus); // includes reserved characters like / and ?
Url.EncodeIllegalCharacters(string s, bool encodeSpaceAsPlus); // reserved characters aren't touched
Url.Decode(string s, bool interpretPlusAsSpace);
```
