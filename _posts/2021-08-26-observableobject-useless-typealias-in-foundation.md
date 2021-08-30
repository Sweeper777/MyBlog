---
tags: swift combine
op_link: https://stackoverflow.com/q/68933894/5133585
op_profile_link: https://stackoverflow.com/users/14460650/iceland
op_name: "Iceland"
title: "A Seemingly Useless Typealias for ObservableObject"
---

### Premise

OP finds this seemingly pointless type alias declaration in Xcode:

![public typealias ObservableObject = ObservableObject](/assets/2021-08-26/1.png)

And asks what its purpose is.

I looked at the screenshot and thought, that looks like the "headers" view that Xcode sometimes displays. Although the other declarations all have documentation comments associated with them, and the type alias is not documented at all, I thought I should still go to the actual documentation site to check.

### The Documentation

So in Xcode, I went "Window" -> "Developer Documentation". I decided to use the search function in this instead of Google because I thought if I used Google I would just get a bunch of results for the `ObservableObject` protocol, rather than the type alias. I looked for "observableobject" in the search bar,

![Searching for "observable object"](/assets/2021-08-26/2.png)

The second result is the type alias that I'm looking for! I clicked on it, and the documentation says:

> A type alias for the Combine framework’s type for an object with a publisher that emits before the object has changed.

"...for the Combine framework's type..." Wait, is this not in the Combine framework? I looked to the right, and apparently the type alias is in `Foundation`!

### My Guess

Huh, does that mean I can do...

{% highlight swift %}
import Foundation
// without import Combine

class Foo: ObservableObject {

}
{% endhighlight %}

Apparently I can! As a result, I guessed that the purpose of the type alias is to allow people to use write `ObservableObject`s without having to import `Combine`. Then I realised, that if I am writing an `ObservableObject`, I am very likely to also be using `@Published`. If there _is_ also a type alias for `Published`, then that is very convincing evidence that supports my guess about the purpose of the type aliases. If there isn't a type alias for `Published`, then my guess would be wrong.

So I searched for "published", and viola, there it is!

![There is also a type alias for Published](/assets/2021-08-26/3.png)