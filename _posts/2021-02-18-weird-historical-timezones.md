---
tags: swift timezone locale
op_link: https://stackoverflow.com/q/66255217/5133585
op_profile_link: https://stackoverflow.com/users/1007522/user1007522
op_name: "user1007522"
---

### Premise

OP is trying to format historical dates. They saw that they have weird zone offsets. For example, this code:

{% highlight swift %}
extension String {
    func yearMonthDayDate() -> Date? {
        return DateFormatter.yearMonthDayFormatter.date(from: self)
    }
}

extension DateFormatter {
    static let yearMonthDayFormatter: DateFormatter = {
        let dateFormatter = DateFormatter()
        dateFormatter.timeZone = .current
        dateFormatter.dateFormat = "yyyy-MM-dd"
        dateFormatter.locale = .current
        return dateFormatter
    }()
}

extension Date {
    func zonedDate() -> String {
        let dateFormatter = DateFormatter()
        dateFormatter.timeZone = .current
        dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ssXXX"
        dateFormatter.locale = .current
        return dateFormatter.string(from: self)
    }
}
print("1000-06-17".yearMonthDayDate()!.zonedDate())
{% endhighlight %}

prints `1000-06-17T00:00:00+00:17`. The zone offset is 17 minutes ahead of GMT. OP asks why the offset is such a weird number.

### What Is Your Current Timezone?

I knew immediately that this is because everywhere around the world used to observe the local mean time, rather than standard timezone offsets, because I've played around with historical dates before. The only thing is, OP didn't say what timezone is `TimeZone.current`, so I was only 90% sure. If their actual current timezone _didn't_ use to observe a local mean time of 17 minutes ahead of Greenwich, then it would be something else, and I would bew wrong. Also, if I knew their current timezone, I would be able to find some sources saying that their local mean time was indeed 17 minutes.

Naturally, I posted a comment asking the OP what their timezone is, but they didn't respond in quite a few minutes. I was a bit impatient and thought, y'know what'd be cool, what if I can _guess_ what their timezone is? I am very good at trial and error, after all! :)

### Let's Guess...

The code I used to test my guesses is this Java code:

{% highlight java %}
System.out.println(ZoneId.of(someTimezoneId) // replace someTimezoneId with my guess
    .getRules().getOffset(LocalDate.of(1000, 1, 1).atStartOfDay()));
{% endhighlight %}

17 minutes ahead of Greenwich... That must be somewhere to the east of the UK, I thought. My first guess was France, so I tried `Europe/Paris`.

Nope, Paris is only 9 minutes ahead. Okay, let's go further east. How about `Europe/Berlin`?

Nope Berlin is almost an hour ahead!

That is when my geographical knowledge starts to run out. What major cities are there between Paris and Berlin? I have no idea. Well, I'll just go on Google Maps!

![Google Maps](/assets/2021-02-18/1.png)

Hmm... Brussels seems like the place. Just a little to the right of Paris, but not too far. Putting in `Europe/Brussels` gave me an offset of +00:17:30. The extra 30 seconds worried me for a bit. I thought `DateFormatter` would have rounded the 30 seconds up, making it 18 minutes. But as I looked around the map, I couldn't find a better match than Brussels.

I decided to take the bet and ask OP "is your timezone Belgium", and oh boy I was right!

### In Writing The Answer...

What's left should have been a few simple Google searches for e.g. sources that say the local mean time in Brussels is 17 minutes ahead of Greenwich, but for some reason I just couldn't find any! In the end I just quoted a few Wikipedia articles and made a reference to `tzdb`.

I also thought that it would be nice to have some code to demonstrate the historical offset. Since the question is tagged Swift, I thought I would just rewrite the code I used to check my guesses in Swift.

{% highlight swift %}
// some date in the 1000s
let date = Date(timeIntervalSinceNow: 86400 * 365 * -1000)
print(TimeZone(identifier: "Europe/Brussels")!.secondsFromGMT(for: date))
{% endhighlight %}

For some bizzare reason, it prints 0! I also tried printing the same thing for other timezones, and my guess is that `secondsFromGMT` always rounds the number of seconds to the nearest hour, or something... Sigh, I guess I'll be posting my Java code then...