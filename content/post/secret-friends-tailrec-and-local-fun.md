+++
author = "Stephan Schröder"
title = "Secret Friends: tailrec & local functions"
date = 2025-06-15T20:10:52+02:00
description = ""
tags = [
    "kotlin", "missed connections", "tailrec", "local function"
]
+++

The Kotlin documentation currently introduces `tailrec` only by showcasing a converging recursive function with a single parameter.
(if that has changed, it might be due to my [youtrack ticket](https://youtrack.jetbrains.com/issue/DOC-34752/common-tailrec-with-accumulator-parameter-pattern-should-be-mentioned-in-documentation "YouTrack ticket: common tailrec with accumulator parameter pattern should be mentioned in documentation"))

```kotlin
val eps = 1E-10 // "good enough", could be 10^-15

tailrec fun findFixPoint(x: Double = 1.0): Double =
    if (Math.abs(x - Math.cos(x)) < eps) x else findFixPoint(Math.cos(x))
```

While this does its job of introducing `tailrec`, the problem is that most tailrec-eligible functions seem to require an accumulation parameter.

### a what?

What do I mean by 'accumulation parameter'? Let's look at the factorial function. Let's write it in a simple recursive fashion:

```kotlin
fun factorial(n: Int): Long =
  if(n<2) 1 else n*factorial(n-1)
```

If you put a `tailrec` in front of that function definition, you'll get two warnings:
- "A function is marked as tail-recursive but no tail calls are found." pointing to `tailrec`.
- "Recursive call is not a tail call." pointing to the invocation of `factorial` in the second line.

The problem is that in `tailrec`-functions the part after the else must be only a call to the recursive function, but here it's: `n*factorial(..)`, so the problem is the `n*`.
Let's rewrite that function by making the multiplication part of the function. This second parameter is what I call an accumulation parameter:

```kotlin
tailrec fun factorial(n: Int, acc: Long = 1): Long =
  if(n<2) acc else factorial(n-1, n*acc)
```

### so what's the problem now?

Now we have a recursive function, that benefits from the `tailrec`-optimization. But the problem is that the new `acc` parameter is leaky.

While we can invoke the function correctly with a single parameter (like `factorial(4)` due to the second parameter bering optional), nothing stops us from providing a second parameter,
but only providing `1` will return the proper result!

### an unlikely language construct saves the day

So how do we hide the second parameter from users? By hiding it in a [local function](https://kotlinlang.org/docs/functions.html#local-functions)!

```kotlin
fun factorial(n: Int): Long {
  tailrec fun _factorial(n: Int, acc: Long): Long =
      if(n<2) acc else _factorial(n-1, n*acc)
    
  return _factorial(n, 1)
}
```

So now `factorial` is a wrapper function. It hides the accumulation parameter and the real tail recursive function away in its body. The leak is fixed and the optimization achieved.

And that's why `tailrec` and `local functions` are secretly best friends!