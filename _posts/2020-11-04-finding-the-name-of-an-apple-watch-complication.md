---
categories: swift
op_link: https://stackoverflow.com/q/64675063/5133585
op_profile_link: https://stackoverflow.com/users/6449292/ahmadreza
op_name: "Ahmadreza"
---

### Premise

OP found this gauge widget in [Apple's sample code](https://developer.apple.com/documentation/clockkit/creating_and_updating_complications):

![screenshot showing guage widget](https://i.stack.imgur.com/wwAO0l.png)

They want to know where the image for that gauge is stored.

I did not know what a "complication" is before this, nor did I have any experience working with ClockKit.

### First Thoughts

Immediately, I guessed that this is not an image, and it's probably not stored in the Asset Catalog like OP has thought. WatchKit probably draws the icon programmtically, so I am looking for a class that represents the icon. Maybe something like a `UIView` subclass, but for WatchKit, I thought.

### Checking Out the Sample Code

The natural next step is to check out the sample code. It has quite a lot of files:

![source files in the sample code](/assets/2020-11-04/sample-code-files.png)

But a lot of them can be ruled out. A lot of them are clearly model classes/structs. I checked each of the `XXXView.swift` files and they only contain some `Text`s and `Button`s. Then I remembered the title of the sample code "Creating and Updating Complications", so I should probably look at `ComplicationController.swift`. I didn't really know what a "complication" is, however, as I don't own an Apple Watch. That doesn't stop me from progressing though. I just assumed that it's a "thing".

My first impression of `ComplicationController.swift` is that it looked a lot like iOS 14 homescreen widgets, which I recently have tried making myself. I saw `CLKComplicationTimelineEntry`, so using my knowledge of homescreen widgets, I guessed that those represent the times at which the ~~widget~~ complication should be updated. But that has nothing with the question here, so I kept looking.

The next thing I saw was the `createTemplate` method returning a `CLKComplicationTemplate`. 

{% highlight swift %}
private func createTemplate(forComplication complication: CLKComplication, date: Date) -> CLKComplicationTemplate {
    switch complication.family {
    case .modularSmall:
        return createModularSmallTemplate(forDate: date)
    ...
{% endhighlight %}

I have no idea what `CLKComplicationTemplate` is just by looking at its name, so I looked at the [docs](https://developer.apple.com/documentation/clockkit/clkcomplicationtemplate):

> An abstract class that defines the base behavior for all templates.

That still doesn't tell me much, but since its an "abstract base class", I thought I should check out one of its subclasses, so I went into the method that the first return statement calls - `createModularSmallTemplate`. There, I saw that it returns a `CLKComplicationTemplateModularSmallStackText`. This is also when I realised this is not WatchKit, but ClockKit.

{% highlight swift %}
private func createModularSmallTemplate(forDate date: Date) -> CLKComplicationTemplate {
    // Create the data providers.
    let mgCaffeineProvider = CLKSimpleTextProvider(text: data.mgCaffeineString(atDate: date))
    let mgUnitProvider = CLKSimpleTextProvider(text: "mg Caffeine", shortText: "mg")
    
    // Create the template using the providers.
    let template = CLKComplicationTemplateModularSmallStackText()
    template.line1TextProvider = mgCaffeineProvider
    template.line2TextProvider = mgUnitProvider
    return template
}
{% endhighlight %}

I immediately looked at the [docs](https://developer.apple.com/documentation/clockkit/clkcomplicationtemplatemodularsmallstacktext) for that and it shows an example of an instance of this class as what seems like 2 labels on the top left of an Apple Watch. Ah, this is what a complication is. Finally something related to the question, I thought. Also by reading its docs, I guessed what the `createTemplate` method is doing. Apparently there are many "families" of complications, and the switch statement in `createTemplate` determines the correct template to return, as (I guess) each template has a family that it "fits".

I looked at a few other subclasses of `CLKComplicationTemplate`, and they all are parts of an Apple Watch, showing different kinds of UI element. I then guessed that the icon in the question must also be one of these `CLKComplicationTemplate`s. So I did a text search for "gauge" in `ComplicationController.swift`. Checking the documentation page for each template I found, I quickly found the answer:

```
CLKComplicationTemplateGraphicCircularOpenGaugeSimpleText
```

### Further discussions

After posting the answer, OP pointed out in the comments that it is not efficient for iPhone to call `createTemplate` from the complications list just to get a preview of the complication, because `createTemplate` reads from disk. I quickly traced through `createTemplate` to see if it really reads from disk. It eventually reaches some Combine related code, but I don't actually know what the data source is. 

My lack of knowledge in Combine stopped me from progressing further in this direction, so I thought I'll assume the OP's claim is true. I thought it's probably because it's sample code, so it doesn't have to be terribly efficient, but that's a bad argument IMO, so in the end I just said "you can make it more efficient yourself".