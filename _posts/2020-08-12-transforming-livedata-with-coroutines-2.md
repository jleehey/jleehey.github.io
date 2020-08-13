---
layout: post
title: "A better way of transforming LiveData with asynchronous dependencies: switchMap"
description: "What to do when our ViewModel LiveData relies on a constantly-changing asynchronous method's value."
tags: android architecture components kotlin viewmodel
---

My recent [post on LiveData asynchronous dependencies]({% post_url 2020-07-17-livedata-transformations-with-coroutines %}) explained how we can use a MediatorLiveData to delay the addition of a data source until after an asynchronous dependency has been resolved.  While this is a good strategy for a one-off dependency, what do we do when we need to make an asynchronous call after every LiveData emission?

Let's review the problem from the previous article:
```kotlin
suspend fun getLocale()

val orderViewState = Transformations.map(orderResponseLiveData) {
    // Compilation error, we must call getLocale() from a coroutine context
    val currency = NumberFormat.getCurrencyInstance(getLocale())
    OrderViewState(currency.format(it.total))
}
```
The assumption for the previous solution with `MediatorLiveData` was that the locale would stay constant throughout the ViewModel's lifespan.  With that assumption, we can assign the result of the coroutine to a local field, and use it for subsequent livedata emissions.  But what do we do when it is possible for `getLocale()`'s value to change after the `ViewModel` has been initialized?

### A better solution: [switchMap](https://developer.android.com/reference/kotlin/androidx/lifecycle/Transformations#switchMap(androidx.lifecycle.LiveData,%20androidx.arch.core.util.Function))

[switchMap](https://developer.android.com/reference/kotlin/androidx/lifecycle/Transformations#switchMap(androidx.lifecycle.LiveData,%20androidx.arch.core.util.Function)) is a LiveData transformation that we overlooked.  This method will run a function we specify on every `value` that is emitted from the `source`.  It is essentially the same as `Transformations.map` except it will swap out the backing LiveData of the transformation each time with a new delegate, allowing us to run asynchronous methods between those assignments.  In our previous example, it would be like removing the MediatorLiveData source, running `getLocale()`, and then adding a new source to the MediatorLiveData each time, all in a single handy method on initialization.

Let's move forward and complete the example.  `switchMap`'s signature is nearly identical to `Transformations.map`, except the result of the map function will be a LiveData to replace the existing source after each method call:

```kotlin
class OrderViewModel : ViewModel() {
    suspend fun getLocale()

    val orderViewState = Repo.orderResponseLiveData.switchMap { order ->
        liveData {
            // We are now in a LiveDataScope and can call coroutines
            val currencyFormat = NumberFormat.getCurrencyInstance(getLocale())
            emit(currencyFormat.format(order.total))
        }
    }

    // Note: switchMap{} extension and liveData{} functions come from
    // lifecycle-livedata-ktx dependency
}
```
You can see that this example is already much cleaner than our previous solution.  We are able to instantiate the LiveData immediately (no need to add to `init`), and it will apply the most recent result of `getLocale()` on each update.

I hope that, at the least, this helps add another entry to your playbook of lifecycle management strategies.