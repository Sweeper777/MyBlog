---
tags: swift library
op_link: https://stackoverflow.com/q/68018900/5133585
op_profile_link: https://stackoverflow.com/users/5877363/n-fredman
op_name: "N.Fredman"
title: "Encoding URLs In a Strange Way"
---

### Premise

OP noticed that the URL `zerø.info`, when put into Chrome, gets percent-encoded into `zer%c3%b8.info`, then it gets turned into something else - `xn--zer-2na.info`. They want to convert URLs into this format.

{% highlight swift %}
urlStr.addingPercentEncoding(withAllowedCharacters: .urlHostAllowed)
{% endhighlight %}

only produces the percent-encoded form.

### What's This Called Again?

Just a few days before answering this question, I watched a YouTube video about recognising scam emails, and I've seen something similar to that format in that video. The context was about how scammers can buy domain names that has Unicode characters that look very similar to legit domain names, and only few email software detect this and warns the user about this effectively. It was mentioned that many web browsers display these domain names in a different format to distinguish them from pure ASCII domain names, and that format was very similar to what the OP is asking here. The name of the format was mentioned, but I could not remember what it is no matter how hard I try. It's literally at the tip of my tongue though.

I looked up "domain name encoding" on Google and the second result was the Wikipedia page for [Punycode](https://en.wikipedia.org/wiki/Punycode). I thought, yes! That's what it's called! Punycode!

### Generating Punycode

I immediately googled ways to generate Punycode in Swift, and found [PunycodeSwift](https://github.com/gumob/PunycodeSwift). In one of the examples there, I saw:

{% highlight swift %}
var sushi: String = "寿司"

sushi = sushi.idnaEncoded!
print(sushi)  // xn--sprr0q
{% endhighlight %}

That seems like exactly what OP wants! The property is called `idnaEncoded`, not "Punycode", which got me a bit confused, but as far as I was concerned, doing `"zerø".idnaEncoded` should produce `xn--zer-2na`.

### Confused

But what about `info`? Looking at the "Examples" section on the wiki page, it should have a `-` suffix! (At this point I kind of just assumed that the transformation is applied to each dot-separated component in order) Maybe the transformation is optional for ASCII-only components? I tried putting `xn--zer-2na.info-` into the address bar of my browser, but it didn't recognise it as a URL. Maybe not then...

I continue reading the wiki page for Punycode, and saw the "Internationalised Domain Names" section:

> To prevent non-international domain names containing hyphens from being accidentally interpreted as Punycode, international domain name Punycode sequences have a so-called ASCII Compatible Encoding (ACE) prefix, "xn--", prepended.Thus the domain name "bücher.tld" would be represented in ASCII as "xn--bcher-kva.tld".

That kind of confused me even more. So the actual Punycode for `zerø` is only `zer-2na`, and the `xn--` part is just a constant prefix? I still don't know why `info` stays as `info`. Is it because it's the TLD, and the transformation only applies to non-TLDs? That would mean TLDs can't have Unicode, which I wasn't quite sure about.

I then saw that the "Internationalised Domain Names" section has a main article - [Internationalized domain name](https://en.wikipedia.org/wiki/Internationalized_domain_name), which, my god, is filled with answers to my questions.

### IDNA Encoding

It has a section called "ToASCII and ToUnicode". I thought it would describe in detail how the encoding process is done. 

> These algorithms are not applied to the domain name as a whole, but rather to individual labels. For example, if the domain name is www.example.com, then the labels are www, example, and com. ToASCII or ToUnicode is applied to each of these three separately.
> 
> [...]
>
> ToASCII leaves unchanged any ASCII label, but will fail if the label is unsuitable for the Domain Name System. [...]

Oh boy, not only did it confirm my assumption that the Punycode transformation is applied to each dot separated component, it also explains why `info` is unchanged!

I was ready to write an algorithm for the OP that uses PunycodeSwift to get the Punycode at this point (splitting by `.`, finding the components with Unicode, converting to Punycode, adding `xn--`, then joining everything with `.`), but when I scrolled down, I saw a section called "Example of IDNA encoding". I thought, hold on, so _that's_ what the `idnaEncoded` property returns!? _That's_ why it's called this weird name!?

I thought it would be worth it to download the pod and try this out for myself, and gosh, `idnaEncoded` does all the splitting, joining and error handling for me. It's the _exact thing_ OP needs!