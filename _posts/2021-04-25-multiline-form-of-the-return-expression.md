---
tags: kotlin language-feature
op_link: https://stackoverflow.com/q/67250202/5133585
op_profile_link: https://stackoverflow.com/users/8205462/alexey-nezhdanov
op_name: "Alexey Nezhdanov"
---

### Premise

OP has this code:

{% highlight kotlin %}
fun check_plain(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: return
    // do checking
}
{% endhighlight %}

Rather than returning immediately when `supplier` returns a null, they want to execute some other code (e.g. log some messages) and then return, and they still want to use the elvis `?:` operator.

{% highlight kotlin %}
fun check_plain(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: /* logger.info("...") and then return */
    // do checking
}
{% endhighlight %}

### Kotkin Has Return Expressions? Nice!

My first thought was, ah, `return` in Kotlin is an _expression_ rahter than a statement, which is why something like:

{% highlight kotlin %}
supplier.get() ?: return
{% endhighlight %}

is valid.

I looked up "kotlin 'return expression'" just to confirm this, and found [this thread](https://discuss.kotlinlang.org/t/why-return-and-throw-is-expression-and-not-statement/10891), which contained an example of using the elvis operator to illustrate the usefulness of making `return` an expression. Great. It seems like OP is using this feature correctly.

Then I thought about the _type_ of the return expression. Since it is an expression, it gotta have a type. Since I can put `return` where a `String` is expected, I thought that `return` can have any type, and its type is automatically inferred by context. This means that its type only depends on the expressions around it, but not the expression that is returned! (Later I checked the spec to see that its type is actually `Nothing`, but the conclusion that it doesn't depend on the type of expression returned still holds)

### Easy Solution

This means that I can write `return <anything>`, and `return` will still work as the right hand side of `?:`. Well, why not

{% highlight kotlin %}
fun check_plain(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: return logger.info(...)
}
{% endhighlight %}

I didn't have any logging frameworks installed, so I tested this with `print` instead, and it works!

However, I realised that this only works because the return type of `logger.info` matches the return type of `check_plain`, and there is only one statement to run. What if that wasn't the case?

### Let's Try Harder

I started experimenting. First I looked at OP's attempt:

{% highlight kotlin %}
fun check_log(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: {
        logger.info { "Nothing interesting here" }
        return@check_log
    }
    // do checking
}
{% endhighlight %}

Clearly that won't work. The lambda isn't even being invoked! The LHS of `?:` is of type `String`, and the RHS of the lambda is of a function type. Also, the `@check_log` part wasn't needed. The default behaviour of `return` is to return from the immediately enclosing _function_, i.e. `check_log`. Therefore, my first attempt was to invoke the lambda:

{% highlight kotlin %}
fun check_log(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: {
        print("hello")
        print("bye") // simulating multiple statements to run
        return
    }()
}
{% endhighlight %}

That gave an error that OP mentioned in the question:

> 'return' is not allowed here

### IntelliJ's Insightful Quick Fix

IntelliJ also says that this is a case of "Redundant lambda creation" and offers a quick fix. I tried the quick fix, and it produced:

{% highlight kotlin %}
inline fun check_log(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: run {
        print("hello")
        print("bye")
    }
}
{% endhighlight %}

There is still a type mismatch error, but it turned  multiple statements into one invocation expression invoking `run`. Well, that means I can just use the trick before and do `return run { ... }`:

{% highlight kotlin %}
fun check_plain(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: return run {
        print("hello")
        print("bye")
    }
}
{% endhighlight %}

Indeed, this worked, but I'm not satisfied. This looks like a trick more than anything. "return run" doesn't read well. What does that even mean?

### What's So Special About 'Here'?

I want to know _why_ "'return' is not allowed _here_", so I went to the [spec](https://kotlinlang.org/spec/expressions.html#return-expressions) and checked. Apparently,

> If a return expression is used in the context of a lambda literal which is not inlined in the current context and refers to any function scope declared outside this lambda literal, it is disallowed and should result in a compile-time error.

Hmm, so I need to inline the lambda? I remember reading about this "inline" feature that Kotlin has, but doesn't quite remember how to use it. I tried putting the word `inline` before the `{` of the lambda. Nope, that upsets the parser completely. How about after the `{`? Still nope. Finally, I quickly skimmed through what the [docs](https://kotlinlang.org/docs/inline-functions.html) has to say about `inline`, and apparently it is a modifier for functions. I tried modifying `check_log` with `inline`, and I got a warning 

> Inlining works best for functions with parameters of functional types

Then I finally remembered what `inline` does and what it means for a lambda to be inlined. A lambda is inlined if it is passed to an `inline` function. "Wasn't `run` an `inline` function?" I thought. If it were, then I could return inside the lambda passed to `run`... The last expression in a lambda is the return value of that lambda, and `run` returns the same value as its lambda... This could work!

{% highlight kotlin %}
fun check_plain(supplier: Supplier<String?>) {
    val arg: String = supplier.get() ?: run {
        print("hello")
        print("bye")
        return
    }
}
{% endhighlight %}

Indeed it does!