---
tags: java kotlin
op_link: https://stackoverflow.com/q/68207855/5133585
op_profile_link: https://stackoverflow.com/users/2808624/user2808624
op_name: "user2808624"
title: "ConcurrentModificationException on a Cloned List!"
---

[Just 2 months ago,](../../05/02/cme-with-sublists.html) I answered a question about CMEs, and here comes another one!

### Premise

OP has an `ArrayList<String>` that they want to `joinToString` on the main thread, but there is another thread constantly adding and removing elements to and from it. OP understands that doing this directly will cause a `ConcurrentModificationException`, so they tried to `clone` it first:

{% highlight kotlin %}
val textLines = globalTextLineArray.clone() as ArrayList<String>
myTextView.text = textLines.joinToString("\n")
{% endhighlight %}

However, CMEs still get thrown _sometimes_. They understand that they should use a thread safe collection, but wonders why cloning the array list doesn't work.

Initially, I thought `clone` calls `iterator` and iterates through the array list too, so the fact that you are "iterating and modifying at the same time" doesn't change. I looked into the source code of `clone` to confirm this, but to my surprise, it doesn't loop through the array list at all:

{% highlight java %}
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
{% endhighlight %}

Also, if what I thought were true, CMEs would happen every time, not _sometimes_, as OP said.

Okay, so it is not that. My next approach was the stack trace. The stack trace that the OP has provided mentions nothing about cloning, so I thought it was a stack trace for when the list is not cloned. I asked the OP for the stack trace for when the list _is_ cloned, but they said the posted stack trace _is_ exactly that! I became very confused.

Last time I was answering a CME question, I found this `checkForComodification` method that can potentially throw a CME. 

{% highlight java %}
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
{% endhighlight %}

I tried to think of situations where `checkForComodification` could throw a CME, but soon I realised that the problem can't be here. `modCount` of the new list is set to 0 when cloning, and after that the new list is not modified again, so it is very obvious that `modCount` will always stay expected.

I looked at the stack trace again and realised that the top was `Itr.next`, so it is impossible that `checkForComodification` threw the exception. I went to `Itr.next` and saw that there is only one line that throws a CME:

{% highlight java %}
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
{% endhighlight %}

And that is when `i >= elementData.length`. In other words, when the cursor of the iterator moved too much, past the bounds of the internal array. That would only happen if `hasNext` reports `true` when in fact there's no next element. Let's see how `hasNext` is implemented:

{% highlight java %}
public boolean hasNext() {
    return cursor != size;
}
{% endhighlight %}

Ah, so a CME would be thrown when `size > elementData.length` _for some reason_, and we iterate to the end. Now I just need to find a situation that would cause `size > elementData.length`.

I looked at what would happen if `add` and `clone` are executed at the same time, and I am the scheduler. I did this by looking at the source code of `add` very carefully, and tries to schedule the lines so that `size > elementData.length` would happen. However, on every implementation I've seen, `add` adds the element and resizes the array first, then updates the `size` field, which means that `elementData.length` is always at least `size` at any moment. There is no point in time where `clone` can step in and clone an array list with a `size > elementData.length`.

Then I suddenly remembered that the other thread also `remove`s elements from the list. I immediately knew that this is what I wanted, because it's the reverse of `add`. If `super.clone` runs before `size` is decremented, and `Arrays.copyOf(elementData, size)` runs after `size` is decremented, then we would have `size > elementData.length`!

Multi-threading without synchronisation is terrifying isn't it?