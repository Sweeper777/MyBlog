---
tags: c# overload-resolution
op_link: https://stackoverflow.com/q/66150292/5133585
op_profile_link: https://stackoverflow.com/users/4033656/marcel-grüger
op_name: "Marcel Grüger"
title: "Operator Overload Resolution + Implicit Conversions"
---

### Premise

OP has made a `PreciseInteger` that looks something like this:

{% highlight c# %}
public class PreciseInteger
{
    public double PreciseValue {get; private set;}
    public int RoundedValue {get; private set;}

    public PreciseInteger(double value)
    {
        PreciseValue = value;
        RoundedValue = (int)Math.Round(value, 0, MidpointRounding.AwayFromZero);
    }

    public static implicit operator PreciseInteger(int number) { ... }
    public static implicit operator PreciseInteger(double number) { ... }
    public static implicit operator int(PreciseInteger number) { ... }
    public static implicit operator double(PreciseInteger number) { ... }
    public override string ToString() { ... }
}
{% endhighlight %}

When they tried to do:

{% highlight c# %}
double myValue = myClass.StoredValue1 / myDivider;
{% endhighlight %}

They found that it uses integer division, and asked how to make it use `double` division instead.

### Does This Actually Compile!?

Nowhere in the question does it mention the types of `myClass.StoredValue1` and `myDivider`, but from context they seem to both be instances of `PreciseInteger`.

At first I didn't believe OP's claim that the `/` operator resolved to the `(int, int)` overload. I didn't even expect `myClass.StoredValue1 / myDivider` to compile. There is no `/` operator defined for `PreciseInteger` after all. Even if the compiler is smart enough to figure out that `PreciseInteger` can be converted to `int`/`double`, I thought it would be ambiguous since `PreciseInteger` can be implicitly converted to both `double` and `int`. I thought the `/(int, int)` overload and the `/(double, double)` overload would be equally applicable. But apparently, as my experiment on dotnetfiddle showed, OP's claims are actually true!

I simply copied the `PreciseInteger` class, and wrote this in `Main`:

{% highlight c# %}
var a = new PreciseInteger(1);
var b = new PreciseInteger(2);
double c = a / b;
Console.WriteLine(c); // prints 0
{% endhighlight %}

Huh, that must mean that `/(double, double)` is a _better function member_. I can easily find the docs that explain what that is, I thought, so I moved on to figuring out how to make it use the `double` overload.

### Forcing The Compiler's Preferences

The first thing I thought of is just casting the two operands to `double`, i.e.

{% highlight c# %}
double myValue = (double)myClass.StoredValue1 / (double)myDivider;
{% endhighlight %}

But that seemed _too_ simple... And as I thought, OP specifically said they didn't want that in the question. 

>  I don't want to use an explicit casting (like `Convert.ToDouble` or `(double)`).

Then I'm rather out of ideas. The only way to make the compiler prefer another overload is to change the parameter types, and the two trivial ways to do that are prohibited by the OP.

That is when I thought, I could just define an _even better_ function member! `/(PreciseInteger, PreciseInteger)` would be an even better function member because of the identity conversion. I immediately tried it out in dotnetfiddle. Initially I used `double` as the operator's return type, but for some reason it didn't compile and I got some very cryptic compiler errors. Thinking back, there was absolutely no reason why this shouldn't work, but I forgot what I did exactly, so I can't reproduce the problem.

Anyway, I changed the operator to return a `PreciseInteger`:

{% highlight c# %}
public static PreciseInteger operator /(PreciseInteger number1, PreciseInteger number2) {
    return number1.PreciseValue / number2.PreciseValue;
}
{% endhighlight %}

It doesn't really matter at the end of the day, as there are implicit conversions to `double`.

### Checking The Spec

After that I looked at the language spec to see exactly where it says that the `int` overload is a [better function member](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#better-function-member). For an overload to be a better function member than another overload, the conversion to that overload's parameter types must be [better](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#better-conversion-from-expression) than than the conversion to the other overload's parameter types. And a conversion is better if the target type is a [better conversion target](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#better-conversion-target). `int` can be implicitly converted to `double`, but `double` can't be implicitly converted to `int`, and that apparently makes `int` a better conversion target.

That is when I thought, this class is kind of weird. It should have value semantics, so should be a struct. That aside, why would one want something that can represent a floating point and an integer at the same time? This smells like someone not liking the way C# works and trying to make it more "&lt;insert weakly-typed language here&gt;-like".