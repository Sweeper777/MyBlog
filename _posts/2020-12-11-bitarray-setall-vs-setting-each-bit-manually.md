---
tags: c#
op_link: https://stackoverflow.com/q/65245591/5133585
op_profile_link: https://stackoverflow.com/users/6866309/pavan-chandaka
op_name: "Pavan Chandaka"
title: "BitArray.SetAll vs Setting Each Bit Manually"
---

### Premise

OP creates a `BitArray` of length 2, and sets both bits to 1 in two different ways:

{% highlight c# %}
bitArray.SetAll(true);
{% endhighlight %}

{% highlight c# %}
bitArray[0] = true;
bitArray[1] = true;
{% endhighlight %}

After that, they copy the `BitArray` to a `byte[]` then use `BitConverter` to convert the `byte[]` to a `long`. They find that that the two ways produce different results. Using `SetAll` produces an unexpected 255, whereas setting the bits individually produces the correct 3.

### Checking the Docs

My first guess was that OP must have missed/misunderstood something in the `BitArray` documentation, so the first thing I checked was that the [constructor](https://docs.microsoft.com/en-us/dotnet/api/system.collections.bitarray.-ctor?view=net-5.0#System_Collections_BitArray__ctor_System_Int32_)'s parameter specifies the number of *bits* in the bit array, rather than say, the number of *bytes*. Then I checked [`SetAll`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.bitarray.setall?view=net-5.0#System_Collections_BitArray_SetAll_System_Boolean_) and [`CopyTo`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.bitarray.copyto?view=net-5.0#System_Collections_BitArray_CopyTo_System_Array_System_Int32_) to see if they do anything funny. To my disappointment, it doesn't, so I have to keep looking.

### Why 255?

While I was reading the docs, I was also thinking about how this wrong 255 can be produced. The only way that can be happen is if the LSBs of the bit array are:

```
1111 1111
```

So I guessed that .NET's `BitArray` must have used an `int[]` or a `long[]` as its underlying store of bits, just like Java's `BitSet`. This means that there can be some bits which are technically stored in the `BitArray`, but are not "in" the `BitArray` as far as client code is concerned. For example, if the `BitArray` is of length 2, the underlying `int[]`/`long[]` would contain one `int`/`long` with 32/64 bits, and 30/62 of those bits are technically "stored", but not "in" the `BitArray` as far as an outside observer is concerned, as the client code can't access them.

`SetAll(true)` must have then set all those `int`s/`long`s to `-1` (all 1s in binary). I wished this were documented, which is why I checked `SetAll`'s docs, but it isn't. `CopyTo`, due to some kind of bug, does not know that those bits are not "in" the `BitArray` and copies them to the `byte[]` regardless. 

### Finding Evidence For A Bug

Now I need to find evidence for this is a bug. At this point I realised I haven't tried to reproduce this problem yet, so I went on [dotnetfiddle](https://dotnetfiddle.net/) and wrote a little snippet:

{% highlight c# %}
BitArray myBA = new BitArray(2);
myBA.SetAll(true);            
byte[] byteArray = new byte[8];
myBA.CopyTo(byteArray, 0);
Int64 finalInt = BitConverter.ToInt64(byteArray,0);
Console.WriteLine(finalInt);
{% endhighlight %}

I was able to reproduce the problem on .NET framework 4.7.2. It printed the wrong 255. I thought if I _cannot_ reproduce this on another framework, e.g. .NET Core, then it would be evidence that this is a bug. So I tried running the same code on .NET 5, and on my own machine, which has .NET Core installed. In both cases, it prints the correct 3.

Aha, I thought, this must be a bug in the .NET framework then. I also tried printing the first few bytes of the `byteArray` to confirm that the problem is in `CopyTo`, not `BitConverter`:

{% highlight c# %}
Console.WriteLine(byteArray[0]);
Console.WriteLine(byteArray[1]);
Console.WriteLine(byteArray[2]);
{% endhighlight %}

That prints 255, 0, and 0, just as I thought.

Then I wanted to confirm my theory about the behaviour of `SetAll(true)`, that it sets all the underlying bits, even if they're not "in" the `BitArray`, to `-1`. I tried to increment the length of the `BitArray`, and print the newly added bit:

{% highlight c# %}
myBA.Length++;
Console.WriteLine(myBA[2]);
{% endhighlight %}

I expected it to print true, but it printed false. I am still convinced that my theory is correct, because I can't think of any other way that it can produce 255. I concluded that setting the `Length` must have automatically cleared the new bits.

Finally, I went to the reference source browsers for [.NET framework](https://referencesource.microsoft.com/) and [.NET Core](https://source.dot.net) to compare how exactly they implemented `SetAll` and `CopyTo`. There, I found solid evidence for my theory on how `SetAll(true)` and `CopyTo` works. Most importantly, `CopyTo` in .NET Core does not copy the extra bits that are not "in" the `BitArray`.