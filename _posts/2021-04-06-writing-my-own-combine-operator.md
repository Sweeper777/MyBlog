---
tags: swift combine
op_link: https://stackoverflow.com/q/66962267/5133585
op_profile_link: https://stackoverflow.com/users/3970488/aswath
op_name: "Aswath"
---

### Premise

OP wants to see if a publisher completes without publishing anything. If that is the case, make it publish a specfied thing. They wonder if there is a Combine operator that does something like that.

{% highlight swift %}
let p1String: [String] = []
let p1 = p1String.publisher
    .map { Int($0) }
    .ifEmpty(publish: nil) // p1 will publish nil and complete

let p2String: [String] = ["1", "2", "3"]
let p2 = p2String.publisher
    .map { Int($0) }
    .ifEmpty(publish: nil) // p2 will publish 1, 2, 3 and complete 
{% endhighlight %}

I initially was skeptical about this and thought this gotta be impossible. The only way to know whether a publisher will complete before publishing anything is to wait for it to complete. If it has not published anything, and has not completed, we don't know if it will publish something in the future. However, if it has completed, it can't publish any more elements, so the specified element can't be published. I posted a comment explaining this to the OP, but then thought about it more and realised that argument could be applied to almost every existing combine operator:

> To transform each published element to another element, that element must have been published. Since it is already published, if we also publish the transformed element, we'd be publishing both the untransformed and transformed elements! Therefore `map` is impossible.

Realising how stupid my impossible argument is, I suddenly _got_ how Combine operators work. The operators are publishers too. Each operator manages an upstream publisher, and whenever the upstream publishes something, it does some logic and publishes some stuff on its own. The consumer only sees what the last publisher and doesn't care about the upstream ones.

I started to imagine how an `ifEmpty(publish:)` operator would work: it would publish everything its upstream publishes, and keeps track of whether its upstream has published anything. It would also know when its upstream has completed. When it has, and the upstream hasn't published anything yet, it would publish the specified thing. It can do this because _it hasn't completed yet_, only the upstream has.

I thought I would look for such a thing in the Combine library first, but I somehow couldn't find anything related to "complete without publishing anything", so I decided to have a jab at it myself. I've figured out the logic already after all.

I started with:

{% highlight swift %}
struct IfEmpty<Upstream: Publisher>: Publisher {
    let upstream: Upstream
    let output: Output
    
    func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input {
        
    }
    
    typealias Output = Upstream.Output
    
    typealias Failure = Upstream.Failure
}
{% endhighlight %}

I have no idea how to implement `receive`, so I had a look at what methods are available on `subscriber`. Turns out, I can make it `receive` things. I guessed that this is how I make my custom publisher publish things.

Still, I don't know how I can make it publish everything another publisher publishes, so I googled "Combine custom operator" and went to the [first result](https://www.wwt.com/article/creating-your-own-custom-combine-operator). It was a rather long article, but I managed to find where they implemented `receive`. Apparently, you had to use `subscribe(subscriber)`.

After that, the logic was rather straightforward, and not much to talk about.

Having posted the answer however, another user reminded me that actually such an operator [_does exist_](https://developer.apple.com/documentation/combine/publishers/zip4/replaceempty(with:)), and is called `replaceEmpty(with:)`. I have no idea how I missed that in my initial search through the list. Maybe because I didn't look for "empty", but rather "complete without...".

I wanted to edit that into my answer, but alas, a third user posted a second answer already...

