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

# Safe Null-Safety or !! considered unsafe

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

## Parse, don't validate

Let's look at it in the context of loading a json file. A json file can contain
arbitrary content, but our app certainly expects a certain structure. This is very much related to the quote:
*"Even a NoSQL database has a schema, albeit an implicit one"*. So let's call this expected structure
the (implicit) schema from now on, even though no explict schema definition file exists. 
So when this json file comes in we first have to ascertain that it conforms to the schema the rest of the application is build around.

Let's look at this with a practical example: The schema we expect is that of a non-empty list of Persons.
In this simplified example a Person always comes with a first name, a last name 
(**falsehoods programmers believe about names**), and also a list of Addresses that always contains one or two Address.
With an Address containing a street name with house number, a postal code and the name of a city.

Given that any json file can always be transformed into/represented by a `Map<String, Any?>` where the values are either "primitive types"
(a `String`, a `Boolean`, an `Int`, an `Float` or `null`) or an `Array<Any?>` or `Map<String, Any?>` containing these types,
let's see how *validating* and *parsing* are different from each other. 

### Pure Validating

In its purest form validation doesn't transform the data structure it validates. It simply returns a Boolean telling you
whether the given data conforms to the required schema:
```kotlin
fun main() {
    val json = readJson("someFile.json")
    if(!json.validatePersons()) error("json didn't meet the required schema")
    // from now on you can access the json data structure freely (within the limits of the schema)
    // knowing that your casts will never fail and accessing the guaranteed content will never return null
    // (the second address isn't guaranteed after all)
}


fun readJson(fileName: String): Any? = TODO() // returns Map, List, String, Int, Float or null

fun Any?.validatePersons(): Boolean = (this as? List)?.let { it.isNotEmpty() && it.all { ::validatePerson } } ?: false

private fun validatePerson(person: Any?): Boolean {
    if(person !is Map) return false
    return persong.getOrNull("firstName").isNonNullAndNonBlankString() &&
            person.getOrNull("lastName").isNonNullAndNonBlankString() &&
            person.getOrNull("addresses").let {
                if(it !is List || it.size !in 1..2) {
                    false
                } else {
                    validateAddress(it[0]) && it[1].let { it == null || validateAddress(it) }
                }
            }
}

private fun validateAddress(address: Any?): Boolean = address is Map &&
    this["streetNameAndNr"].isNonNullAndNonBlankString() &&
    this["postalCode"].isNonNullAndNonBlankString() &&
    this["city"].isNonNullAndNonBlankString()

private fun Any?.isNonNullAndNonBlankString() = (this as? String)?.isNotBlank() ?: false
private fun Any?.isPositiveInt() = (this as? Int)?.let { it > 0 } ?: false
```
In a typed language like Kotlin operating on `Any?` is a major red flag, but in untyped languages you might not even
blink an eye when reviewing code like this.

While accessing the data like this is safe in principle, the compiler won't be able to flag if you later violate the schema
you validated against. While
```kotlin
val persons = json as List
if(persons.isNotEmpty()) {
    val person = persons[0] as Map<Any?, Any?>
    val firstAddress = person["addresses"] as Map<Any?, Any?>
    val streetNameAndNr = firstAddress["streetNameAndNr"] as String
}
```
is technically "safe" after the `validate()`-step, no code review would ever let it pass in a typed language.

### maximum parsing

Parsing is different from validating in that it not only tells you, if the data is valid, but also puts it in a structure
that will explicitly retain the guarantees that the validation only momentarily and locally verifies.

```kotlin
fun main() {
    val json = readJson("someFile.json")
    val persons: List<Person> = json.parsePersons()
    // do stuff with a strongly typed data structure that the compiler can reason about
}


fun readJson(fileName: String): Any? = TODO() // returns Map, List, String, Int, Float or null

fun Any?.parsePersons(): List<Person> = (this as? List)?.mapNotNull( ::parsePerson ) ?: emptyList()

private fun parsePerson(personJson: Any?): Person? {
    if(personJson !is Map) return null
    val nonBlankNames = listOf(
            personJson.getOrNull("firstName"),
            personJson.getOrNull("lastName"),
        ).mapNotNull {
            NonBlankString.newOrNull(it)
        }
    if(nonBlankNames.size!=2) return null
    val (firstName, lastName) = nonBlankNames
    
    val addressListJson = personJson.getOrNull("addresses") as? List<Any?> ?: emptyList()
    val addresses = addressListJson.mapNotNull( ::parseAddress )
    return if(address.size in 1..2) {
        Person(firstName, lastName, addresses[0], addresses.getOrNull(1))
    } else null
}

private fun parseAddress(addressJson: Any?): Address? {
    if(addressJson !is Map) return null
    val fields = listOf(
            addressJson.getOrNull("streetNameAndNr"),
            addressJson.getOrNull("postalCode"),
            addressJson.getOrNull("city"),
    ).mapNotNull {
        NonBlankString.newOrNull(it)
    }
    if(fields.size!=3) return false
    val (streetNameAndNr, postalCode, city) = fields
    return Address(streetNameAndNr, postalCode, city)
}

data class Person(
    val firstName: NonBlankString,
    val lastName: NonBlankString,
    val firstAddress: Address,
    val secondAddress: Address?, 
)

data class Address(
    val streetNameAndNr: NonBlankString,
    val postalCode: NonBlankString,
    val city: NonBlankString,
)

@JvmInline
value class NonBlankString private constructor(val value: String) {
    companion object {
        fun newOrNull(value: String?): NonBlankString? =
            if(value.isBlankOrNull()) null else NonBlankString(value)  
    }
}
```

## some parsing patters

### Example of an DTO using Jackson

### What to do in the presence of multiple possible schemas

Notes & References:
- falsehoods programmers believe about names
- falsehoods programmers believe about addresses
- parse, don't validate