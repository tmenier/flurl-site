# Flurl is a modern, fluent, asynchronous, testable, portable, buzzword-laden URL builder and HTTP client library.

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
        first_name = "Frank",
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
    httpTest.RespondWith(200, "OK");
    // act
    await sut.CreatePersonAsync();
    // assert
    httpTest.ShouldHaveCalled("http://api.mysite.com/*")
        .WithVerb(HttpMethod.Post)
        .WithContentType("application/json");
}
```
</div>

<div markdown="1" class="col-md-12 well">

<div markdown="1" class="col-md-4">
### <i class="fa fa-cloud-download"></i> Get It
For just the URL builder, install [Flurl](https://www.nuget.org/packages/Flurl/). For everything else, install [Flurl.Http](https://www.nuget.org/packages/Flurl.Http/).
</div>

<div markdown="1" class="col-md-4">
### <i class="fa fa-book"></i> Learn It
You've come to the right place. Check out the [docs](http://127.0.0.1:8000/docs/fluent-url/).
</div>

<div markdown="1" class="col-md-4">
### <i class="fa fa-stack-overflow"></i> Ask a Question
For programming question related to Flurl, please [ask](http://stackoverflow.com/questions/ask?tags=flurl) on Stack Overflow.
</div>

<div markdown="1" class="col-md-4">
### <i class="fa fa-github"></i> Report an Issue
To report a bug, request a feature, or ask a more general question, [open an issue](https://github.com/tmenier/Flurl/issues/new) on GitHub.
</div>

<div markdown="1" class="col-md-4">
### <i class="fa fa-twitter"></i> Stay Informed
To stay current on releases and other announcements, [follow @FlurlHttp](https://twitter.com/flurlhttp) on Twitter. 
</div>

<div markdown="1" class="col-md-4">
### <i class="fa fa-code-fork"></i> Contribute
Code contributions are welcomed. Please see the [Contribution Guidelines](https://github.com/tmenier/Flurl/wiki/Contribution-Guidelines).
</div>

</div>

<div markdown="1" class="col-md-12">
### Acknowledgements

Flurl is truly a community effort. Special thanks to the following contributors:

- [@kroniak](https://github.com/kroniak) for incredible work on cross-platform support, [automating the build](https://ci.appveyor.com/project/kroniak/flurl/branch/master) and making improvements to the project structure and processes.
- [@lvermeulen](https://github.com/lvermeulen) for creating and maintaining [Flurl.Http.Xml](https://github.com/lvermeulen/Flurl.Http.Xml).
</div>