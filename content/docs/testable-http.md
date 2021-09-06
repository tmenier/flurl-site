## Testable HTTP

Flurl.Http provides a set of testing features that make isolated arrange-act-assert style testing dead simple. At its core is `HttpTest`, the creation of which kicks Flurl into test mode, where all HTTP activity in the test subject is automatically faked and recorded.

````c#
using Flurl.Http.Testing;

[Test]
public void Test_Some_Http_Calling_Method() {
    using (var httpTest = new HttpTest()) {
        // Flurl is now in test mode
        sut.CallThingThatUsesFlurlHttp(); // HTTP calls are faked!
    }
}
````

Most unit testing frameworks have some notion of setup/teardown methods that are executed before/after each test *. For classes with lots of tests against HTTP-calling code, you might prefer this approach:

````c#
private HttpTest _httpTest;

[SetUp]
public void CreateHttpTest() {
    _httpTest = new HttpTest();
}

[TearDown]
public void DisposeHttpTest() {
    _httpTest.Dispose();
}

[Test]
public void Test_Some_Http_Calling_Method() {
    // Flurl is in test mode
}
````

*NOTE: Due to a [known issue](https://github.com/tmenier/Flurl/issues/375) with the mechanism used to signal call faking in the SUT, instantiating `HttpTest` from an **async** setup method will not work.*

### Arrange

By default, fake HTTP calls return a 200 (OK) status with an empty body. Of course you'll likely want to test your code against other responses.

````c#
httpTest.RespondWith("some response body");
sut.DoThing();
````

Use objects for JSON responses:

````c#
httpTest.RespondWithJson(new { x = 1, y = 2 });
````

Test failure conditions:

````c#
httpTest.RespondWith("server error", 500);
httpTest.RespondWithJson(new { message = "unauthorized" }, 401);
httpTest.SimulateTimeout();
````

`RespondWith*` methods are chainable:

````c#
httpTest
    .RespondWith("some response body")
    .RespondWithJson(someObject)
    .RespondWith("error!", 500);
    
sut.DoThingThatMakesSeveralHttpCalls();
````

Behind the scenes, each `RespondWith*` adds a fake response to a thread-safe queue.

Starting in 3.0, you also have the ability to set up responses that only apply to requests that match specific criteria. This example demonstrates all possibilities:

```c#
httpTest
    .ForCallsTo("*.api.com*", "*.test-api.com*") // multiple allowed, wildcard supported
    .WithVerb("put", "PATCH") // or HttpMethod.Put, HttpMethod.Patch
    .WithQueryParam("x", "a*") // value optional, wildcard supported
    .WithQueryParams(new { y = 2, z = 3 })
    .WithAnyQueryParam("a", "b", "c")
    .WithoutQueryParam("d")
    .WithHeader("h1", "f*o") // value optional, wildcard supported
    .WithoutHeader("h2")
    .WithRequestBody("*something*") // wildcard supported
    .WithRequestJson(new { a = "*", b = "hi" }) // wildcard supported in sting values
    .With(call => true) // check anything on the FlurlCall
    .Without(call => false) // check anything on the FlurlCall
    .RespondWith("all conditions met!", 200);
```

Need to make real calls in certain cases?

```c#
httpTest
    .ForCallsTo("https://api.thirdparty.com/*")
    .AllowRealHttp();
```

### Act

Once an `HttpTest` is created and any specific responses are queued, simply call into a test subject. When the SUT makes an HTTP call with Flurl, the real call is effectively blocked and the next fake response is dequeued and returned instead. However, when only one response remains in the queue (matching any filter criteria, if provided), that response becomes "sticky", i.e. it is not dequeued and hence gets returned in all subsequent calls.

There is no need to mock or stub any Flurl objects in order for this to work. `HttpTest` uses the logical asynchronous call context to flow a signal through the SUT and notify Flurl to fake the call.

### Assert

As HTTP calls are faked, they are automatically recorded to a call log, allowing you to assert that certain calls were made. Assertions are test framework-agnostic; they throw an exception at any point when a match is not found as specified, signaling a test failure in virtually all testing frameworks.

`HttpTest` provides a couple assertion methods against the call log:

````c#
sut.DoThing();

// were calls to specific URLs made?
httpTest.ShouldHaveCalled("http://some-api.com/*");
httpTest.ShouldNotHaveCalled("http://other-api.com/*");

// were any calls made?
httpTest.ShouldHaveMadeACall();
httpTest.ShouldNotHaveMadeACalled();
````

You can make further assertions against specific calls, fluently of course:

````c#
httpTest.ShouldHaveCalled("http://some-api.com/*")
    .WithQueryParam("x", "1*")
    .WithVerb(HttpMethod.Post)
    .WithContentType("application/json")
    .WithoutHeader("my-header-*")
    .WithRequestBody("{\"a\":*,\"b\":*}")
    .Times(3);
````

`Times(n)` allows you to assert that the call was made a specific number of times; otherwise, the assertion passes when one or more matching calls were made. In all cases where a name and value can be passed, a `null` value (the default) means ignore and just assert the name. And like with test setup criteria, the `*` wildcard is supported virtually everywhere.

When the `With*` methods don't give you everything you need, you can go down a level and assert the call log directly:

````c#
Assert.That(httpTest.CallLog.Any(call => /* assert anything about the call */));
````

`CallLog` is an `IList<FlurlCall>`. A `FlurlCall` object contains lots of useful information as specified [here](../configuration/#event-handlers).
