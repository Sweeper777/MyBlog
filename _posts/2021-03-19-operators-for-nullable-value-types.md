---
tags: c# language-feature generics
op_link: https://stackoverflow.com/q/66701056/5133585
op_profile_link: https://stackoverflow.com/users/9623401/amjad
op_name: "amjad"
---

### Premise

OP notices that the `Nullable<T>` type doesn't actually declare any operators, such as `==`, yet they can use those operators on nullable types. They assume that the compiler must be calling `object.Equals` on the nullables, which would involve boxing them ("cast to `object`").

They then ask why doesn't `Nullable<T>` just overload the operators like this in the first place,

{% highlight c# %}
public struct Nullable<T> where T : struct {
   // ...
   public static bool operator == (Nullable<T> a, Nullable<T> b) {
      // do some nesseary null check...

      return value.Equals(other);
   }
}
{% endhighlight %}

I was a bit surprised at the fact that despite not declaring any user-defined operators, I can still use `==` on `Nullable<T>` too. I thought this has to be some compiler magic. Just when I wanted to go check the language spec, someone posted an [answer](https://stackoverflow.com/a/66701099/5133585). 

This actually saved me a lot of time. I didn't know where to start searching for in the language spec, but the link in that answer directly pointed me to "[Lifted Operators](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-value-types#lifted-operators)". I remember coming across this term [when I was looking stuff up about how operator overload resolution works](/2021/02/11/overload-resolution-implicit-conversions.html), but I didn't pay much attention to it at that time and forgot about it.

I went to the corresponding section in the language spec (the link in the answer points to a C# reference page, not the language spec, and I prefer the latter), and found that the lifted operators applies the regular, non-lifted versions of the operators if both operands are not null. This means that OP's assumption that `==` on nullable types calls `object.Equals` is incorrect. Then I thought, now that I've invalidated their assumption, OP is definitely going to ask "how exactly are lifted operators implemented then?"

Usually, I don't like to look into how exactly things are implemented (because these things are very prone to change, and usually not very interesting), but if I don't put that in my answer, the OP's probably not going to be satisfied, and my answer will probably be the same as the existing one, with the only difference being that I quoted from the language spec. So I opened up sharplab.io...

Apparently for two `int?`s `a` and `b`, `a == b` implemented as

{% highlight c# %}
(a.GetValueOrDefault() == b.GetValueOrDefault()) & (a.HasValue == b.HasValue);
{% endhighlight %}

Just as I expected, not very interesting.

At that point, I kind of accepted the fact that `Nullable<T>` doesn't declare any operators as one of those design decisions made by the language design team that we will never know the reason why. I thought I would show OP how to write a user-defined operator that does follow the spec, just to show that it can be done, but the language team didn't choose to do it this way.

That is when I realised, that I _can't_ write this as a user-defined operator. The spec implies that a lifted operator cannot exist without the non-lifted operator, so if there isn't a `==` operator on two `T` operands for some type `T` _to begin with_, there should be no _lifted_ `==` for type `T` either. However, we cannot declare a user-defined operator only for those `Nullable<T>` where there _is_ a `==` operator for `T`. We can't constrain the generic parameter of a user-defined operator, let alone constraining it to _that_.

So this is why they used compiler magic...