+++
author = "Stephan Schröder"
title = "Kotlin just got Virtual Threads"
date = 2023-09-25T14:34:01+02:00
description = "showcasing Virtual Threads in Kotlin"
tags = [
    "kotlin",
]
+++

It's easy to forget, but whenever Java gains new functionality, so does Kotlin (if it uses a JVM-backend with a JDK >= 21).

## JDK 21 comes with support for Virtual Threads

So Kotlin with a JDK 21 backend has support for [Virtual Treads](https://openjdk.org/jeps/444). E.g. let's look at how we could implement a
`concurrentMap` extension functions:

```kotlin
inline fun <T, R> Iterable<T>.concurrentMap(
    crossinline transform: (T) -> R
): List<R> = Executors.newVirtualThreadPerTaskExecutor().use { execService -> 
    this
        .map { execService.submit<R> { transform(it) } }
        .map { it.get() }
}
```
The nice thing about this function is that the rest of the code doesn't even know that Virtual Threads are used. It can
certainly be optimized, but as an example it will suffice. We can test it with this code:
```kotlin
fun main() {
    println("Let's try out VirtualThreads!")
    val urlsToFetch: List<URI> = listOf(
        URI.create("https://news.ycombinator.com/"),
        URI.create("https://www.heise.de/"),
        URI.create("https://slashdot.org/"),
    )

    val totalTime: Duration = measureTime {
        // the urls are fetched concurrently!
        val timedResults: List<TimedValue<Result<String>>> =
            urlsToFetch.concurrentMap { fetch(it) }

        val sumOfIndividualTimesInMs = timedResults.sumOf {
            it.duration.inWholeMilliseconds
        }
        println("sum of individual execution times: ${sumOfIndividualTimesInMs}ms")
    }
    println("total execution time: ${totalTime.inWholeMilliseconds}ms")
}

// operation with blocking IO
fun fetch(uri: URI): TimedValue<Result<String>> = measureTimedValue {
    runCatching {
        uri.toURL().openStream().use { it.readAllBytes().decodeToString() }
    }
}
```
As expected the total execution time is way lower than the combined execution time its parts.
```text
Let's try out VirtualThreads!
sum of individual execution times: 3359ms
total execution time: 1240ms
```
You can find the repo for this project [here](https://github.com/simon-void/vthreads_with_kotlin_demo).

The minimum version for Kotlin and Gradle to use to be able to compile to JDK 21 is **Kotlin 1.9.20** and **Gradle 8.5**.
(Technically you could already configure Gradlew 8.4 to produce Java21 bytecode via [toolchains](https://docs.gradle.org/8.4/release-notes.html#support-for-building-projects-with-java-21), but the Gradle scripts itself couldn't
be executed on a Java21-JVM, which made the whole setup a bit too cumbersome for my taste.)

## What about Coroutines?

But Kotlin already has [Coroutines](https://kotlinlang.org/docs/coroutines-overview.html). So why would Virtual Threads be used in Kotlin anyway?

Well, both Coroutines and Virtual Threads enable concurrent programming, but according to Kotlin's project lead, both
approaches are optimized for different things and therefor either one can be the appropriate one to use depending on the
use-case. Take a look at his [talk at KotlinConf'23](https://www.youtube.com/watch?v=zluKcazgkV4).

## Conclusion

A rising tide (jdk version) lifts all boats (languages compiling to JVM bytecode).