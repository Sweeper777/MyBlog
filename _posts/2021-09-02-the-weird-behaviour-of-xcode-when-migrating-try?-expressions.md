---
tags: swift language-feature xcode
op_link: https://stackoverflow.com/q/69030052/5133585
op_profile_link: https://stackoverflow.com/users/1816142/user1816142
op_name: "user1816142"
---

### Premise

OP noticed that when migrating this code...

{% highlight swift %}
if let coordinatesArray = try? NSKeyedUnarchiver.unarchivedArrayOfObjects(ofClass: CLLocation.self, from: coordinatesArrayData) {
  return coordinatesArray
}
{% endhighlight %}

...to Swift 5, Xcode adds a cast to the `try?` expression:

{% highlight swift %}
if let coordinatesArray = ((try? NSKeyedUnarchiver.unarchivedArrayOfObjects(ofClass: CLLocation.self, from: coordinatesArrayData)) as [CLLocation]??) {
  return coordinatesArray
}
{% endhighlight %}

As a result, they want an explanation of why Xcode does this.

### The Existing Answer

When I saw this question, there was a already another answer by Sh_Khan (now deleted) posted. The answer explained that the double optional is due to `unarchivedArrayOfObjects` returning a `[CLLocation]?`, and the `try?` operator wrapping another layer of `Optional` around that. I was initially convinced by this, but after a while, realised that the answer did not actually explain why Xcode inserts the cast when converting to Swift 5. If the type of the entire `try?` expression was already `[CLLocation]??`, the cast would be unncessary!

I tried OP's code out, and tried to see exactly what type is the `try?` expression:

{% highlight swift %}
let coordinatesArrayData = Data([])
let coordinatesArray = try? NSKeyedUnarchiver.unarchivedArrayOfObjects(ofClass: CLLocation.self, from: coordinatesArrayData)
{% endhighlight %}

Putting the cursor on `coordinatesArray`, the "Quick Help" panel shows that it is of type `[CLLocation]?`. There is no double optionals, and Sh_Khan seems to be incorrect, at least for the version of Swift that I'm using - 5.4.2.

### My Hypothesis

I hypothesised that the cast to `[CLLocation]??` that Xcode adds could be because in older versions of Swift, `try?` used to produce a double optional in such cases. Xcode added the cast to match the old behaviour, I guessed. To see if this is true, I thought I should find an old Swift compiler. I looked for "Swift 4 compiler online" and found [TutorialsPoint](https://www.tutorialspoint.com/compile_swift_online.php)'s Swift 4 compiler. I tried running this code:

{% highlight swift %}
func f() throws -> Int? {
    return 1
}

print(type(of: try? f()))
{% endhighlight %}

and it printed `Optional<Int>`. Hmm, it seems like it still had the same behaviour in Swift 4. Next, I tried to find a Swift 3 compiler online. I found [Ideone](https://ideone.com/), which didn't say which version of Swift it was using when I was editing the code. I thought I should try using it anyway, so I did. Surprisingly, Ideone printed `Optional<Optional<Int>>` for the same code, and it now says that it is using Swift 4.2.2. So was this behaviour in Swift 4.2, but not in Swift 4.0 and not in Swift 5? That would be so weird!

In any case, if I can find the documentation for this change, that could make a very useful answer.

I decided to first go on https://bugs.swift.org/ to see if anyone has reported the double optionals as a bug, because I'd imagine people would find it quite annoying to deal with. I searched for "try optional", and added a filter for the "resolved" status, since I know this behaviour is gone in the newest Swift version. The first result, [SR-9934](https://bugs.swift.org/browse/SR-9934) is exactly about this. In the comments, Jordan Rose mentioned [SE-0230](https://github.com/apple/swift-evolution/blob/master/proposals/0230-flatten-optional-try.md), which is the Swift Evolution proposal that suggested this behaviour for `try?`. Great! This is exactly what I needed. In the bug report,

### The Migrated Code Has Different Behaviour!

As I was writing the answer, I noticed that even after Xcode's change, the code does not have the same behaviour as in Swift 4. Here's what the `try?` expression produces each Swift version and in each situation.

| Version | Case 1: `unarchivedArrayOfObjects` returns nil | Case 2: `unarchivedArrayOfObjects` throws | Case 3: everything works |
|-|------------------------------------------------|-------------------------------------------|--------------------------|
| Swift 4 | `.some(.none)` | `.none` | `.some(.some(...))` |
| Swift 5 | `.none` | `.none` | `.some(.some(...))` |
| Swift 5 (Xcode conversion) | `.some(.none)` | `.some(.none)` | `.some(.some(...))` |

The converted code doesn't differentiate between Case 1 and 2, just like the new behaviour of `try?`. On the other hand, the old code does differentiate between Case 1 and 2 - if it is case 1, the execution would go into the `if let` and hit the return statement, otherwise it would skip the `if let`. I do think that it is quite rare to see code that does this though.

After some conversations in the comments, apparently OP didn't care about the difference between Case 1 and Case 2, and it turned out they instead wanted the new `try?` behaviour instead. Oh well...