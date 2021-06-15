---
tags: java
op_link: https://stackoverflow.com/q/67945409/5133585
op_profile_link: https://stackoverflow.com/users/11786885/eve-neumann
op_name: "Eve Neumann"
---

### Premise

OP was writing some code using `EnumSet`s:

{% highlight java %}
> enum X{GREEN,RED,YELLOW}
> X.values()
{ GREEN, RED, YELLOW }
> EnumSet.allOf(X.class);
{% endhighlight %}

and as soon as that last line was executed, an exception was thrown.

```
java.lang.ClassCastException: class X not an enum
   at java.base/java.util.EnumSet.noneOf(EnumSet.java:113)
   at java.base/java.util.EnumSet.allOf(EnumSet.java:132)
   at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
   at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:78)
   at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) 
```

But `X` is clearly an enum!

### A REPL?

The first thing I found weird about OP's code was that they were using a REPL-like environment, as indicated by those `>`. However, it doesn't seem to be JShell. Just to be sure, I checked on JShell:

```
jshell> enum X{GREEN,RED,YELLOW}
|  created enum X

jshell> X.values()
$2 ==> X[3] { GREEN, RED, YELLOW }

jshell> EnumSet.allOf(X.class);
$3 ==> [GREEN, RED, YELLOW]
```

The output seems to be in a totally different from what the OP had, and I cannot reproduce the problem at all.

This is when I started to suspect that it's a bug with whatever IDE the OP is using. I thought about asking the OP what IDE and Java version they are using, but then saw that they already mentioned it in the question! 

### What Buggy IDE Are They Using?

They are apparently using something called "DrJava". A quick Google search gave me [drjava.org](http://www.drjava.org), which describes DrJava as:

> DrJava is a lightweight development environment for writing Java programs. It is designed primarily for students, providing an intuitive interface and the ability to interactively evaluate Java code. [...] it is under active development by the JavaPLT group at Rice University.

The "for students" bit and the "developed by a university" bit worries me a little, because in my experience, these sort of things are never well-maintained. There is a link to the [bug tracker](https://sourceforge.net/p/drjava/bugs/), where I simply searched for "enumset". There was only a single result, and that result was the one I needed - [enums can't appear in EnumSets](https://sourceforge.net/p/drjava/bugs/744/).

Worded this way, this bug is so silly that it's ridiculous that it still hasn't been fixed (at least the bug report is still open), since ***12*** years ago! I guess "students" don't need `EnumSet`s, huh?

### Afterword

I downloaded DrJava to check whether the bug still exists in the newest version, but I couldn't run it. There's apparently some `com.apple.eawt.ApplicationListener` class that it couldn't find. 

This is also when I noticed that the OP's stack trace has the text `java.base`. I'm pretty sure that's the "module name" thing introduced in Java 9. I went to some Java 9 documentation, and compared that with the corresponding Java 8 documentation page to confirm. OP claimed that they are using Java 8 though...

Anyway, I found the bug report, and that's enough for a Stack Overflow answer.