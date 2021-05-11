---
tags: xcode swift
op_link: https://stackoverflow.com/q/67430453/5133585
op_profile_link: https://stackoverflow.com/users/14252600/user14252600
op_name: "user14252600"
---

### Premise

OP wants to use this Objective-C enum in Swift:

{% highlight objc %}
typedef NS_ENUM (NSInteger, BJVPlayerStatus) {
    BJVPlayerStatus_playing,
    // other cases...
    
    BJVPlayerStatus_Playing DEPRECATED_MSG_ATTRIBUTE("use `BJVPlayerStatus_playing`") =
        BJVPlayerStatus_playing
    // other deprecated cases...
};
{% endhighlight %}

But in Swift, when OP tries to do:

{% highlight swift %}
let playerStatus: BJYPlayerStatus = .playing
{% endhighlight %}

There is an error saying that `playing` is ambiguous. Apparently, there are two `playing`s in Swift: one's marked with "K" and one's marked with "V":

![Xcode Autocomplete showing two members named playing, one marked with K, the other marked with V](/assets/2021-05-07/1.png)

I don't *really* know how Objective-C enums work.

### Solving the Problem First

From my little bit of (almost none) understanding of Objective-C, the Objective-C enum has a `BJVPlayerStatus_playing` case, and a deprecated `BJVPlayerStatus_Playing`, which has the same value as the non-deprecated case. The difference seems to be only the capitalised `P` in the deprecated case. Both of these name, when translated to Swift, got turned into `playing` and that created the conflict.

So the simple solution is to somehow hide the deprecated case from Swift - it's deprecated after all, so you are not supposed to use it in new (Swift) code. I looked up how to "hide objective-c member from swift", and found [this](https://stackoverflow.com/questions/31771217/how-to-hide-an-objective-c-declaration-from-swift). 

The answer is apparently to use separate headers. I *think* what the answer meant is that there would be two header files - one contains both the deprecated and non-deprecated cases, and the other one only the non-deprecated cases (I don't know the syntax for this at all, but I think you can do this). In the Objective-C files, you would import the former header file, and in the Swift bridging header, you would import the latter.

I also remembered that you can set the Swift name of Objective-C members using a special syntax. If OP can change the Swift name of the deprecated case to something else, that's also a solution. I don't remember what that is exactly other than it's all caps and is called something like "SWIFT NAME", so I looked up "swift name for objective-c method", and the first result was [Apple's documentation](https://developer.apple.com/documentation/swift/objective-c_and_c_code_customization/renaming_objective-c_apis_for_swift). And I recognised what I was looking for immediately, the `NS_SWIFT_NAME` macro. Huh, apparently these things are what macros are.

### What Do K and V Represent?

Now that I've found the solutions, I moved on to figue out the difference between "K" and "V". One of them must represent "an enum case". So I declared an enum like this:

{% highlight swift %}
enum Foo {
    case foo
}
{% endhighlight %}

and typed in `Foo.foo` and see what autocomplete marks `Foo.foo` as. I saw that `Foo.foo` is marked as "K". It seems like "K" represents enum cases then. Now what could "V" be? The first thing that came to my mind is "V" for "Variable", so I tried declaring a `var`:

{% highlight swift %}
var a = 10
{% endhighlight %}

and typed in `a` and saw that autocomplete marks it as "V". I also tried `let` constants and they get marked as "V" too, but enums can't have `let` constants so we don't need to consider them. 

### Trial And Error

Great! Now we know that one of the Objective-C enum constants has been translated to a Swift enum constant, but the other has beem translated into a `var`. I tried changing the Objective-C enum to see how the translation changes. First I tried changing the name to something else so that we know whether this behavior is due to the Swift names being the same.

{% highlight objc %}
typedef NS_ENUM (NSInteger, BJVPlayerStatus) {
    BJVPlayerStatus_playing,
    
    BJVPlayerStatus_playing2 =
        BJVPlayerStatus_playing
};
{% endhighlight %}

Now it can be clearly seen that `playing` is marked as "K" and `playing2` is marked as "V", so obviously the `= BJVPlayerStatus_playing` part is causing `BJVPlayerStatus_playing2` to translate to a `var`. Naturally, the implementation of the `var` is:

{% highlight swift %}
var playing2: BJVPlayerStatus {
    playing1
}
{% endhighlight %}

I tried changing the Objective-C enum to:

{% highlight objc %}
typedef NS_ENUM (NSInteger, BJVPlayerStatus) {
    BJVPlayerStatus_playing,
    
    BJVPlayerStatus_playing2 = 0
};
{% endhighlight %}

This would technically have the same effect as `BJVPlayerStatus_playing2 = BJVPlayerStatus_playing`, as I _think_ the first enum constant has the value 0, but interestingly, `BJVPlayerStatus_playing2` gets translated to another enum constant in Swift this time.

But anyway, I've already got what I need for an answer.