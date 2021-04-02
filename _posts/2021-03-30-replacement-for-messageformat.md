---
tags: java
op_link: https://stackoverflow.com/q/66870234/5133585
op_profile_link: https://stackoverflow.com/users/1045142/simon-martinelli
op_name: "Simon Martinelli"
title: "Replacement for MessageFormat"
---

### Premise

OP is looking for a way to format dates, numbers and other strings, by replacing certain placeholders in a format string with values. A locale can also be specified to control the locale used to do all the formatting. `MessageFormat` does exactly this:

{% highlight java %}
// returns String: Hello, Number: 123, Date: 2021/04/01 15:01
MessageFormat.format("String: {0}, Number: {1}, Date: {2}", "Hello", 123, new Date());
{% endhighlight %}

However, it doesn't support `java.time` types such as `LocalDate`. It simply calls `toString` on it, and the locale applied to the `MessageFormat` has no effect on the formatting.

This is the first time I've seen the `MessageFormat` class. I didn't know the Java Standard Library had such a thing until this day. I initially thought it was part of some popular library. I looked at its [documentation](https://docs.oracle.com/javase/7/docs/api/java/text/MessageFormat.html), hoping that I can find a way to _make it_ support `LocalDate` after working out how it works.

Although it provides quite a lot of ways for customisation, like `ChoiceFormat`, different date and number styles and all that kind of cool stuff, it is stuck on using `DateFormat`/`SimpleDateFormat` to format dates, and there is no way to change that to something like `DateTimeFormatter`.

![MessageFormat uses DateFormat and SimpleDateFormat](/assets/2021-03-30/1.png)

Having seen this, I ran out of ideas. Then I thought, `java.time` has been there for a while now, people has definitely noticed that `MessageFormat` doesn't support `java.time`. OP surely isn't the first one to have this question, so I googled `messageformat java.time`, and the first result starts with:

> [There needs to be support for java.time in java.text](http://mail.openjdk.java.net/pipermail/core-libs-dev/2013-December/023665.html)...

Oho, someone is thinking not dissimilar to the OP here. I clicked on the result, and it's a mailing list. The formatting there was really hard to read with all those quotes everywhere. (Maybe it's just me, not used to reading mailing lists...) I didn't understand the flow of the conversation, but what I did understand, was that there was someone who wants something similar to what OP wants.

The mailing list mentioned two bug reports:

- [8016743](https://bugs.openjdk.java.net/browse/JDK-8016743): java.text.MessageFormat does not support java.time.* types
- 8016742: JAXB does not support java.time.* types

The first one seems more relevant, so I went to the first one, whose link can be found at the top of the page.

Then I am delighted to see that the bug is marked "resolved", and the resolution is "won't fix". Great! That means there's probably a replacement for it. I scrolled down and saw a comment explaining that:

> The `MessageFormat` is designed to work with `java.text.Format` classes, so it uses `DateFormat`/`SimpleDateFormat` to format date/time. Providing support for `java.time.format.DateTimeFormatter` to format `java.time` types (`TemporalAccessors`) may complicate the `MessageFormat` API. It is always recommended to use `java.util.Formatter` which provides support for formatting `java.time` types.

I thought I'd work out how to use `Formatter` exactly, and add an example of using it in my answer, rather than only dumping that quote and calling it a day.

After reading the documentation, I found that `Formatter` uses the C-style placeholders (`%d`, `%s`, `%x` etc), which I had always disliked. But whatever, _I'm_ not the one using it at the end of the day. `Y` is for the ISO 8601 format date according to the docs, so I tried:

{% highlight java %}
StringBuilder sb = new StringBuilder();
Formatter formatter = new Formatter(sb, Locale.US);
formatter.format("Today is %Y", LocalDate.now());
System.out.println(sb);
{% endhighlight %}

and it says that it can't recognise `Y` as a conversion specifier. I tried a few other conversions, and was able to get the conversions for strings and numbers working, just not dates. After struggling for a few dozen minutes or so, I decided to reread the documentation, and realised that all the date time conversions required a `t` prefix!

{% highlight java %}
formatter.format("Today is %tY", LocalDate.now());
{% endhighlight %}

That's just silly!
