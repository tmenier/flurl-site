## Fluent URL Building

Flurl's URL builder is best explained with an example:

```c#
using Flurl;

var url = "http://www.some-api.com"
	.AppendPathSegment("endpoint")
	.SetQueryParams(new {
		api_key = ConfigurationManager.AppSettings["SomeApiKey"],
		max_results = 20,
		q = "Don't worry, I'll get encoded!"
	})
	.SetFragment("after-hash");
```

The example above (and most on this site) uses an extension method off `String` to implicitly create a `Url` object. You can do exactly the same explicitly if you prefer:

```c#
var url = new Url("http://www.some-api.com").AppendPathSegment(...
```

As of 3.0, all extension methods available on `String` are also available on `System.Uri`. In either case, you can use the builder methods and convert back to the original representation (using `ToString()` or `ToUri()`) in single fluent call chain.

In addition to the object notation above, `SetQueryParams` also accepts any collection of key-value pairs, Tuples, or a Dictionary object. These alternatives are particularly useful for parameter names that are not valid C# identifiers. There's also a `SetQueryParam` (singular) if you want to set them one by one. In any case, these methods overwrite any previously set values of the same name, but you can set multiple values of the same name by passing a collection:

```c#
var url = "http://www.mysite.com".SetQueryParam("x", new[] { 1, 2, 3 });
Assert.AreEqual("http://www.mysite.com?x=1&x=2&x=3", url)
```

Builder methods and their overloads are highly discoverable, intuitive, and always chainable. A few destructive methods are also included, such as `RemoveQueryParam`, `RemovePathSegment`, and `ResetToRoot`.

### Parsing

In addition to building URLs, `Flurl.Url` is effective at decomposing an existing one:

```c#
var url = new Url("https://user:pass@www.mysite.com:1234/with/path?x=1&y=2#foo");
Assert.AreEqual("https", url.Scheme);
Assert.AreEqual("user:pass", url.UserInfo);
Assert.AreEqual("www.mysite.com", url.Host);
Assert.AreEqual(1234, url.Port);
Assert.AreEqual("user:pass@www.mysite.com:1234", url.Authority);
Assert.AreEqual("https://user:pass@www.mysite.com:1234", url.Root);
Assert.AreEqual("/with/path", url.Path);
Assert.AreEqual("x=1&y=2", url.Query);
Assert.AreEqual("foo", url.Fragment);
```

Although similar to the parsing capabilities of `System.Uri`, Flurl aims to be more compliant with  [RFC 3986](https://tools.ietf.org/html/rfc3986), and more true to the actual string provided, and therefore differs in the following ways:

- `Uri.Query` includes the `?` character; `Url.Query` does not.
- `Uri.Fragment` includes the `#` character; `Url.Fragment` does not.
- `Uri.AbsolutePath` always includes a leading `/` character; `Url.Path` includes it only if it's actually present in the original string, e.g. for `"http://foo.com"`, `Url.Path` is an empty string.
- `Uri.Authority` does not include user info (i.e. `user:pass@`); `Url.Authority` does.
- `Uri.Port` has a default value if not present; `Url.Port` is nullable and is not defaulted.
- `Uri` will make no attempt to parse a relative URL; `Url` assumes that if the string doesn't start with `{scheme}://`, then it starts with a path and parses it accordingly.

`Url.QueryParams` is a special collection type that maintains order and allows duplicate names, but is optimized for the typical case of unique names:

```c#
var url = new Url("https://www.mysite.com?x=1&y=2&y=3");
Assert.AreEqual("1", url.QueryParams.FirstOrDefault("x"));
Assert.AreEqual(new[] { "2", "3" }, url.QueryParams.GetAll("y"));
```

### Mutability

A `Url` is effectively a mutable builder object that implicitly converts to a string. If you need an immutable URL, such as a base URL as a member variable of a class, a common patter is to type it as a `String`. Methods that need to build off of it can do so using string extension methods, thereby implicitly creating a new `Url` object each time (which is cheap) and adding no more keystrokes than if were typed as a `Url` to begin with. The difference is the original string remains unmodified.

Another way to get around the mutable nature of `Url` when needed is to use the `Clone()` method, which simply creates and exact copy of the current `Url`.

### Encoding

Flurl takes care of encoding characters in URLs but takes a different approach with path segments than it does with query string values. The assumption is that query string values are highly variable (such as from user input), whereas path segments tend to be more "fixed" and may already be encoded, in which case you don't want to double-encode. Here are the rules Flurl follows:

- Query string values are fully URL-encoded.
- For path segments, *reserved* characters such as `/` and `%` are *not* encoded.
- For path segments, *illegal* characters such as spaces are encoded.
- For path segments, the `?` character is encoded, since query strings get special treatment.

In some cases, you might want to set a query parameter that you know to be already encoded. `SetQueryParam` has optional `isEncoded` argument:

```c#
url.SetQueryParam("x", "don%27t%20touch%20me", true);
```

While the official URL encoding for the space character is `%20`, it's very common to see `+` used in query parameters. You can tell `Url.ToString` to do this with its optional `encodeSpaceAsPlus` argument:

```c#
var url = "http://foo.com".SetQueryParam("x", "hi there");
Assert.AreEqual("http://foo.com?x=hi%20there", url.ToString());
Assert.AreEqual("http://foo.com?x=hi+there", url.ToString(true));
```

### Utility Methods

`Url` also contains some handy static methods, such as `Combine`, which is basically a [Path.Combine](http://msdn.microsoft.com/en-us/library/dd991142.aspx) for URLs, ensuring one and only one separator character between parts:

```c#
var url = Url.Combine(
    "http://foo.com/",
    "/too/", "/many/", "/slashes/",
    "too", "few?",
    "x=1", "y=2"
// result: "http://www.foo.com/too/many/slashes/too/few?x=1&y=2"
```

And to help you avoid some of the notorious [quirks](https://github.com/tmenier/Flurl/issues/262) associated with the various URL encoding/decoding methods in .NET, Flurl provides some "quirk-free" alternatives:

```c#
Url.Encode(string s, bool encodeSpaceAsPlus); // includes reserved characters like / and ?
Url.EncodeIllegalCharacters(string s, bool encodeSpaceAsPlus); // reserved characters aren't touched
Url.Decode(string s, bool interpretPlusAsSpace);
```
