---
tags: c# language-feature
op_link: https://stackoverflow.com/q/67086586/5133585
op_profile_link: https://stackoverflow.com/users/8587776/digvijaysinh-gohil
op_name: "Digvijaysinh Gohil"
---

### Premise

OP asks about the difference between these, as they both seem to subscribe to the event, and sets a method as the event handler.

{% highlight c# %}
SomeEvent += SomeMethod;
{% endhighlight %}

{% highlight c# %}
SomeEvent = SomeMethod;
{% endhighlight %}

### The Difference Is Obvious, But Let's Look Deeper

Obviously, one of them adds `SomeMethod` as a _new_ handler (as the semantics of `+` suggests), while the other one discards all the previous handlers, and sets `SomeMethod` as the sole handler. However, since OP wrote that they want to know exactly what's going on under the hood, I decided to investigate more.

I thought I would experiment on SharpLab for a bit, then I would check the spec to see whether what I see is specified behaviour or an implementation detail.

First thing I tried was:

{% highlight c# %}
public void M() {
    E = F;
}

public void F(object sender, EventArgs e) { }

event EventHandler E;
{% endhighlight %}

I see that in the decompiled C#, `E = F;` has been decompiled to:

{% highlight c# %}
this.E = new EventHandler(F);
{% endhighlight %}

Ah, right, it's creating a new instance of the delegate. Then I remembered this is actually a [method group conversion](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/conversions#method-group-conversions), so I checked that section of the spec, and indeed it is specified to create a new instance of a delegate.

### It's Just a Bunch Of Delegates, Right?

This made me assume that events are really just thread-safe wrappers around delegates, so I thought I would go look up what `+=` does on delegates next. I went into the [delegates](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/delegates) section of the spec, and searched for `+=`, and found this section:

> Delegates are combined using the binary + and += operators.

I didn't actually know that delegates can be added with `+`. I always thought you had to use `+=`!

It is at this point that I remembered, that events have `add` and `remove` accessors. It's just that I tend to forget about them because you never need to write them. When are _they _called then!? I quickly opened up the [events](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#events) section, which says that `+=` and `-=` calls the `add` and `remove` accessors, so events are not the same as delegates after all! I also found that events with has no explciit accessors (the kind that I've always been writing) are actually called "field-like events". Field-like events apparently generate a delegate-typed field.

### Apparently Not

I then tried a bunch of things on SharpLab, and found that you can't use `=` on a non-field-like event:

{% highlight c# %}
public void M() {
    E = F; // doesn't compile
}

public void F(object sender, EventArgs e) { }

event EventHandler E {
    add { }
    remove { }
}
{% endhighlight %}

That's interesting! That could suggest that `=` is actually setting the delegate backing field! 