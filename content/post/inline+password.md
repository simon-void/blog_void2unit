+++
author = "Stephan Schröder"
title = "Really useful 4 lines of Kotlin code (using inline class)"
date = "2023-02-16"
description = "use an inline class to prevent leaking passwords in your logs"
tags = [
    "kotlin",
    "inline class",
]
+++


So here are 4 lines of code that should probably be part of every Kotlin/JVM project containing authentication:
```kotlin
@JvmInline
value class Password(val value: String) {
    override fun toString() = "***"
}
```

This class is an example of an [inline class](https://kotlinlang.org/docs/inline-classes.html) and is mostly used for additional type safety,
but this specific one increases safety in another way as well.

## How is this useful?

So let's assume you have a config class that contains data on how to connect to some server,
e.g. this [Spring Configuration Properties](https://www.baeldung.com/kotlin/spring-boot-configurationproperties) class:
```kotlin
@ConfigurationProperties(prefix = "someserver")
data class SomeServerConfig @ConstructorBinding constructor(
    val username: String,
    val password: Password,
    val host: String,
    val port: Int,
)
```

The primary advantage of inline classes is that the compiler can now prevent parameter mix-ups between "stringly-types" values
(or other basic types like Int, Long, and such). E.g. imagine a version where `SomeServerConfig.password` was a normal `String`.
In that case this code
```kotlin
fun login(username: String, password: String) {...}

login(config.password, config.username)  // ups, wrong order
```
wouldn't give you a compile-time warning. But if `password` is of a different type, this kind of bug will be caught immediately.

### So how does this version of a password increase security even more?

It's safe to log! Developers often log the contents of there configuration objects
```kotlin
logger.info("using settings: $someServerConfig")
```
This often leaks passwords into log files, which we want to avoid for obvious reasons.
This brings us to the reason we overrode the `toString`-method of our `Password` class. No further leakage of secrets does occur, even
without the programmer doing the logging having to think about it. The log file is clean:
```text
2023-02-16T15:35:05 INFO 26356 --- using settings: SomeServerConfig(username=asmith84, password=***, host=someserver.com, port=34)
```

## But what if the original String value is needed?

Then you use normal property access syntax: `password.value` 

## Why "@JvmInline value class" and not "data class" or even "class"?

The reason to use `@JvmInline value class` is because it's more efficient. Data classes and normal classes lead to an additional pointer indirection
at runtime, which makes the code a tiny bit slower. Inline classes (mostly) disappear completely at runtime (the exception is putting these instances in typed collections and similar generic constructs).
So you get all the additional compiletime safety without additional runtime cost.

## Anything else to know?
- You don't need any additional Spring configuration to deserialization into inline classes instead of basic classes (String, Int, ...).
- Another nifty thing to add to inline classes is validation, so that only validated instances of your data can exist:
```kotlin
@JvmInline
value class Username(val value: String) {
    init {
        require(value.length in 6..20) {"username must be between 6 and 20 characters long, but was ${value.length} in the case of $value"}
    }
}
```