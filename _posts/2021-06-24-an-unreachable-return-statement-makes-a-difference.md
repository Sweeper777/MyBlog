---
tags: c#
op_link: https://stackoverflow.com/q/68109376/5133585
op_profile_link: https://stackoverflow.com/users/4430204/taquion
op_name: "taquion"
---

### Premise

OP declared an implicit conversion like this:

{% highlight c# %}
struct SomeWrapper
{
    public Guid guid;

    public static implicit operator SomeWrapper(Guid guid) => new SomeWrapper {guid = guid};
}
{% endhighlight %}

They thought that this would compile, but does not:

{% highlight c# %}
static Task<SomeWrapper> PleaseDoNotCompile() => Task.Run(() => Guid.NewGuid());
{% endhighlight %}

Then, they rewrote the lambda with a block body, and added a rather meaningless return statement, and this time it compiles successfully!

{% highlight c# %}
static Task<SomeWrapper> WhyDoYouCompile() => Task.Run(() =>
{
    return Guid.NewGuid();

    return new SomeWrapper();
});
{% endhighlight %}

Naturally, they ask why this is the case.

### An Unreachable Statement Actually Matters

At first, I was very surprised that this extra unreachable return statement _actually_ matters. I was quite skeptical, and OP wasn't quite clear about whether the unreachable return was required or not for the code to compile, so I copy and pasted it to my IDE, and deleted the line `return new SomeWrapper();` to check for myself. Amazingly, it didn't compile!

Alright, it's gotta be in the spec somewhere，I thought. First place I thought of was the [type inference](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#type-inference) section, but after checking there, it seems to be more about how the compiler infers the type parameters for methods, rather than the return type of a lambda expression. 

### Where to Look In the Spec

In the context of this question, I already know how the compiler infers the generic parameter for `Task.Run`. In the case of `PleaseDoNotCompile`, it doesn't take into account the fact that OP is assigning to `Task<SomeWrapper>` and merely 

1. looks at the lambda, 
2. sees that it has an expression of type `Guid`
3. infers the type parameter to be `Guid`. 

In the case of `WhyDoYouCompile`, the compiler 

1. looks at the lambda 
2. sees that it has the return type `SomeWrapper` *somehow*
3. infers the type parameter of `Task.Run` to be `SomeWrapper`. 

I was not interested in step 3, which is what the type inference section of the spec seems to be about. I wanted to know how the compiler does step 2, because that's what is affected by the extra return statement.

The next place I looked was [anonymous function expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#anonymous-function-expressions). I was hoping to find some info on how the return type of a lambda is determined here. However, I wasn't able to find anything. That's kind of expected, now that I think about it as I write this. Lambda expressions don't have a type - they can be converted to either expression trees or delegate types, so the section talking about them wouldn't be talking about _types_ as much.


Out of ideas, I just did a full text search for "return type" on the page, and went through the results. A particular result caught my eye - a link to a section called [inferred return type](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#inferred-return-type). The section begins by saying:

> The inferred return type of an anonymous function `F` is used during type inference and overload resolution...

That sounds like exactly what I need!

### Found It!

As I read through the section, it indeed explains why the extra return matters:

> If the body of `F` is a block and the set of expressions in the block's return statements has a best common type `T`, then the inferred result type of `F` is `T`.

Not "the block's _reachable_ return statements", just "the block's return statements". Ha! Another weird C# quirk.