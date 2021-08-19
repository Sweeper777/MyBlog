---
tags: c# formatting
op_link: https://stackoverflow.com/q/68781491/5133585
op_profile_link: https://stackoverflow.com/users/211764/rob-hoff
op_name: "Rob Hoff"
---

### Premise

OP was using the "D04" format specifier to format some numbers:

{% highlight c# %}
Console.WriteLine($"{-1:d04}"); // -0001
Console.WriteLine($"{1:d04}");  // 0001
{% endhighlight %}

The problem with this is that the numbers are not the same width. There would be one extra character when the number is negative. OP wants a format specifier that produces:

```none
-001
0001
```

### Looking for a Custom Format Specifier

At first, I thought that's quite a weird requirement, but I guess I'll give it a try. I went straight to the [custom number format specifiers](https://docs.microsoft.com/en-us/dotnet/standard/base-types/custom-numeric-format-strings) page because I doubted there is such a specialised standard format specifier for this. I wanted to look for something like a "negative sign specifier `-`", that lets me say "if the number is negative, put a negative sign there, otherwise put a 0 there". Then the format would just be

```none
-000
```

### The Semicolon

But there isn't such a thing :( But as I read through the list of available specifiers, I found an unfamiliar one - `;`. I know roughly what all the other speciiers (`0`, `#`, `.`, `,`, `%`, `‰`, `e`, `\\`) does, except `;`, and it's apparently called a "section speciier". This name has almost nothing to do with negative numbers, which almost made me ignore it.

The documentation says:

> The semicolon (`;`) is a conditional format specifier that applies different formatting to a number depending on whether its value is positive, negative, or zero. To produce this behavior, a custom format string can contain up to three sections separated by semicolons. These sections are described in the following table.

Aha! This is actually very useful for what OP is trying to do. After a bit of trial and error, I found that this works:

```none
0000;-000
```

But then I remembered that OP was using the standard format specifier `D`, but `0000;-000` isn't using that. I tried changing the format specifier to:

```none
d04;d03
```

But that isn't recognised as a valid format specifier. It seems like I can't mix standard and custom format specifiers :(

Therefore, I thought I'd include in my answer the special things that `D` does, that `0000;-000` doesn't do. After having a look at the [docs](https://docs.microsoft.com/en-us/dotnet/standard/base-types/standard-numeric-format-strings#decimal-format-specifier-d), the only special thing that `D` does is that it uses the `NegativeSign` of the current `NumberFormatInfo`. That's a super duper minor thing IMO, but I'll mention it just to be safe, I thought.

### Aw, Another XY Problem!

Then I had a look at the comments, where OP had said that the reason why they were trying to do this were because they want to sort some strings, and they want to "line things up". I thought, well, in that case, you could just use the _alignment_ specifier!

Console.WriteLine($"{1,5:d04}");  // " 0001" (note the leading space)
Console.WriteLine($"{-1,5:d04}"); // "-0001"

What an XY problem!