---
tags: java overload-resolution
op_link: https://stackoverflow.com/q/65560114/5133585
op_profile_link: https://stackoverflow.com/users/7083574/tongmutou
op_name: "tongmutou"
---

### Premise

In this code:

{% highlight java %}
public class Test {

    public void a() {
    }

    private static void a(String s) {
    }

    private static void b() {
    }

    private class Inner extends Test{
        void c() {
            a();
            a("");
            b();
        }
    }
}
{% endhighlight %}

OP is wondering why `a("");` produces a compiler error saying that `a` is inaccessible, but the other two calls are fine.

After some examination of the code, I saw that even though the `a` that OP wants to access is `private`, OP is still accessing it inside of `Test` (to be precise, `a` is being accessed inside of `Inner`, which is inside of `Test`), so there should be no accessibility problems. 

Then I saw `Inner extends Test`, so there is also an inherited parameterless `a` in `Inner`. I realised the first call to `a();` calls `this.a();`, rather than `Test.this.a();`. Using the same logic, I thought, I could say the same for `a("");`: `a("");` tries to call `this.a("");` rather than `Test.this.a("");`. The reason that fails then becomes simple: there is no `a(String)` on `this`, since private methods are not inherited. But then I'd have to explain why `b();` succeeds (Why doesn't `b();` call `this.b();`, but `Test.this.b()`?).

Before I went any further, I thought I would play around a bit, to see what are the changes I could make, to make `a("");` compile. As I found previously, I could make the call `Test.this.a("");`. I also found that removing `extends Test` will also make `a("");` compile, so it seems like the inherited parameterless `a` does play a part in this. I also tried renaming the `a(String)` overload to `f(String)`, and suddenly `f("");` compiles. I guessed that this has something to do with _names_, and this might also explain why `b();` refers to `Test.this.b();`.

Without hesitation, I looked up the "names" section of the JLS. The first place I looked at was "[Simple Method Names](https://docs.oracle.com/javase/specs/jls/se14/html/jls-6.html#jls-6.5.7.1)", where it says:

> The rules of method invocation require that the Identifier either denotes a method that is in scope at the point of the method invocation [...]

Okay, so `a` and `b` denote methods that are in scope... Fair enough. What does "in scope" mean? I navigated to "[Scope of a Declaration](https://docs.oracle.com/javase/specs/jls/se14/html/jls-6.html#jls-6.3)", where it says:

> The scope of a declaration of a member m declared in or inherited by a class type C (§8.1.6) is the entire body of C, including any nested type declarations.

That seems obvious enough... So the non-inherited parameterless `a` has the scope of the whole `Test` class, whereas the inherited parameterless `a` has the scope of the `Inner` class. They are both in scope in `c`, huh? Which one would `a` refer to then?

From my assumption that `a();` calls `this.a();` rather than `Test.this.a();`, I was sure that there must be some kind of "hiding" going on - `Test.this.a();` is somehow hidden. I scrolled down to the next section, and lucky me, it says "[Shadowing and Obscuring](https://docs.oracle.com/javase/specs/jls/se14/html/jls-6.html#jls-6.4)". That seems promising. I looked at what shadowing does:

> Some declarations may be shadowed in part of their scope by another declaration of the same name, in which case a simple name cannot be used to refer to the declared entity.

and what obscuring does:

> A simple name may occur in contexts where it may potentially be interpreted as the name of a variable, a type, or a package. In these situations, the rules of §6.5.2 specify that a variable will be chosen in preference to a type, and that a type will be chosen in preference to a package. Thus, it is may sometimes be impossible to refer to a type or package via its simple name, even though its declaration is in scope and not shadowed. We say that such a declaration is obscured.

Seems like shadowing is what I need. I then looked at when exactly is a method shadowed:

> A declaration d of a method named n shadows the declarations of any other methods named n that are in an enclosing scope at the point where d occurs throughout the scope of d.

Aha! This must be it! The inherited parameterless `a` shadows the two overloads of `a`s declared in `Test`, so only the inherited `a` is accessible.

When I started writing the answer, I realised that this quote:

> The rules of method invocation require that the Identifier either denotes a method that is in scope at the point of the method invocation [...]

is not as strong a statement as I would like it to be. Even if the inherited `a` shadows the `a`s in `Test`, the name `a` still "denotes a method that is in scope at the point of the method invocation". The reason why `a("")` doesn't work is not because `a` does not "denotes a method that is in scope at the point of the method invocation", but because the argument `""` cannot be applied to the parameterless `a`. To argue that, I'd have to quote the overload resolution section.

While I was reading the [overload resolution section](https://docs.oracle.com/javase/specs/jls/se14/html/jls-15.html#jls-15.12.1), I found a short quote that I can use to answer the question perfectly, without needing to explain names, scopes and shadowing:

> For the class or interface to search, there are six cases to consider, depending on the form that precedes the left parenthesis of the MethodInvocation:
>
> - If the form is MethodName, that is, just an Identifier, then:
>
>     - If the Identifier appears in the scope of a method declaration with that name (§6.3, §6.4.1), then:
> 
>         - If there is an enclosing type declaration of which that method is a member, let T be the innermost such type declaration. The class or interface to search is T.

I should have just gone to this section at the very beginning!

