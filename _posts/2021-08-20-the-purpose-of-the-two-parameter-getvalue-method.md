---
tags: kotlin language-feature
op_link: https://stackoverflow.com/q/68860335/5133585
op_profile_link: https://stackoverflow.com/users/16467709/ntos
op_name: "ntos"
title: "The Purpose of the 2-parameter Map.getValue Method"
---

### Premise

OP finds [this overload of `getValue` on kotlinlang.org](https://kotlinlang.org/api>>/latest/jvm/stdlib/kotlin.collections/get-value.html).

{% highlight kotlin %}
operator fun <V, V1 : V> Map<in String, V>.getValue(
    thisRef: Any?,
    property: KProperty<*>
): V1
{% endhighlight %}

and wonders what its purpose is.

### That Looks Really Weird!

This is the first time that I have seen this `Map` method, and I was quite confused as to what it does too. Its two parameters don't make too much sense in the context of a `Map`, and somehow this only applies to `Map`s that has _strings_ as keys. The generics is also a bit weird, it can return any subtype of the map's value type!

I first read the description in the documentation:

> Returns the value of the property for the given object from this read-only map.

Okay, so I guess "given object" refers to the `thisRef` parameter? My guess didn't last very long. The "Parameters" section says:

> Parameters
> 
> `thisRef` - the object for which the value is requested (not used).

"Not used"? Then how is this method going to "return the value of the property for the given object" if it doesn't even know what "the given object" is?

### The Source Code

Fortunately, Kotlin's standard library is open source, so I clicked on the source link in the documentation to see exactly whst `getValue` does:

{% highlight kotlin %}
public inline operator fun <V, V1 : V> Map<in String, @Exact V>.getValue(thisRef: Any?, property: KProperty<*>): V1 =
    @Suppress("UNCHECKED_CAST") (getOrImplicitDefault(property.name) as V1)
{% endhighlight %}

From the implementation, I could see that it is basically doing the same thing as `map[property.name]`, except it uses the implicit default and also casts to the type argument `V1`. This explains why `thisRef` is ignored, but the purpose of this function is still a mystery. Why would someone pass in a `KProperty` to this function, just to access the value associated with that property's name? If they wanted to do that, they could have just did `map[property.name]` rather than `map.getValue(null, property)`, which is a lot longer. The ignored `thisRef` could also be removed if the sole purpose of `getValue` was to access maps with a `KProperty`.

### This Is An Operator

Then I noticed that this is an _operator_ function, which means that there exists a way to call it without using its name, like how the syntax `foo[x]` just translates to a call to the `get` operator: `foo.get(x)`. Similarly, `a + b` translates to a call to the `plus` operator: `a.plus(b)`. I just need to find out what syntax translates to `someMap.getValue(x, y)`. That syntax would probably be the "idiomatic" way to use `getValue`, just like how `foo[x]` is more idiomatic than `foo.get(x)`.

So I looked up "kotlin overloadable operators" on Google and found [this page](https://kotlinlang.org/docs/operator-overloading.html). I did a text search for `getValue` and found:

> **Property delegation operators**
>
> `provideDelegate`, `getValue` and `setValue` operator functions are described in [Delegated properties](https://kotlinlang.org/docs/delegated-properties.html).

Oh! Property delegates! That was when everything made sense to me. `getValue` is one of those methods that you have to write, when you are writing your own property delegate. So the purpose of this `getValue` function is "to allow `Map` to be used as a property delegate". This explains why `thisRef` is unused - `thisRef` is only there because the `getValue` operator required it.

### Why Use Maps As Property Delegates?

I still have one more thing that I'm unsure about though. Why would you ever want to use a map as a property delegate? I tried coming up with a usage example of using a map as a property delegate:

{% highlight kotlin %}
val someMap = mapOf(
    "one" to 1,
    "two" to 2,
    "three" to 3
)

class Foo {
    val one by someMap
}

fun main() {
    val foo = Foo()
    println(foo.one)
}
{% endhighlight %}

But that just looks kind of silly, and contrived. Not being satisfied, I googled "kotlin map as property delegate", and found [this SO post](https://stackoverflow.com/questions/41629688/kotlin-when-to-delegate-by-map), where there is a much better example:

{% highlight kotlin %}
// the map could come from e.g. JSON
class MutableUser(val map: MutableMap<String, Any?>) {
    var name: String by map
    var age: Int     by map
}
{% endhighlight %}

The map has value type `Any?`, but the properties that delegate to it has types `String` and `Int`. It took me a while to realise why this works at all - it's due to the weird generic type parameters that `getValue` has. It declares two generic type parameters `V` and `V1`, and `V1` is a subtype of `V`. The map's value type is `V`, but the return type is `V1`, which makes it possible for `getValue` to return anything that is a subtype of the map's value type. At first I thought this was unsafe, but now looking at how this is used, this makes a lot of sense.