---
tags: c# functional-programming
op_link: https://stackoverflow.com/q/65745802/5133585
op_profile_link: https://stackoverflow.com/users/9325131/topica
op_name: "topica"
title: "Haskell's Do Notation in C#!?"
---

### Premise

By defining `Select` and `SelectMany` extension methods on `Nullable<T>` appropriately,

{% highlight c# %}
public static class NullableExtensions
{
    public static U? Select<T, U>(this T? nullableValue, Func<T, U> f)
        where T : struct
        where U : struct
    {
        if (!nullableValue.HasValue) return null;
        return f(nullableValue.Value);
    }

    public static V? SelectMany<T, U, V>(this T? nullableValue, Func<T, U?> bind, Func<T, U, V> f)
        where T : struct
        where U : struct
        where V : struct
    {
        if (!nullableValue.HasValue) return null;
        T value = nullableValue.Value;
        U? bindValue = bind(value);
        if (!bindValue.HasValue) return null;
        return f(value, bindValue.Value);
    }
}
{% endhighlight %}

OP is trying to create something similar to Haskell's `do` notation in C#, using LINQ query syntax:

{% highlight c# %}
int? nv1 = 5;
int? nv2 = 3;
int? nv3 = 8;
var q = from v1 in nv1
        from v2 in nv2  // Error CS0453: anonymous type is not struct
        from v3 in nv3
        select v1 + v2 + v3;
{% endhighlight %}

Unfortunately this produces an error.

After reproducing the error at [sharplab.io](https://sharplab.io), I noticed the error message says that some anonymous type is not a value type, so it does not fulfill the constraint for the type parameter `V` of `SelectMany`, that `V` has to be a value type. There are no anonymous types used in the source code, so I guessed that the compiler must have compiled the query expression to something that uses an anonymous type.

To figure out exactly what code the compiler compiled the query expression into, I rewrote the query expression with arrays to make it actually _compile_:

{% highlight c# %}
int[] nv1 = { 5 };
int[] nv2 = { 3 };
int[] nv3 = { 8 };
var q = from v1 in nv1
        from v2 in nv2
        from v3 in nv3
        select v1 + v2 + v3;
{% endhighlight %}

The compiler generated some very unreadable names on sharplab.io:

```
<>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
int[] array = new int[1];
array[0] = 5;
int[] source = array;
int[] array2 = new int[1];
array2[0] = 3;
<>c__DisplayClass0_.nv2 = array2;
int[] array3 = new int[1];
array3[0] = 8;
<>c__DisplayClass0_.nv3 = array3;
Enumerable.SelectMany(Enumerable.SelectMany(source, new Func<int, IEnumerable<int>>(<>c__DisplayClass0_.<M>b__0), <>c.<>9__0_1 ?? (<>c.<>9__0_1 = new Func<int, int, <>f__AnonymousType0<int, int>>(<>c.<>9.<M>b__0_1))), new Func<<>f__AnonymousType0<int, int>, IEnumerable<int>>(<>c__DisplayClass0_.<M>b__2), <>c.<>9__0_3 ?? (<>c.<>9__0_3 = new Func<<>f__AnonymousType0<int, int>, int, int>(<>c.<>9.<M>b__0_3)));
```

Nevertheless, I read through the generated code, and noticed that there are indeed anonymous classes being created. I'd rather show a nicer version of the code in my answer, but I couldn't be bothered to clean up the generated code, so I decided to rewrite the query expression myself.

After much struggling, I came up with:

{% highlight c# %}
var q =
    nv1.SelectMany(x => 
       nv2.SelectMany(x => nv3, (v2, v3) => new { v2, v3 }), 
       (v1, v2v3) => v1 + v2v3.v2 + v2v3.v3);
{% endhighlight %}

This doesn't look the same as the generated code, but it doesn't matter. Here's why: while writing this, I realised that the need for an anonymous type arises because the parameter `f` in `SelectMany` only takes 2 parameters, but we are working with 3 (or potentially more!) `int?`s here. No matter what, we need a way to "clump" 2 values together to form "one" value so that it can be passed to `f`, and then decomposed again. An anonymous class seems to be what the compiler has chosen to use.

An alternative to anonymous types is `ValueTuple`s, which _are_ value types, and would have worked with `int?`s:

{% highlight c# %}
var q =
    nv1.SelectMany(x => 
       nv2.SelectMany(x => nv3, (v2, v3) => (v2, v3)), 
       (v1, v2v3) => v1 + v2v3.v2 + v2v3.v3);
{% endhighlight %}

OP seems to have thought of this too, as their workaround is:

{% highlight c# %}
var q = from v1 in nv1
        from v2 in nv2
        select (v1, v2) into temp
        from v3 in nv3
        select temp.v1 + temp.v2 + v3; 
{% endhighlight %}

This "using anonymous type to clump values" behaviour, which seemed like such an innocent little feature, has become really annoying in this situation, hasn't it? I wanted to find out whether this is specified behaviour, or whether it is compiler-dependent. After a quick read of the [Query Expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#query-expressions) section of the language spec, it's not hard to find that this behaviour is specified, and specified in the clearest way possible...

Having found the last nail in the coffin, I was ready to post the answer saying that this just can't be done, but then I thought, "what if we remove the `where V: struct` constraint?" I tried doing that, and `V?` stopped compiling. Aha, that's why the constraint is there - `Nullable<T>` requires that `T` is a value type. That means that this could work if we write our own `Nullable<T>` type _without_ requiring `T` be a value type! But then I remembered that the built in `Nullable<T>` acutally has quite a lot of compiler support regarding boxing, so creating my own `Nullable<T>` _just_ for some pretty `do` notation syntax is a bit... you know.
