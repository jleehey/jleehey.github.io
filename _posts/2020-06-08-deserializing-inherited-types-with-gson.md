---
layout: post
title: "Deserializing JSON:API derived types in Gson with Kotlin"
description: "Polymorphism and deserialization can be tricky. Here is a way to deserialize to the correct subclass with Gson and Kotlin in your Android application."
tags: polymorphism deserialization gson kotlin android
---

Polymorphism with deserialization can be tricky. Here is a way to deserialize to the correct subclass with Gson and Kotlin in your Android application.  The following page shows a solution for a sample API response built according to the [json:api](https://jsonapi.org/) specification.

## The problem
Say you have an abstract class, `Vehicle`, and 2 subclasses, `Car` and `Boat`:
```kotlin
abstract class Vehicle(val type: String)
class Car : Vehicle("car")
class Boat : Vehicle("boat")
```
Now say you want to deserialize the following type of object:
```kotlin
data class Property (
    val owner: String,
    val vehicle: Vehicle
)
```
With this example JSON:
```json
{
    "owner": "John",
    "vehicle": {
        "type": "boat"
    }
}
```
When you go to deserialize this, Gson will attempt to deserialize the `vehicle` field as a `Vehicle` object, instead of choosing to deserialize as a `Boat`. Gson would then throw an exception because abstract classes cannot be instantiated.

We need a way for Gson to tell which final class to deserialize to based on the object's `type` field.  For this particular example, we want the response object's `vehicle` field to be of the class `Boat`.

# JSON:API
Take a look at a sample Json API response:
```json
{
    "data": [{
        "type": "boat",
        "id": "1",
        "attributes": {
            "engineType": "Jet Propulsion"
        }
    }, {
        "type": "car",
        "id": "2",
        "attributes": {
            "bodyType": "Sedan"
        }
    }]
}
```
We see that the `data` portion of the response contains 2 different types of objects, a boat and a car. Each of those objects contain a different set of attributes.  We need a way to deserialize these attributes into their correct object types.

There are some libraries that can do this for us, such as the [jsonapi-parser](https://github.com/yigit/jsonapi-parser) published by none other than Yigit himself.  But this may not be as easy to implement if you already have a well-established API layer, or it may require more overhead on dependencies than you want.  At the very least, we can delve into how it works and shed some light on the solution (or perhaps find our own).

## Our solution: A custom Gson deserializer
To accurately deserialize to the correct object types, we need to look at the raw `JsonElement` before instantiating the class.  Gson allows us to do this by registering a custom [JsonDeserializer](https://github.com/google/gson/blob/master/gson/src/main/java/com/google/gson/JsonDeserializer.java) as a type adapter.

# Step 1: The data models we need for JSON:API
First, let's create the attributes objects for vehicles, cars and boats:
```kotlin
abstract class VehicleAttributes
class BoatAttributes(engineType:String)
class CarAttributes(bodyType:String)
```

Next, let's stub out the super class of these attribute holders, known as [Resource Objects](https://jsonapi.org/format/#document-resource-objects) (we will call them `JsonApiResource`):
```kotlin
open class JsonApiResource<T>(
        val type: String? = null,
        val id: String? = null,
        val attributes: T? = null,
)
```
Notice that this class is generically typed.  We want to be able to pass in a custom `attributes` type in order to allow us to use this class for all JsonAPI responses.

We next want to create our container response class, known as a [Document](https://jsonapi.org/format/#document-structure), which we are calling a `JsonApiDocument`:
```kotlin
open class JsonApiDocument<T>(
        val data: Array<JsonApiResource<T>?>? = null
)
```
This class is also typed to help categorize our different groups of responses.

Finally, we want to instantiate the response object and vehicle resource type classes for our vehicles:
```kotlin
class BoatResource: JsonApiResource<BoatAttributes>
class CarResource: JsonApiResource<CarAttributes>
class VehicleResponseDocument: JsonApiDocument<VehicleAttributes>
```
By explicitly declaring these types, we can pass them in directly to the [Gson.fromJson()](https://github.com/google/gson/blob/ceae88bd6667f4263bbe02e6b3710b8a683906a2/gson/src/main/java/com/google/gson/Gson.java#L816) method for deserialization.

# Step 2: The JsonDeserializer type adapter
```kotlin
class VehicleResourceDeserializer : JsonDeserializer<JsonApiResource<VehicleAttributes>> {
    override fun deserialize(json: JsonElement?, typeOfT: Type?, context: JsonDeserializationContext?): JsonApiResource<*>? {
        return when (json?.asJsonObject?.get("type")?.asString) {
            "boat" -> context?.deserialize(json, BoatResource::class.java)
            "car" -> context?.deserialize(json, CarResource::class.java)
            else -> null
        }
    }
}
```
Let's take a second to talk over what this class is doing.  When this is registered as a type adapter, and Gson sees an object with a declared type of `JsonApiResource<VehicleAttributes>`, it will defer to the `deserialize` method in this `VehicleResourceDeserializer` instead of using it's own default typing.

So, when this `deserialize` method is triggered, we first check the `type` property of the json object, then inform the deserializer which is the correct final object to deserialize to.

# Step 3: Usage
Our final step is to register and use the type adapter to deserialize our JSON API response:
```kotlin
fun parseResponse(json: String) : VehicleResponseDocument {
    val gson = GsonBuilder().apply {
        registerTypeAdapter(JsonApiResource::class.java, VehicleResourceDeserializer())
    }.build()

    return gson.fromJson(json, VehicleResponseDocument::class.java)
}
```

And voila! Our `VehicleResponseDocument` will contain resource objects of both car and boat attribute types.  We have now made our own custom deserializer to allow for different resource data attribute types within the same Json API Document.

## Some final thoughts
You may be thinking that there are ways to simplify this, and you'd be right. On the other hand, you may also be thinking that there are ways to make this work for broader applications, and you'd also be right.  For instance, did you know that the Json API spec allows the `data` field to be either an array _or_ a single object, uncontained?  This means separate class types for both versions of the response.

There are many ways to accomplish the implementation of the JsonAPI.  In many cases, this may seem tedious and would probably be easier to adopt a library that handles the bulk of this work.  

Since this article is focused on an Android implementation, I'll insert my opinion that the JSON:API specification is in general not very mobile friendly. It is much more easily used with weakly typed languages that are more common in web development, such as Javascript.  But there are benefits and drawbacks to every API implementation, so if this is the route your API team has chosen, I hope this helps clear some of the mud.