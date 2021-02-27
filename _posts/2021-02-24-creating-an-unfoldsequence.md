---
tags: swift functional-programming
op_link: https://stackoverflow.com/q/66343697/5133585
op_profile_link: https://stackoverflow.com/users/9759263/f-zer
op_name: "F. Zer"
title: "Creating an UnfoldSequence"
---

### Premise

OP is trying to implement a function called `unfoldr`. They tried:

{% highlight swift %}
func unfoldr<A, B>(_ f: @escaping (B) -> (A, B)?) -> (B) -> UnfoldFirstSequence<A> {
    return { b in sequence(
        first: b, next: { x in
                switch f(x) {
                case .some(let(a, b)):
                    return Optional(a)
                default:
                    return Optional.none
            }
            }
        )
    }
}
{% endhighlight %}

They got the compiler error:

> Cannot convert value of type 'B' to expected argument type 'A'.

After copy-pasting the code into Xcode Playgrounds, I understood exactly why the error appears. `sequence(first:next:)` accepts a `B` and a `(B) -> B?`, and returns a `UnfoldFirstSequence<B>`. However, the closure the OP passed in returns a `A?`. 

The `b` in the value binding pattern `let (a, b)` seems to be discarded.

Clearly the OP has misunderstood _something_. "What on earth is this person trying to do?" I thought.

I had a closer look at the parameter types of `unfoldr`, and noticed that it seems to be _curried_. It takes a `(B) -> (A, B)?`, returns a `(B) -> UnfoldFirstSequence<A>`, so essentially the caller would need to pass a `(B) -> (A, B)?` _and_ a `B`, in order to get a `UnfoldFirstSequence<A>`. Then it occurred to me that `B` acts as a _state_, whereas `A` is the element type of the sequence. The function `(B) -> (A, B)?` takes in a state, produces a pair that consists of

- the next element
- the next state

Then the next state is fed back into the function, producing the element after that, and so on and so forth.

It follows that the `B` that is passed to the function returned by `unfoldr` acts as the initial state.

I have no idea how I came up with all of that, but I thought this is the only reasonable way that you can form a sequence of `A`s with a function of type `(B) -> (A, B)?`.

The next natural thing to do is of course implementing `unfoldr` myself. I dropped the idea of using `sequence(first:next:)` immediately, because it has no state that I can control. All I get to control is the next element, given the previous element.

I typed in `sequence` in Xcode, and immediatelt, I saw the overload I need

![Xcode autocomplete showing sequence(state:next:)](/assets/2021-02-24/1.png)

Great! I just need to adapt the `(B) -> (A, B)?` function to a function that modifies the `inout` state parameter instead. I also realised that this returns an `UnfoldSequence`, rather than an `UnfoldFirstSequence`. The former has an extra generic parameter that indicates the type of the state, it seems.

{% highlight swift %}
func unfoldr<A, B>(_ f: @escaping (B) -> (A, B)?) -> (B) -> UnfoldSequence<A, B> {
    return {
        b in
        sequence(state: b) { x in
            switch f(x) {
                case .some(let(a, b)):
                    x = b
                    return Optional(a)
                default:
                    return Optional.none
            }
        }
    }
}
{% endhighlight %}

I went into the documentation to see exactly how `UnfoldSequence` and `UnfoldFirstSequence` are different. Apparently, `UnfoldFirstSequence` is declared as:

{% highlight swift %}
typealias UnfoldFirstSequence<T> = UnfoldSequence<T, (T?, Bool)>
{% endhighlight %}

That's a funny state type to have, I thought. I have a feeling that `sequence(first:next:)` is implemented using `sequence(state:next:)`, so I checked the [source code](https://github.com/apple/swift/blob/a73a8087968f9111149073107c5242d83635107a/stdlib/public/core/UnfoldSequence.swift#L43), and indeed it is! I have no idea how though.

Anyway, I posted the answer, and [Leo Dabus](https://stackoverflow.com/users/2303865/leo-dabus) pointed out that my code can be written in a significantly more idiomatic way.

I also noticed that `unfoldr` sounds like something that came out of Haskell, so I checked on Hoogle, and indeed it is! It does exactly what I assumed it does. Now I have an argument for when people accuse me of assuming OP's intentions. :)