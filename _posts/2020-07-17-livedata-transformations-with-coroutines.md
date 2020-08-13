---
layout: post
title: "Transforming LiveData with asynchronous dependencies"
description: "What to do when our ViewModel LiveData relies on an asynchronous method's value."
tags: android architecture components kotlin viewmodel
---

I recently ran into a problem with mapping one LiveData object to another with an asynchronous dependency.  I wanted to translate a LiveData `Order` API response into a view state String.  Normally, this is easily done with something like this:

```kotlin
data class OrderResponse(val total: Double)

val orderViewState = Transformations.map(Repo.orderResponseLiveData) {
    val currency = NumberFormat.getCurrencyInstance(Locale.US)
    currency.format(it.total)
}
```

The issue comes when we want to use a Locale that we fetch from a location server-side.  Let's simplify the problem as the following:
```kotlin
suspend fun getLocale()

val orderViewState = Transformations.map(orderResponseLiveData) {
    // Compilation error, we must call getLocale() from a coroutine context
    val currency = NumberFormat.getCurrencyInstance(getLocale())
    OrderViewState(currency.format(it.total))
}
```

We need the currency instance for use in the `Transformations.map()`, but we can't call any asynchronous coroutines from a synchronous context.  How do we resolve this?

### MediatorLiveData usage

Let's look at the source of `Transformations.map`:
```java
public static <X, Y> LiveData<Y> map(
        @NonNull LiveData<X> source,
        @NonNull final Function<X, Y> mapFunction) {
    final MediatorLiveData<Y> result = new MediatorLiveData<>();
    result.addSource(source, new Observer<X>() {
        @Override
        public void onChanged(@Nullable X x) {
            result.setValue(mapFunction.apply(x));
        }
    });
    return result;
}
```
We see that it is essentially a wrapper for `MediatorLiveData` that adds a single new source.  So let's try postponing the source addition until we have all of the objects we need. 

This is an example solution within the ViewModel `init`:

```kotlin
class OrderViewModel : ViewModel() {

    lateinit var currencyFormat
    val orderViewState = MediatorLiveData<OrderViewState>()

    init {
        viewModelScope.launch {
            // Succeeds because we are calling getLocale() from a coroutine context
            currencyFormat = NumberFormat.getCurrencyInstance(getLocale())
            orderViewState.addSource(Repo.orderResponseLiveData) { order ->
                orderViewState.postValue(currencyFormat.format(order.total))
            }
        }
    }
}
```

By delaying the addition of the MediatorLiveData source until the assignment of the `currencyFormat` is complete, we can guarantee that all necessary objects are available from the target context.  By initializing the LiveData with an empty `MediatorLiveData`, we can guarantee that the observer won't throw an exception and is free to call `viewModel.orderViewState.observe()` at any time.