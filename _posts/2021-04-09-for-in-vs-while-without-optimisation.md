---
tags: swift language-feature
op_link: https://stackoverflow.com/q/67021110/5133585
op_profile_link: https://stackoverflow.com/users/11191573/hyunsu
op_name: "HyunSu"
title: "For Loop vs While Loop Without Optimisation"
---

### Premise

OP finds that in debug mode, a for loop like this:

{% highlight swift %}
var sum = 0
for i in 0..<10000000 {
    sum += i
}
print(sum)
{% endhighlight %}

is slower than a while loop that does the same thing:

{% highlight swift %}
var sum = 0
var i = 0
while i<10000000 {
    sum += i
    i += 1
}
print(sum)
{% endhighlight %}

and asks why this is the case.

I knew right of the bat why this is the case, but just to confirm it, I went to [godbolt.org](https://godbolt.org) and compared the assembly code when compiling without any options versus with `-O`, versus the while loop. Most of it I couldn't understand, but I could see that the unoptimised version has a bunch of `IndexingIterator`s lying around, so my suspicion was probably right - the for loop is turned into something like this:

{% highlight swift %}
let range = 0..<10000000
var iterator = range.makeIterator()
while let next = iterator.next() {
    ...
}
{% endhighlight %}

and `iterator.next` is probably the expensive operation here. 

I felt like I have to explain _why_ `next` is expensive, so I tried to look for its source code. I first looked for `Range.swift` since its `Range`'s iterator. It'll probably be in the same file, I thought. I did a text search for "next" on that page, and found [line 693](https://github.com/apple/swift/blob/main/stdlib/public/core/Range.swift#L693):

{% highlight swift %}
public mutating func next() -> Bound? {
    defer { _current = _current.advanced(by: 1) }
    return _current
}
{% endhighlight %}

Without thinking too much, I added that to my answer. I didn't check whether it's actually doing a lot of work as I wanted to be the first to answer.

I also noted that `next` is a protocol method, and from an [article](https://medium.com/@venki0119/method-dispatch-in-swift-effects-of-it-on-performance-b5f120e497d3) that I read a while back, I learned that protocol methods use table dispatch, which is slower than static dispatch.

After posting the answer, I read through it again and realised that `next` isn't actually doing much. It's just calling `advanced(by:)`! Sure, `advanced(by:)` is also table dispatched, but I would have expected `next` to do a lot more than that, e.g. checking the bounds. Then I realised that this isn't even the iterator for `Range`. This is the iterator for `PartialRangeFrom`!

I quickly searched for the extension that conformed `Range` to `Sequence` and found [line 214](https://github.com/apple/swift/blob/main/stdlib/public/core/Range.swift#L214) that said:

{% highlight swift %}
public typealias Iterator = IndexingIterator<Range<Bound>>
{% endhighlight %}

Oh of course! I should have known from the assembly code... I looked for `IndexingIterator` in `Range.swift`, but couldn't find its declaration. That means it's in another file somewhere, so I had to do a search on the whole repo for `struct IndexingIterator`. Eventually, I found it in `Collection.swift`. Fair enough, I guess.

The [correct `next` method](https://github.com/apple/swift/blob/main/stdlib/public/core/Collection.swift#L125) actually does what I expect (i.e. quite a lot of work):

{% highlight swift %}
public mutating func next() -> Elements.Element? {
    if _position == _elements.endIndex { return nil }
    let element = _elements[_position]
    _elements.formIndex(after: &_position)
    return element
}
{% endhighlight %}

Just to be sure, I thought, I'll do a time profile of this, and experimentally show that `next` _is_ actually doing a lot of work. This time profile ended up causing OP to ask [another quesiton](https://stackoverflow.com/questions/67032783/why-is-indexingiterator-next-using-dynamic-dispatch), which made me realise my second mistake - `next` is statically dispatched!

Fortunately for me, I'm not completely wrong, `Collection.formIndex`, and `Collection.subscript`, which `next` calls are still table dispatched :)