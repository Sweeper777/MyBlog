---
tags: kotlin language-feature
op_link: https://stackoverflow.com/q/68690146/5133585
op_profile_link: https://stackoverflow.com/users/9671423/dmitrii-rashchenko
op_name: "Dmitrii Rashchenko"
title: "Why Type Inference for Anonymous Objects Doesn't Work at Top Level"
---

### Premise

OP noticed that this code produces an error:

{% highlight kotlin %}
val data = object {
    val field = 5
}

fun main(){
    println(data.field)  // Unresolved reference: field
}
{% endhighlight %}

If `data` was declared in `main` however, the code compiles:

{% highlight kotlin %}
fun main(){
    val data = object {
        val field = 5
    }
    println(data.field)
}
{% endhighlight %}

This is unintuitive, as normally, the _place_ something is declared shouldn't affect its accessible members. OP asks for an explanation.

### Does `object { ... }` Mean Something Else at Top Level?

I pasted this into IntelliJ IDEA, and reproduced the behaviour. My first guess was that the top level `data` has the inferred type of `Any`, which is why the OP was not able to access `field`. I hovered over the declaration of the top level `data`, and IntelliJ showed that its declared type is indeed `Any`. I also compared this with the `data` declared in `main`. IntelliJ shows that _its_ type is `<anonymous object : Any>`, so my guess was correct.

Just to make sure, I checked that the classes of `field` in both cases are both an anonymous type by printing `data.javaClass`. The `main` function `data` gave:

    class MainKt$main$data$1

and the top level `data` gave:

    class MainKt$data$1

Good, so it's not like an anonymous object expression at the top level somehow produces an `Any` object, instead of an anonymous class.

### Checking the Spec

Next, I tried going to the language spec to see if there is any special rules about type inference related to anonymous object expressions. I didn't go straight to the [type inference](https://kotlinlang.org/spec/type-inference.html#type-inference) section, because I know it's a very mathematical section, and I probably wouldn't understand anything even if I went there. I instead went to the [object literals](https://kotlinlang.org/spec/expressions.html#object-literals) section. Apparently that's what those `object { ... }` things are called.

Immediately, I found these very relevant sentences:

> The type of an anonymous object is a special kind of type which is usable (and visible) only in the scope where it is declared.
>  
> ...
>
> When a value of an anonymous object type escapes current scope:
>
> - If the type has only one declared supertype, it is implicitly downcasted to this declared supertype;
> - ...

That (`data` escaping the scope) could be why its type is `Any` when it's declared at the top level, and an anonymous type when it's declared locally! Now I just need to show that:

- the top level `data` "escapes current scope", and;
- the local `data` does not

### What Does "Escaping the Current Scope" Mean?

I looked for a section about scopes in the language spec, and found "[scopes and identifiers](https://kotlinlang.org/spec/scopes-and-identifiers.html#scopes-and-identifiers)". I thought there would be some info about what it means for a value to "escape current scope". However, after a text search, I found no mentions of "escape" in that section. The section only talks what the scopes _are_, gives a list of them, defines what it means for two scopes to be _linked_, and describes how the different scopes are linked.

One idea I had was that "escaping the current scope" meant "using the declaration in another scope", and that nested scopes count as "another scope" (perhaps?). To try to verify this, I tried declaring another property at the top level using `data.field`:

{% highlight kotlin %}
val data = object {
    val field = 5
}
val foo = data.field
{% endhighlight %}

It didn't compile, with the same error. 

Running out of ideas, I went back to the object literals section to see if I missed something. I saw that there is some example code there to illustrate how the type of an anonymous object is visible only in the declaring scope. I don't usually read examples in specs (for some reason that I myself don't even know), but this time I decided to read the example. 

An interesting thing I found was how declaring an instance property _public_ seem to count as "escaping the current scope", as the comments in this code seem to imply:

{% highlight kotlin %}
fun baz(): Base = object : Base(), I {}
// OK, as an anonymous type is implicitly
//   cast to Base

private fun qux() = object : Base(), I {}
// OK, as an anonymous type does not escape
//   via private functions
{% endhighlight %}

I thought, maybe this applies to the top level too? So I tried making `data` private, and viola, `field` suddenly becomes accessible! So I guess "escaping the current scope" actually means "making it accessible to another scope", and this also explains why the local `data.field` is accessible.

### I'm Blind

While I was writing the answer, I went back to the spec to quote from it, and that was when I saw:

> Note: in this context “escaping current scope” is performed immediately if the corresponding value is declared as a non-private global- or classifier-scope property, as those are parts of an externally accessible interface.

It's written right there all along! I'm so blind!
