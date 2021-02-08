---
tags: java generics
op_link: https://stackoverflow.com/q/66037939/5133585
op_profile_link: https://stackoverflow.com/users/2032413/bcf
op_name: "bcf"
---

### Premise

OP is wondering why this code compiles...

{% highlight java %}
List<? extends List<Number>> list3 = new ArrayList<>();
List<List<Double>> list4 = (List<List<Double>>) list3;
{% endhighlight %}

...but it doesn't compile without the nested generics:

{% highlight java %}
List<Number> list1 = new ArrayList<>();
List<Double> list2 = (List<Double>) list1;
{% endhighlight %}

Clearly, in both code snippets we are trying to convert `List<Number>` to `List<Double>`, but the first code snippet is only an unchecked (but not invalid) cast.

### Huh, This Is Weird...

I was a bit surprised that you can cast from `List<? extends List<Number>>` to `List<List<Double>>`, so I tried out OP's code in IntelliJ IDEA. IntelliJ did not report any errors, so it seems like OP was right. Since I did not expect this at all, I decided to just quote from the spec to explain the behaviour (i.e. "because the spec says so").

I decided to start looking from [5.5 Casting Contexts](https://docs.oracle.com/javase/specs/jls/se14/html/jls-5.html#jls-5.5), but then realised that the "XXX Contexts" sections don't actually what specific conversions are allowed, only the _kinds_ of conversions that are allowed. 

### Allowed Narrowing Reference Conversions

I thought the conversion in question is either a [narrowing reference conversion](https://docs.oracle.com/javase/specs/jls/se14/html/jls-5.html#jls-5.1.6), or a [widening reference conversion](https://docs.oracle.com/javase/specs/jls/se14/html/jls-5.html#jls-5.1.5). I looked at both sections and realised that it can't be a widening reference conversion, as one of the necessary conditions for it is that there is subtype relation between the types being converted. (Now that I think back, I don't know why I was so sure that `List<? extends List<Number>>` is a subtype of `List<List<Double>>`, because the subtyping rules for generics are quite complicated. A better reason for why this isn't a widening reference conversion would be that if it were, it would be allowed in an assignment context)

Out of the [three necessary conditions for a widening reference conversion to be allowed](https://docs.oracle.com/javase/specs/jls/se14/html/jls-5.html#jls-5.1.6.1), "If there exists a parameterized type X that is a supertype of T (`List<List<Double>>`), and a parameterized type Y that is a supertype of S (`List<? extends List<Number>>`), such that the erasures of X and Y are the same, then X and Y are not provably distinct" is the only one that is more complicated to resolve. The other two are trivially true. 

This condition threw me off for a bit, since it's talking about _parameterised supertypes_ of S and T. I thought this doesn't cover the cases where the generic type has no generic superclass/superinterfaces, but after reading [4.10 Subtyping](https://docs.oracle.com/javase/specs/jls/se14/html/jls-4.html#jls-4.10) I realised that I haven't considered bounded wildcards as the type arguments. Even if `A<T>` has no generic superinterfaces/superclasses, things like `A<? extends T>` is still a supertype of `A<T>`.

The other thing that threw me off was the "if there exists a type such that..., then..." structure. It seems to mean "for all types, if..., then...", but just worded in a rather bad way.

### "Provably Distinct"

I thought, suppose I have a parameterised supertype `X` of `T`, and a parameterised supertype `Y` of `S`, and their erasures are the same, then I need to show that they are not provably distinct. I'll assume `X` to be `Collection<List<Double>>` and `Y` to be `Collection<? extends List<Number>>` for now.

Then I looked up the [sufficient conditions for two paramterised types to be provably distinct](https://docs.oracle.com/javase/specs/jls/se14/html/jls-4.html#jls-4.5). The first one is trivially false, so I only needed to consider the second one: "Any of their type arguments are provably distinct". Oh gosh! "Provably distinct" is an overloaded term, isn't it?

Fortunately, the sufficient conditions for _type arguments_ to be provably distinct are just in the [next section](https://docs.oracle.com/javase/specs/jls/se14/html/jls-4.html#jls-4.5.1). It's basically three cases:

- if both type arguments are not type variables or wildcards, then they must be the same type.
- if exactly one of them is a type variable or wildcard, then its upperbound `S` must not satisfy `|S| <: |T|` or `|T| <: |S|`, where `T` is the type of the other type argument.
- if both of them is a type variable or wildcard, then their upperbounds `T` and `S` must not satisfy `|S| <: |T|` or `|T| <: |S|`

For `Collection<List<Double>>` and `Collection<? extends List<Number>>`, the second case applies. `S` is `List<Number>` and `T` is `List<Double>`. Then I realised that `|S|` and `|T|` are the same type! I'm not sure whether the `<:` relation is reflexive. Intuitively, a type is not a subtype (or a supertype, for that matter) of itself, but who knows what the JLS thinks. So I went back to 4.10, and found that types _are_ subtypes (and supertypes) of themselves, but not _property subtypes_ of themselves. The supertype relation is defined as the transitive and _reflexive_ closure of the direct supertype relation, and the subtype relation is the inverse of the supertype relation.

Alright then, I thought, so `Collection<List<Double>>` and `Collection<? extends List<Number>>` are not provably distinct, and there is a cast between them, and wrote my answer.

### The Spec Was Wrong!?

After a while, I got a comment asking what `javac` version I was using. I was rather confused, but I ran the code with `javac` once more anyway. To my surprise, the cast failed! I became even more confused. _Did the OP ask this question on a false premise? Was the OP lying? How did I reach a wrong conclusion by following the spec? Why didn't IntelliJ report any errors?_

A few days later, I posted a [related question](https://stackoverflow.com/q/66075419/5133585), and no answers as of time of writing.