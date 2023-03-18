+++
author = "Stephan Schröder"
title = "Safe Null Safety or !! considered unsafe"
date = 2023-03-16T21:56:19+01:00
description = "Kotlin's flavour of null safety is famously unsound, but this unsoundness can be dealt with at the edge to technologies that have no concept of nullability (e.g. Java or JSON)."
tags = [
    "kotlin",
]
draft = true
+++

# Safe Null Safety or !! considered unsafe

Learning Kotlin changed the way I prefer to program.
Since then, it's a bit tedious to work in programming languages that don't offer null safety and sum types (sealed classes) in one way or another.
This is why I tend to get puzzled when I meet other developers (or their code) who clearly didn't come to the same conclusion.
Sometimes you hear sentences like "Nullability is overrated", sometimes it's code which is littered with the not-null assertion operator `!!`,
clearly only there to shut up the compiler.

So let's see how to restrict the unsoundness of Kotlin's null safety in order to harness the full power of nullability.

## Kotlin's unsound constructs


### platform types

Next to a non-nullable type `T` and a nullable type `T?` there is also the platform type `T!`. You can assign this type yourself,
instead it is assigned by Kotlin's type inference to any instance that is returned by Java code (at least those that haven't been
annotation with [certain nullability annotations](https://www.baeldung.com/kotlin/platform-types#java-annotations-supporting-nullability-check)).

The designers of Kotlin could have decided to make any

- `lateinit`/`Delegates.notNull()`
- not-null asserting code `x!!` (or in a slightly different forms `x ?: errror("x expected to not be null")`, `requireNotNull(x)`, `checkNotNull(x)`)

### how to handle nullable values
- argument to functions expecting a nullable value
- `if`, `when`, `?:` to handle case of null
- `if`, `when`, `try catch` are expressions and can help initialise non-nullable variables
- `?:` to provide a fallback value
- `filterNotNull()` to filter a list containing `List<T?>` to a `List<T>`

**definition of unsafe**
- ok to use if you verify that it's sound in a way the compiler can't
- use sparingly/controlled, in other words not littered throughout the code base

**benefits of sound(ish) null safety**
no surprises! ->
- the null becomes 100% intentional to denote the absence of an object on a domain level
- make the null useful e.g. as mark an object for immediate removal (`mapNotNull()`)
- no null checks needed -> shorter straight forward code
- invocation chains are no longer brittle

**But what about Lombok's @Builder annotation? There seems to be no in-build builder facility in Kotlin. Do we have to implement a Builder for each class manually?**

**TLDR:** No, we don't. Use optional parameters (=parameters with default arguments) in your Kotlin class and use named parameters when invoking it.

## What is the problem Lombok @Builder is solving?

Let's assume we have a DTO containing many properties including `username` and `password`. The class has one constructor to initialize all its properties.
Invoking this constructor in Java>=10 will look something like this:

```java
var a = new A(null, null, "a_schmidt", "1234", null, null);
```

The issues here include:
- a lot of visual noise (having to write `null` for all the parameters you don't have a value for)
- are you sure that username and password are on the right position without jumping to the constructor definition?
  The compiler only guarantees that the types are matching, but how do you know that you didn't mix up the order of - in this case - username and password?