+++
author = "Stephan Schröder"
title = "The Joy of writing helper functions in Kotlin"
date = 2023-09-07T22:31:52+02:00
description = ""
tags = [
    "kotlin", "mockserver"
]
+++

Sometimes it feels unreasonably good to take a perfectly fine Java API and lay a tiny veneer of Kotlin on top of it.

## A minor inconvenience

I was recently tasked to build a test harness using [MockServer](https://www.mock-server.com/) around our proxy server
written in Kotlin.

### MockServer's Java API

So this is how you could use MockServer's `ClientAndServer` class in Java. Since it implements `Closeable`,
I'll use a [try with resources](https://www.baeldung.com/java-try-with-resources) to make sure, the instance is
closed once the test is over.
```java
try(var mockServer = ClientAndServer.startClientAndServer(host, port, port)) {
    mockServer.when(
        request()
            .withSecure(isSecure)
            .withMethod("GET")
            .withPath("/view/cart")
            .withCookies(
                cookie("session", "4930456C-C718-476F-971F-CB8E047AB349")
            )
            .withQueryStringParameters(
                param("cartId", "055CA455-1DF7-45BB-8535-4F83E7266092")
            )
        )
        .respond(
            response()
               .withBody("some_response_body")
               .withStatusCode(201)
        );
    
    // let's assume we defined a small helper function to assemble the url of the mockServer
    String mockServerUrl = assembleUrl(isSecure, host, port);
    
    // add the test code here
}
```
This doesn't look half bad, and we could probably simply translate this 1-to-1 to Kotlin
(`try with resource` will become [use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html), of course)
but there are some tiny details that aren't very kotliny:
- the `when`-method name is just unfortunate, since it's a keyword in Kotlin
- the builder pattern is rare in Kotlin code (because of Kotlin's [scope functions](https://kotlinlang.org/docs/scope-functions.html)),
so seeing it here used so much sure would stick out. So I'd use `apply` whenever I configure request and response.

## putting it all together

Instead of handling all these little issues every time we construct a `ClientAndServer`, let's write a helper method:

```kotlin
inline fun useMockServer(
    isSecure: Boolean = true,
    host: String = "localhost",
    port: Int = 1080,
    crossinline givenRequest: HttpRequest.() -> Unit,
    crossinline returnResponse: HttpResponse.() -> Unit,
    crossinline test: (mockServerUrl: String) -> Unit,
) {
    /** Using @see[port] as local and remote port in the context of the mock server.
        It seems to work as intended, but I might not understand all the implications.*/
    ClientAndServer.startClientAndServer(host, port, port).use { mockServer: ClientAndServer ->
        mockServer.`when`(
            HttpRequest.request().apply{
                givenRequest()
                // setting this option automatically is more convenient, but a bit magic
                withSecure(isSecure)
            }
        ).respond(HttpResponse.response().apply(returnResponse))
        val mockServerUrl = buildString {
            append(if (isSecure) "https" else "http")
            append("://$host:$port")
        }
        test(mockServerUrl)
    }
}
```
The amount of non-trivial Kotlin concepts I managed to cram into this helper function is impressive or scary, depending on from where you look at it. We have: 
- inline functions with crossinline lambda parameters with context receivers
- default values
- trailing commas
- and, as you'll see shortly, we'll use named parameters to make the code more readable.

Not all the concepts used are necessary. The use of `inline` is a bit overkill. `useMockServer` will be used in Unit tests and are therefore not performance critical,
but it's kind of rare to find an opportunity to use `crossinline`, so I couldn't resist.

With all of this using our helper method is very concise and "clean".
Even the part's of the Java Api I continue to use blend right in!
```kotlin
useMockServer(
    isSecure = isSecure,
    host = host,
    port = port,
    givenRequest = {
        withSecure(isSecure)
        withMethod("GET")
        withPath("/view/cart")
        withCookies(
            cookie("session", "4930456C-C718-476F-971F-CB8E047AB349")
        )
        withQueryStringParameters(
            param("cartId", "055CA455-1DF7-45BB-8535-4F83E7266092")
        )
    },
    returnResponse = {
        withBody("some_response_body")
        withStatusCode(201)
    },
) { mockServerUrl ->
    // add the test code here
}
```
And that's the joy of writing and using helper functions in Kotlin.
