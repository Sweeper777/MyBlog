---
tags: c# library functional-programming
op_link: https://stackoverflow.com/q/69350132/5133585
op_profile_link: https://stackoverflow.com/users/427525/brian-kessler
op_name: "Brian Kessler"
title: "Consuming TryAsync"
---

### Premise

OP has a method that returns `TryAsync`, which is from the [`LanguageExt`](https://github.com/louthy/language-ext) library.

{% highlight c# %}
private TryAsync<bool> Relay(
    MontageUploadConfig montageData,
    File sourceFile,
    CancellationToken cancellationToken
) => new(async () =>
{
    byte[] fileContent = await _httpClient.GetByteArrayAsync(sourceFile.Url, cancellationToken);
    return await _attachmentController.TryUploadAttachment(montageData.EventId, fileContent, sourceFile.Name);
});
{% endhighlight %}

They ask how to consume the `TryAsync<bool>` returned by this merthod.

I have not even heard of such a library.

### Google-fu

I simply googled "tryasync", trying to find some documentation for `TryAsync`. Instead of finding documentation, I ended up finding [a very random page](https://www.csharpcodi.com/csharp-examples/LanguageExt.Prelude.Append(TryAsync,%20TryAsync)/#google_vignette) that has the source code of one of the methods related to `TryAsync`:

{% highlight c# %}
[Pure]
public static TryAsync<A> Append<SEMI, A>(this TryAsync<A> lhs, TryAsync<A> rhs) where SEMI : struct, Semigroup<A> =>
    lhs.Append<SEMI, A>(rhs);
{% endhighlight %}

There is a link to the source file in which this is located - `TryAsync.Prelude.cs`. I clicked on the link, hoping to see other methods related to `TryAsync`, including those that can be used to "consume" it.

I was brought to a source browser thingy, with a file explorer on the left. I found the "TryAsync" folder, and saw a `TryAsync.cs`. I thought, that must be where the "core" members of `TryAsync` are declared! I opened the file, and was surprised by the fact that `TryAsync` is actually a delegate:

{% highlight c# %}
public delegate Task<Result<A>> TryAsync<a>();
{% endhighlight %}

Aha, so that's what the `Try` part means. Rather than a `Task` that can throw ezceptions, this is a `Task` that wraps the exceptions into the `Result`. Well now knowing this, consuming this is just as easy as:

{% highlight c# %}
Result<bool> result = await Relay(...)();
{% endhighlight %}

### That's It?

Is that really all there is to the question? It can't be that simple, right? I thought. Since the OP seems to be trying to write code in a really functional style, I thought maybe they want to consume it in a "functional way", or in a way that is considered idiomatic in the context of the LanguageExt library. So I looked at the other files in the "TryAsync" folder, trying to find other methods that can be used, but I could not find anything. Every method is about combining/composing two or more `TryAsync`s in some way. I stopped for a bit and asked myself rhetorically: how can you get more "functional" than `await Relay(...)()`? The _nature_ of this operation is a side effect!

Then I thought, maybe OP doesn't know how to consume the `Result`. If I talk a bit about how to consume the `Result` in my answer, it will be more complete. So I navigated to `Result/Result.cs`, and found that `Result` has `IfSucc` and `IfFail`. Well, just demonstrating those two methods should be enough.

### The Catch

Just to be safe, I went on the library's [GitHub page](https://github.com/louthy/language-ext) to learn the general concepts of the library. It apparently is trying to make C# into Haskell. In particular, this caught my eye:

> The types all have constructor functions rather than public constructors that you instantiate with `new`. They will always be `PascalCase`:

{% highlight c# %}
Option<int> x = Some(123);
Option<int> y = None;
List<int> items = List(1,2,3,4,5);
Map<int, string> dict = Map((1, "Hello"), (2, "World"));
{% endhighlight %}

So OP should have created a `TryAsync` like this:

{% highlight c# %}
TryAsync(async () =>
{
    byte[] fileContent = await _httpClient.GetByteArrayAsync(sourceFile.Url, cancellationToken);
    return await _attachmentController.TryUploadAttachment(montageData.EventId, fileContent, sourceFile.Name);
});
{% endhighlight %}

### OP's Actual Problem (possibly)

I thought this was a stylistic choice, so I didn't mention this in my answer, but as [another answer](https://stackoverflow.com/a/69358330/5133585) later pointed out, OP _definitely_ should have created `TryAsync` this way, instead of `new`. Using `new` instead of `TryAsync(...)` could also be why OP asked the question in the first place... Oh well.