---
tags: swift locale
op_link: https://stackoverflow.com/q/65966539/5133585
op_profile_link: https://stackoverflow.com/users/9400147/crowjam
op_name: "crowjam"
---

### Premise

OP is trying to find Danish names using a regex, but they don't want to hardcode all the characters in the Danish alphabet into the regex, as they might allow more languages later.

In the code they showed, they are using `String.range(of:options:locale)` to match a string against a regex:

{% highlight swift %}
func isUserNameValid(userName: String, locale: Locale) -> Bool {
    return userName.range(
        of: "(?mi)^[a-z](?!(?:.*\\.){2})(?!(?:.* ){2})(?!.*\\.[a-z])[a-z. ]{1,}[a-z]$",
        options: .regularExpression,
        range: nil,
        locale: locale) != nil
}
{% endhighlight %}

They also ask whether `range(of:options:locale:)` take into account the `locale` parameter, and detects "special language characters" automatically.

### range(of:options:locale)

I decided to answer the question about `range(of:options:locale:)` first, as I thought a quick read of its [documentation](https://developer.apple.com/documentation/foundation/nsstring/1417348-range) should reveal the answer. I knew the `locale` parameter is used for locale-sensitive string comparisons, but I was not sure whether it is still taken into account if it were a regex. The docs for the `locale` parameter said:

> The locale argument affects the equality checking algorithm. For example, for the Turkish locale, case-insensitive compare matches “I” to “ı” (`U+0131 LATIN SMALL DOTLESS I`), not the normal “i” character.

That's nothing surprising. I then looked at the [docs](https://developer.apple.com/documentation/foundation/nsstring/compareoptions/1410450-regularexpression) for `.regularExpression`:

> The search string is treated as an ICU-compatible regular expression. If set, no other options can apply except `caseInsensitive` and `anchored`. You can use this option only with the `rangeOfString` methods and `replacingOccurrences(of:with:options:range:)`.

It doesn't say anything about ignoring locales, but then regexes are usually locale-insensitive, so I thought I'd better test it out myself.

{% highlight swift %}
// non nil
print("I".range(of: "(?i)i", options: .regularExpression, range: nil, locale: Locale(identifier: "tr-TR")))
// nil
print("I".range(of: "(?i)ı", options: .regularExpression, range: nil, locale: Locale(identifier: "tr-TR")))
// non nil
print("I".range(of: "ı", options: .caseInsensitive, range: nil, locale: Locale(identifier: "tr-TR")))
// nil
print("I".range(of: "i", options: .caseInsensitive, range: nil, locale: Locale(identifier: "tr-TR")))
{% endhighlight %}

It does seem like the locale is being ignored.

### Matching Only Danish Names

Moving on to the problem of "how to actually match Danish names", my first thought was natural language detection. I looked up the `NaturalLanguage` framework, and found [`NLLanguageRecognizer`](https://developer.apple.com/documentation/naturallanguage/nllanguagerecognizer). After a quick look at its documentation, I tried it on the name given in the question:

{% highlight swift %}
// da
print(NLLanguageRecognizer.dominantLanguage(for: "Lærke")!.rawValue)
{% endhighlight %}

YAY that seemed to work! Just to be sure, I looked up some more Danish names and checked each of them:

{% highlight swift %}
// en
NLLanguageRecognizer.dominantLanguage(for: "William")?.rawValue
// en
NLLanguageRecognizer.dominantLanguage(for: "Noah")?.rawValue
// pt
NLLanguageRecognizer.dominantLanguage(for: "Oscar")?.rawValue
// es
NLLanguageRecognizer.dominantLanguage(for: "Lucas")?.rawValue
{% endhighlight %}

Sigh, none of them are considered Danish by `NLLanguageRecognizer`. I also considered giving it some `languageHints` and `languageConstraints`, but that made it too tolerant, calling every name Danish. `NLLanguageRecognizer` probably isn't suitable for recognising single words like this.

### Matching Only Danish *Letters*

Then I thought to myself, if OP is using regex, they probably meant to match "letters that are in the Danish alphabet", rather than "names that have a Danish origin". I quickly went to Wikipedia and looked up what the Danish alphabet. Apparently, it is just A-Z, plus Å, Æ, and Ø. I guess they just want a way to generate these characters in a non-hardcoded way.

I knew that in some regex flavours, you can match against specific Unicode properties with `\p`. I'm not sure if the "ICU-compatible" flavour, which is what `range(of:options:locale:)` appears to be using, supports `\p`. I quickly [looked it up](http://userguide.icu-project.org/strings/regexp), and it does support `\p`! Now I just need to find a Unicode Property that all characters in the Danish alphabet have, but no other characters have.

The closest that I could find was the "Script" property. All Danish letters are in the Latin script, but there are non Danish letters that are in the Latin script, like the dotless i. I thought maybe regex isn't the right way to go.

Then I had one hell of an idea. I remembered that `Locale` has a property that gives the set of characters used in the language of that locale. I quickly looked it up, and found it's called `exemplarCharacterSet`. I tried printing the one for Danish, but the `description` property for `CharacterSet` isn't very helpful, so I found an [SO answer](https://stackoverflow.com/a/55476889/5133585) that converts a `CharacterSet` to a more readable format.

I tried Danish:

{% highlight swift %}
print(String.UnicodeScalarView(Locale(identifier: "da-dk").exemplarCharacterSet!.allUnicodeScalars()))
// ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyzÅÆØåæø
{% endhighlight %}

That seems to work! OP just needs to check each character in the string, rather than using a regex.

I tried a few locales that are known to have really large character sets, namely Chinese, Japanese and Korean. For the first two, only a handful of characters are in the set, but for Korean, a _buttload_ of Hangul got printed.

### Silly Me!

That's when I thought "that's good enough", and posted my answer. A while after that, the question got closed as a duplicate of another question about `\p{L}`. Indeed, the OP probably just needed `\p{L}` (match all letters, no matter the language), but I somehow interpreted the question in a _really_ weird way. Now that I think about it, who the hell would only want to match Danish letters? "All letters" is far more likely!