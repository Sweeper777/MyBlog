---
tags: java
op_link: https://stackoverflow.com/q/65768501/5133585
op_profile_link: https://stackoverflow.com/users/38663/jbu
op_name: "jbu"
title: "Creating New URL From Context"
---

### Premise

The code

{% highlight java %}
new URL(new URL("http://asdf.com/z"), "a/b/c");
{% endhighlight %}

produces the URL

    http://asdf.com/a/b/c

OP wonders why the `z` path component is being removed, and what the first parameter of the `URL(URL, String)` constructor means exactly.

### Playing Around with the Code

First, I reproduced the behaviour (of course!), then I tried some other variations of the code. For example, I tried printing the path of `new URL("http://asdf.com/z")`, to see whether it has parsed the `/z` part as its path. I tried adding a `/` at the start of `a/b/c`. I tried replacing `z` with some other path...

{% highlight java %}
System.out.println(new URL("http://asdf.com/z").getPath()); // /z
System.out.println(new URL(new URL("http://asdf.com/z"), "/a/b/c")); // http://asdf.com/a/b/c
System.out.println(new URL(new URL("http://asdf.com/hello"), "a/b/c")); // http://asdf.com/a/b/c
System.out.println(new URL(new URL("http://asdf.com/"), "a/b/c")); // http://asdf.com/a/b/c
System.out.println(new URL(new URL("http://asdf.com"), "a/b/c")); // http://asdf.com/a/b/c
{% endhighlight %}

The results basically shows what the OP has found out already - the last path component is being removed. From these results, I'm pretty confident that the first parameter (`context`) is basically the "base URL" of the second parameter `spec`. Now I just need to show that the `z` being removed is expected behaviour, or a bug.

### Searching the Docs

To show that either is the case, I need some documentation. OP has included the link to the [JavaDoc](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/net/URL.html#%3Cinit%3E(java.net.URL,java.lang.String)) in their question, which was super helpful. Going through it, I reached the point where it said:

> Otherwise, the path is treated as a relative path and is appended to the context path, as described in RFC2396.

Hmm... It doesn't actually describe how it does the "appending". Guess I'll check out RFC2396 then.

RFC2396 is apparently the RFC that specifies what URIs are. To find the part of it about appending relative paths to base paths, I simply searched for "append". I found lots of references to appendices, but after going through those, I got to the sentence:

> The reference's path component is appended to the buffer string.

That smells like something I'm looking for. Scrolling up a bit, this is a sub-step of a step that starts with "If this step is reached, then we are resolving a relative-path reference." in a section called [Resolving Relative References to Absolute Form](https://tools.ietf.org/html/rfc2396#section-5.2). That sounds like exactly what we are doing here!

Following the steps in that section, I got to (emphasis mine):

> (a) All but the last segment of the base URI's path component is
> copied to the buffer.  **In other words, any characters after the
>last (right-most) slash character, if any, are excluded.**
>
> (b) The reference's path component is appended to the buffer
string.

So getting rid of the `z` is actually expected behaviour! By reading that, I also thought of a way to keep the `z` - simply add another `/` at the end, so that the last path segment is `""`, so that nothing is ignored.

### My Paranoia

Just to be extra safe, I tested a similar code snippet in Swift:

{% highlight swift %}
// http://asdf.com/a/b/c
print(URL(string: "a/b/c", relativeTo: URL(string: "http://asdf.com/z"))!.absoluteURL)
// http://asdf.com/z/a/b/c
print(URL(string: "a/b/c", relativeTo: URL(string: "http://asdf.com/z/"))!.absoluteURL)
{% endhighlight %}

Fortunately, it produced expected results :)