+++
author = "Stephan Schröder"
title = "Safe Null-Safety or Null at the Gates"
date = 2023-05-22T21:10:19+01:00
description = "All 'harmfull' nulls should be dealt with at the edge between systems that allow nullability and Kotlin"
tags = [
    "kotlin",
]
draft = true
+++

# Safe Null-Safety or Null at the Gates

Since I learned about non-nullable types, I don't like to work in languages that don't offer them in one way or another
(non-nullable types and sealed classes actually).
This is why I tend to get puzzled when I encounter developers (or their code) who clearly didn't come to the same conclusion.
Sometimes you hear sentences like "Nullability is overrated", sometimes it is deeper layers of the programm being littered with the not-null assertion operator `!!`.

This article is about how I use null-safety to extract its maximum value.

## TLDR

The aim of safe null-safety is to get rid of nullable types as early as possible, which is at the border between parts of
the system that do have non-nullability information attached to it (Kotlin code, SQL databases, GraphQL, ...) and those
parts that don't (Java code without nullability annotations, NoSQL databases, REST, ...)

## The slightly longer version

There are only two types of null values, expected ones ("not every Person instance has an Address linked to it, and this instance hasn't")
and unexpected ones ("every Person should have a first name, but this entry from the NoSQL db comes without one").

Whenever you see a random null-assertion sprinkled in the codebase, that means one of two things:
- in the case of an "unexpected null" there was a failure to block the null from entering deeper into the codebase by declaring that variable/property non-null.
- in the case of an "expected null" it should be clear how to handle it. There should be a fallback value/behaviour available to handle this case.

The aim of proper null-safety is to get rid of unexpected nulls as early as possible.
You should only ever find null-assertion statements of whatever kind in the validation layer of data at the border
to null-unaware datasource (or in tests).

### Discipline

There isn't really a big idea in this blog post, the big idea are non-nullable types on their own. The point of this
blog post is that you have to be disciplined to see this all the way through, even when things don't stay as easy as
simply putting non-nullable annotations on your DTOs.

I've heard of a Kotlin REST endpoint were every property of the DTO was made nullable, because the data sent actually came
from a NoSQL database. Since no guarantees can be made in a NoSQL db, no non-nullability guarantees where given inside
the Kotlin app. Whoever wrote this code clearly forgot that 
**every database, even if it is called schemaless, has a schema**. After all, pure random data isn't useful (as a source)
of information. The better approach would have been to make the implicit schema of the data contained in the db explicit.

#### an organically grown database might have several schema, actually

The primary schema of a database might change over time leaving datasets of the older schemas within the database.
E.g. imagine having a NoSQL customer database that is used to contain only the name and email-address of a customer, but
nowadays also the address and telephone number of the customer are stored. As long as you only have two overlaying schemas,
using nullable properties seems sensible, at least as first, until some devs will realize that the presence of one property
(let's say the telephone number) does imply the presence of another property (the address). 
I think what we've reached here as soon as more than one implicit schema is present in our database is the limit of
normal single class validation. Instead of validating that the data conforms to one schema with several nullable properties,
actually we still should do that as a first step, we should than parse this "raw domain object" into a "sealed domain object"
with one child per implicit database. **If your data has more than one schema, make it explicit!**

```kotlin
sealed class Customer(
    val name: String,
    val email: ValidEmail, // ValidEmail is a value class wrapping String with some additional validation
) {
    class V1(
        name: String,
        email: ValidEmail,
    ): Customer (name, email)
    
    class V2(
        name: String,
        email: ValidEmail,
        val address: Address,
        val phoneNumber: ValidPhoneNr,
    ): Customer (name, email)
}
```

Different Services of you class can now reject whole versions of the Customer domain object. A Single check if you're
in a certain subclass is necessary, smart-cast will do the rest, no need for superfluous null-checks or not-null-assertion.

## Lower hanging Fruits

But maybe that's all going too far, and the real issue is that (parts of) your team only recently switch from Java to
Kotlin. In that case there might be some valuable lessons to be substantiated. It should go without saying, but everything that can be non-nullable, should be non-nullable. This not only holds for
properties but also variable. Kotlin being expression-oriented instead of statement-oriented goes a long way in this regard.


### avoid initialising variables with null

A typical pattern for variable initialisation in Java looks like this:

```java
Object obj;
if(...) {
    obj = $expression1;
} else {
    obj = $expression2;
}
```

If you run the IntelliJ's Java-to-Kotlin transpiler over it, it'll produce:

```kotlin
var obj: Any? = null
if(...) {
    obj = $expression1
} else {
    obj = $expression2
}
```

This isn't ideal! Since in Kotlin *if-else* is an expression and the variable doesn't need to be nullable (or mutable).
You should immediately fix this up to:

```kotlin
val obj: Any = if(...) {
    $expression1
} else {
    $expression2
}
```

Of course not only is *if-else* an expression but so are *when* and *try-catch*. Since *try-catch* can look a bit different,
even though it is essentially the same, let me give an example more:

```java
public Object someFunc(...) {
    Object result;
    try {
        result = $expression;
    } catch (... e) {
        log(e)
        throw e
    }
    return result;
}
```

should become

```kotlin
someFunc(...): Any = try { // non-nullable Any
    $expression
} catch (e: ...) {
        log(e)
        throw e
}
```

In this case we didn't only make the variable non-nullable but also superfluous. But note that the non-nullability of the
result of the *try-catch* expression is still present in the non-nullability of the function's return type.

### avoid initialising properties with null

This is so straight forward that it's mainly for completeness here. Use *lateinit* (or [Delegates.notNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/not-null.html))
to replace nullable properties with non-nullable ones.

Instead of autogenerated Kotlin code like this

```kotlin
class Service {
    @Autowired
    var otherService: OtherService? = null
    ...
}
```

use 


```kotlin
class Service {
    @Autowired
    lateinit var otherService: OtherService
    ...
}
```

or even better in this example use constructor injection!

```kotlin
class Service(
    var otherService: OtherService, 
) {
}
```
