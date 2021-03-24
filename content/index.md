# Flurl is a modern, fluent, asynchronous, testable, portable, buzzword-laden URL builder and HTTP client library for .NET.

<div markdown="1" class="col-md-6">
## Code It
```c#
// Flurl will use 1 HttpClient instance per host
var person = await "https://api.com"
    .AppendPathSegment("person")
    .SetQueryParams(new { a = 1, b = 2 })
    .WithOAuthBearerToken("my_oauth_token")
    .PostJsonAsync(new
    {
        first_name = "Claire",
        last_name = "Underwood"
    })
    .ReceiveJson<Person>();
```
</div>

<div markdown="1" class="col-md-6">
## Test It
```c#
// fake & record all http calls in the test subject
using (var httpTest = new HttpTest()) {
    // arrange
    httpTest.RespondWith("OK", 200);
    // act
    await sut.CreatePersonAsync();
    // assert
    httpTest.ShouldHaveCalled("https://api.com/*")
        .WithVerb(HttpMethod.Post)
        .WithContentType("application/json");
}
```
</div>

<div markdown="1" class="col-md-12">
### Flurl is available on [NuGet](https://www.nuget.org/packages/Flurl.Http/) and is free for commercial use. It runs on a wide variety of platforms, including .NET Framework, .NET Core, Xamarin, and UWP.
</div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-cloud-download"></i> Get It</h3>
<p>For just the URL builder, install <a href="https://www.nuget.org/packages/Flurl/">Flurl</a>. For everything else, install <a href="https://www.nuget.org/packages/Flurl.Http/">Flurl.Http</a>.</p>
</div></div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-book"></i> Learn It</h3>
<p>You've come to the right place. Check out the <a href="docs/fluent-url/">docs</a>.</p>
</div></div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-stack-overflow"></i> Ask a Question</h3>
<p>For programming questions related to Flurl, please <a href="http://stackoverflow.com/questions/ask?tags=flurl">ask</a> on Stack Overflow.</p>
</div></div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-github"></i> Report an Issue</h3>
<p>To report a bug or request a feature, <a href="https://github.com/tmenier/Flurl/issues/new/choose">open an issue</a> on GitHub.</p>
</div></div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-twitter"></i> Stay Informed</h3>
<p>To stay current on releases and other announcements, <a href="https://twitter.com/flurlhttp">follow @FlurlHttp</a>.</p>
</div></div>

<div class="col-md-4"><div class="well">
<h3><i class="fa fa-code-fork"></i> Contribute</h3>
<p>To contribute code, please see the <a href="https://github.com/tmenier/Flurl/wiki/Contribution-Guidelines">contribution guidelines</a>.</p>
</div></div>
