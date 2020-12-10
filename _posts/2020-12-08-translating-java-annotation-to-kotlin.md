---
tag: java
op_link: https://stackoverflow.com/q/65191684/5133585
op_profile_link: https://stackoverflow.com/users/10027592/starriet
op_name: "starriet"
---

### Premise

OP saw that the Java code:

{% highlight java %}
@Component(modules = NetworkModule.class)
public interface ApplicationComponent {
    ...
}
{% endhighlight %}

is translated into the Kotlin code:

{% highlight kotlin %}
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
    ...
}
{% endhighlight %}

OP is confused because according to their knowledge, `NetworkModule.class` in Java should be translated as `NetworkModule::class.java` in Kotlin, not `[NetworkModule::class]`.

I didn't know how Kotlin annotations work either, but I do know, at a theoretical level, how Java annotations work, as I have read [the section of JLS](https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.6) about annotations while answering another question. I didn't know what Dagger is either, but I don't think that matters too much.

### Initial Guess

Since I didn't know what the `[]` in `[NetworkModule::class]` indicate either, I made a guess that that is how you are supposed to pass classes to annotations in Kotlin, so I did a little experiment to verify my claim. I first had to look up how to write a Java annotation that accepts parameters, and wrote a simple annotation that accepts a `Class<?>`:

{% highlight java %}
public @interface Component {
    Class<?> foo();
}
{% endhighlight %}

Then in Kotlin, I tried to annotate a class with it, without using `[]`:

{% highlight kotlin %}
@Component(foo = Foo::class)
class Foo
{% endhighlight %}

It worked, so seems like I was wrong.

### Google-fu

I thought maybe a better idea is to just Google it, so I googled `kotlin class in square brackets` and found [this](https://stackoverflow.com/questions/56802421/square-brackets-after-function-call), which is about using square brackets to call the `get` operator, and also [this](https://discuss.kotlinlang.org/t/make-square-brackets-mandatory-for-annotations/446/50), which is about surrounding the annotation with `[]`, rather than prefixing with `@`. Neither of these is what I'm looking for.

At this point, an idea popped into my head - maybe `[]` means "array"! I mean, `[]` means array in Swift, so this is not too far of a stretch, even though I've only created arrays in Kotlin using `arrayOf`. Therefore, I changed my search term to `kotlin array in annotation`, and that is when I found [this](https://stackoverflow.com/questions/44069045/kotlin-how-to-pass-array-to-java-annotation). It shows exactly the syntax is question, and just as I imagined, the `[]` is used to pass an array to annotations. Following the link in [yole](https://stackoverflow.com/users/147024/yole)'s answer, I quickly read through the [documentation](http://kotlinlang.org/docs/reference/annotations.html#java-annotations) about this syntax.

Just to make absolutely sure that you can't pass the array without the `[]`, I changed my annotation to:

{% highlight java %}
public @interface Component {
    Class<?>[] foo();
}
{% endhighlight %}

and as expected, the use site emits an error:

{% highlight kotlin %}
@Component(foo = Foo::class) // error
class Foo
{% endhighlight %}

### Did I Make a Mistake?

In my answer, I mentioned that the Java equivalent of the code should say

{% highlight java %}
@Component(modules = { NetworkModule.class })
{% endhighlight %}

I overlooked the fact that OP actually mentioned the original Java code, the code from which the Kotlin code is translated. That Java code does not have the `{}`! It is only after I posted my answer that I saw the original Java code. For a second there I thought the `Component` annotation accepts a single `Class`, rather than an array of them, and I made a giant mistake somewhere.

On a second thought though, I guessed that this can't be possible. The Kotlin docs clearly says that `[]` is for arrays, so this must be some *Java* language feature that I don't know. I guessed that Java probably has a feature that allows you to omit the `{}` when there is only one element, so I immediately went to the JLS to look for evidence.

Finally, I found it in [§9.7.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-9.html#jls-9.7.1), and edited the explanation into my answer.