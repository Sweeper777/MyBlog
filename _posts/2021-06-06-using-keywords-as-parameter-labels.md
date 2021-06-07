---
tags: swift language-feature
op_link: https://stackoverflow.com/q/67858629/5133585
op_profile_link: https://stackoverflow.com/users/2713740/luca
op_name: "Luca"
---

### Premise

OP saw a function declared like this:

{% highlight swift %}
func setAge(for person: String, to value: Int) {
    print("\(person) is now \(value)")
}
{% endhighlight %}

They are surprised that `for` is used as a argument label. Since `for` is also a Swift keyword, they wonder if using `for` has some sort of special meaning, or if it is just another normal argument label, like `foo` and `bar`.

### Keywords *Can* Be Used As Argument Labels, But...

I know for a fact that keywords can be used as argument labels even without backticks, but I was not sure whether it's _all_ the keywords, or just _some_ keywords. I first went to the [declarations](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html#ID471) section of the Swift reference to check:

It really doesn't say much, unfortunately:

> A parameter has a name, which is used within the function body, as well as an argument label, which is used when calling the function or method. By default, parameter names are also used as argument labels.

> A name before the parameter name gives the parameter an explicit argument label, which can be different from the parameter name. The corresponding argument must use the given argument label in function or method calls.

### How About Swift Evolution?

Then I thought maybe this is documented in one of the Swift Evolution proposals, since this isn't something you'd think of when you first design a language (at least _I_ wouldn't think of allowing keywords as argument labels). So I searched for "swift keywords as argument labels", and the second result was the _very first_ Swift Evolution proposal - [SE-0001 Allow (most) keywords as argument labels](https://github.com/apple/swift-evolution/blob/main/proposals/0001-keywords-as-argument-labels.md). Great! That looks like just the thing I'm looking for.

Since the title for the proposal says "(most) keywords", I naturally looked for which keywords are not allowed as argument labels. According to the document, `inout`, `var` and `let` are not allowed. Ah that makes sense, I thought, because back then when this proposal was published, there were still `var` and `let` parameters in Swift! If I remember correctly, you could use `var` parameter to make a parameter mutable, but unlike `inout`, the changes won't reflect on the caller's side. You used to also be able to explicitly mark a parameter as constant using `let`, I think. `inout` used to go in front of the parameter's _name_, rather than its type, which is why `inout` is not allowed as an argument label either.

### Oopsie!

So I wrote my answer, stating which keywords are not allowed, but then I thought, SE-0001 was from 6 years ago. Things probably have changed since then... Then I saw an upvote, and immediately after that the upvote was undone. That seemed to have confirmed my suspicion.

The obvious thing to do is of course try all 3 out in a playground:

{% highlight swift %}
func f(inout x: Int) {}
func g(let x: Int) {}
func h(var x: Int) {}
{% endhighlight %}

The first one didn't compile. The second and third ones gave warnings that say "`let`/`var` in this position is interpreted as an argument label" instead. Huh, interesting, because that's not consistent at all. Nowadays, there are no more `let` or `var` parameters, and `inout` is placed before the type rather than the parameter name, so in my opinion, these should either all compile (with or without warnings), or all fail to compile for backwards compatibility.

But anyway, now I know my answer was wrong.
