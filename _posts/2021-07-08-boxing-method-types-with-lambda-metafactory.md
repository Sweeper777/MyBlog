---
tags: java
op_link: https://stackoverflow.com/q/68301835/5133585
op_profile_link: https://stackoverflow.com/users/5841827/marcel
op_name: "Marcel"
---

### Premise

OP is trying to use `LambdaMetafactory` to generate lambdas that implement these functional interfaces from given getters and setters in a given class:

{% highlight java %}
@FunctionalInterface
public interface IGetter<T, R>
{
  @Nullable
  R get( T object );
}

@FunctionalInterface
public interface ISetter<T, I>
{
  void set( T object, @Nullable I value );
}
{% endhighlight %}

They have written some code. It works most of the time, except for the case where the getter/setter gets/sets a primitive field. They ask about how to "box" that to the wrapper type, so the primitive case is covered too.

Before this point, I didn't even know this API existed in the JDK.

I first tried to see for myself how the OP's methods are supposed to work, so I wrote a reference type getter,

{% highlight java %}
public class Main {
  public String getFoo() {
    return "Foo";
  }
}
{% endhighlight %}

and tried to use the `createGetterViaMethodname` method to get it, in the form of an `IGetter<Main, String>`. `createGetterViaMethodname` has the signature:

{% highlight java %}
public static <T, R> IGetter<T, R> createGetterViaMethodname( final Class<T> clazz, 
                                                              final String methodName,
                                                              final Class<R> fieldType )
{% endhighlight %}

Based on the parameter names, I guessed that I'm supposed to use it like this:

{% highlight java %}
createGetterViaMethodname(Main.class, "getFoo", String.class)
    .get(new Main()) // this should give me "Foo"
{% endhighlight %}

And indeed, it gave me `"Foo"` when I ran it. Then, I tried a primitive getter:

{% highlight java %}
public int get0() {
  return 0;
}

// ...
createGetterViaMethodname(Main.class, "get0", int.class)
    .get(new Main());
{% endhighlight %}

and I get an exception just as OP said. The exception message says:

> Receiver class `Main$$Lambda$14/0x0000000800b69c40` does not define or inherit an implementation of the resolved method `abstract java.lang.Object get(java.lang.Object)` of interface `IGetter`.

The stack trace shows that this exception gets thrown only when I call `get`. `createGetterViaMethodname` did not throw an exception. Next, I tried to use `Integer.class` instead of `int.class` as the last parameter, hoping that OP didn't try this and that this would solve it. However, there is still an exception, but interestingly, the message is different:

> no such method: `Main.get0()Integer/invokeVirtual`

and this is thrown at the `findVirtual` call. I hypothesised that `findVirtual` is the method that actually finds the getter, given the name and the return type. 

Then, I tried to figure out what the `LambdaMetafactory.metafactory` call is doing. Judging by the arguments the OP passed, it seems to be creating the interface implementation. We tell it what interface we want to implement (`MethodType.methodType( IGetter.class )`, no idea why it is wrapped in a `MethodType` though), the method to implement (`get`), and some other _types_, which I'm not sure what the significnce is. Anyway, I thought, judging from the exception message I got from passing `int.class`, it appears that the wrong interface method got implemented. Rather than implementing `Object get(Object);` (the erased version of `R get(T);`), we implemented `int get(Object);` because we used `int.class`.

I guessed that out of the 6 parameters we pass to `LambdaMetafactory.metafactory`, either the 4th or the 6th, or both, determines the type of the method the interface implementation is going to have. Either way, the method's type depends on `type`, so to solve the problem, we just need to add some code that says: "if `type` is a primitive, change it to the corresponding wrapper type". I looked in `MethodType`'s list of methods to see if there is anything that tells me if it is a primitive, and found `hasPrimitives`. That is when I realised `MethodType` doesn't represent _one_ type. It represents all the collection of types that define a method - its return type and all its parameter types. I should check the `fieldType` parameter instead. It even has the more familiar `Class` type.

Next, I need to convert a primitive `Class` to the corresponding wrapper `Class`. I know this must be hardcoded, because I've seen [this question](https://stackoverflow.com/questions/3473756/java-convert-primitive-class) before, so I just copy and pasted that solution with the map into my code. Now I need a way to change the return type of a `MethodType`. Going through the list of methods, I can see that it has `changeReturnType`, which returns a new `MethodType` object, rather than setting it. I also found `changeParameterType`, which would be useful when fixing  `createSetterViaMethodname`.

In the end, I ended up with:

{% highlight java %}
MethodType type = target.type();
if (fieldType.isPrimitive()) {
  type = type.changeReturnType(map.get(fieldType));
}
{% endhighlight %}

And it works!