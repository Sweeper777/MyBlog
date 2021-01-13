---
tags: java
op_link: https://stackoverflow.com/q/65641230/5133585
op_profile_link: https://stackoverflow.com/users/3360497/rahman
op_name: "Rahman"
title: "Mocking a Method to Return an Optional<T>"
---

### Premise

OP has this test code:

{% highlight java %}
@Test
void findByIdTest(){
    Address myAddress = new Address();
    when(addressRepository.findById(12)).thenReturn(Optional.of(myAddress));

    Address theAddress = addressServiceImpl.findById(12);
    assertNotNull(theAddress);
}
{% endhighlight %}

and wonders why including the `Optional.of` call causes a compiler error:

> no suitable method found for `thenReturn(java.util.Optional<com.luv2code.springsecurity.demo.entity.Address>`

I don't know anything about Mockito.

### My Plan

My first thought was that the error occurs because `thenReturns` expects an `Address`, not an `Optional<Address>`. To confirm this, I simplify looked up the [JavaDoc](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html) of the library that the OP is using - Mockito. My idea was to first find out what `when` returns, and in that type, find the `thenReturns` method, and see what it accepts as a parameter. My gut feeling told me that since `when` seems to be statically imported in OP's code, it's probably in this `Mockito` class, that has the same name as the library.

### Searching for the docs

I scrolled down the page, and saw example after example of how to use Mockito, which wasn't what I needed. I was looking for the "Method Summary" section of the docs, in order to find the docs for `when`. Thinking retrospectively, I could have just Cmd+F'ed for "method summary", but I didn't. Anyway, there were so many examples that I started to doubt whether this is actually a JavaDocs page, rather than a pure "Getting Started" page. I scrolled back up, and saw the classic JavaDocs page header, with all the navigation links that I'm familiar with, I am convinced that the JavaDocs for `when` is definitely on this page.

I tried Cmd+F'ing for "when", but that just finds all the `when` calls in the examples. Running out of ideas, I chose the dumb option and just tried tediously scrolling through the examples, finally finding the method summary section, and the [`when`](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html#when-T-) method.

It appears to take any `T`, and returns an [`OngoingStubbing<T>`](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/stubbing/OngoingStubbing.html):

{% highlight java %}
public static <T> OngoingStubbing<T> when(T methodCall)
{% endhighlight %}

Naturally, I clicked on the link in `OngoingStubbing`. Right at the top of that page, there is an explanation that says:

> Simply put: "**When** the x method is called **then** return y". E.g:
>
> `when(mock.someMethod()).thenReturn(10);`

### Aha!

That is when it all clicked for me. After seeing that, I recalled that Mockito is a mocking framework, and this `when(...).thenReturn(...)` is mocking a method to return a specified value. Judging from the later lines of OP's code, `addressServiceImpl.findById` returns an `Address`, not an `Optional<Address>`, so, at a semantic level, it can't be made to return `Optional.of(myAddress)`.

I also found the docs for [`thenReturn`](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/stubbing/OngoingStubbing.html):

{% highlight java %}
OngoingStubbing<T> thenReturn(T value)
{% endhighlight %}

So it takes a `T`, which is the same `T` that `when` accepts. This is exactly what I expected. Now I have another explanation for why this doesn't work, at the type-system level. 

### Another Answer?

An hour after my answer was posted, [Luis Costa](https://stackoverflow.com/users/4147392/luis-costa) posted another [answer](https://stackoverflow.com/a/65641971/5133585) saying that the problem is in the configuration. I have no idea what that means, but if that answer were correct, that meant there was some quirk in Mockito that allows you to pass `Optional<T>` to `thenReturn`, and you just need to "configure" it correctly. 

I was actually quite worried that my answer was wrong, but then OP disagreed with the other answerer and accepted my answer in the end. OP didn't think it was a configuration issue because `findById(any())` worked with `Optional.of`, which seems weird to me at first. It might be possible that `findById` has two overloads - one that takes an `int`, which returns an `Address`, and one that takes some reference type (since [`any`](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/ArgumentMatchers.html#any--) returns a `T`) and returns an `Optional<Address>`?