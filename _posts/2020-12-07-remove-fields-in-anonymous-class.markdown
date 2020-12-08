---
layout: so-post
author: Sweeper
categories: csharp
op_link: https://stackoverflow.com/q/65178867/5133585
op_profile_link: https://stackoverflow.com/users/12918011/kenny-smith
op_name: "Kenny Smith"
---

### Premise

OP wants to serialise a JSON object with four values. They put them in an anonymous class:

{% highlight c# %}
return JsonConvert.SerializeObject(new
{
    errorCode,
    parameter1,
    parameter2,
    context 
});
{% endhighlight %}

However, some values can be null and they don't want the null values to appear in the JSON, so they ask for a way to remove the null values from the anonymous class.

### My Initial Response

The moment I read the question, I started thinking about the `??` operator, but after a few attempts at thinking of a solution directly, realised that removing a field in anonymous class conditionally is probably not possible. I realised that "suppose you *could* do this, what'd happen if you tried to access the property that would be conditionally removed? How would the compiler know whether the access is valid?" I can easily "prove that by contradiction" that doesn't make sense so I made a mental note that I should mention that at the end of my answer, and moved on to search for workarounds.

### Workarounds

My first thought was dictionaries, putting all the values in a dictionary like this:

{% highlight c# %}
new Dictionary<string, object>
{
    { nameof(errorCode), errorCode },
    { nameof(parameter1), parameter1 },
    { nameof(parameter2), parameter2 },
    { nameof(context), context } 
}
{% endhighlight %}

and then using LINQ to remove the null KVPs. That sounds like a lot of work, as I have to repeat each parameter. LINQ also produces an `IEnumerable<KeyValuePair<string, object>>`, which I have to convert to a dictionary again. Urghh. There's gotta be a better way, I thought.

The second thought is similar to the above, but with a list of the parameters. It was a split second before I rejected this, because I immediately realised that I would lose the parameter names would be gone if I do that, and the serialised JSON would be a JSON array, rather than the JSON object that the OP wanted.

### Google-fu

It is at this moment that I realised that OP is serialising JSON using [NewtonsoftJson](https://www.newtonsoft.com). I only know the basics of how to use this library, but I know it's a powerful JSON library that's very customisable. Maybe you can tell it to not serialise the null values...

So I looked up `csharp newtonsoftjson remove null values` on Google and found this [`NullValueHandling`](https://www.newtonsoft.com/json/help/html/NullValueHandlingIgnore.htm) thing. In that page, there is literally a code example doing what the OP wants! Apparently, you just need to pass in a `JsonSerializerSettings` with `NullValueHandling` set to `Ignore`. However, that example called `SerializeObject` with 3 arguments - `object`, `Formatting`, and `JsonSerializerSettings`. I need an overload that takes only `object` and `JsonSerializerSettings`.

At this point, I could have just believed in the developers of NewtonsoftJson, and assumed that that overload exists, but I quickly looked up the [documentation page](https://www.newtonsoft.com/json/help/html/M_Newtonsoft_Json_JsonConvert_SerializeObject_5.htm) for `JsonConvert`, and double-checked, just in case.

Finally, I was ready to post the answer. All the above happened only in a matter of 5 minutes within the posting of the original post, *somehow*.
