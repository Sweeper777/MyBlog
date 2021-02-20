---
tags: swift misc
layout: post
---

I recently saw this tweet by Jeff Atwood:

<blockquote class="twitter-tweet" data-theme="dark"><p lang="en" dir="ltr">This is the calendar equivalent of going a bit too far with regular expressions in programming <a href="https://t.co/G6SOCZbj2i">pic.twitter.com/G6SOCZbj2i</a></p>&mdash; Jeff Atwood (@codinghorror) <a href="https://twitter.com/codinghorror/status/1356440700917702656?ref_src=twsrc%5Etfw">February 2, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Apparently he thought that was going too far, but IMO, that's a fantastic idea for a calendar! For god's sake, it shows information about the whole year with only about 90 squares!

I want to have this for 2022, and 2023, and all the years in the future too, I thought, so I decided to write an iOS app that generates such a beautiful thing for any given year.

First I gotta figure out how to do the grid. I remember from one of my previous projects that there is a CocoaPod called [GridLayout](https://github.com/mihaimihaila/GridLayout) that can position views in a grid just like this. I installed that pod, and quickly added some labels to the grid. After messing with the alignment for a bit, I got:

![labels in a grid, with no padding](/assets/2021-02-20/1.png)

{% highlight swift %}
var white = true
var gridItems = [GridItem]()
for i in 0..<31 {
    gridItems.append(
        GridItem(generateLabel(
                    text: "\(i + 1)",
                    textColor: white ? .black : .white,
                    bgColor: white ? .white : .black),
                    row: i % 7, column: i / 7,
                    horizontalAlignment: .stretch)
    )
    white.toggle()
}
let grid = UIView.gridLayoutView(items: gridItems,
                                    rows: Array(repeating: .fill, count: 7),
                                    columns: Array(repeating: .fill, count: 5))
{% endhighlight %}

It appears that `UILabel`s don't have padding... This is not like CSS where I can just say `padding: 10px;` and it would be magically added... So I looked on Stack Overflow, found [this question](https://stackoverflow.com/questions/27459746/adding-space-padding-to-a-uilabel), which led me to a solution - the CocoaPod [PaddingLabel](https://github.com/levantAJ/PaddingLabel).

After that, I also added [SwiftDate](https://github.com/malcommac/SwiftDate), just to simplify the date operations. The rest is rather simple.

Here's the final result:

![final result](/assets/2021-02-20/2.png)

The code can be found [here](https://github.com/Sweeper777/CompactCalendarGenerator).

Just after that, I saw another tweet:

<blockquote class="twitter-tweet" data-conversation="none" data-theme="dark"><p lang="en" dir="ltr">As <a href="https://twitter.com/kmett?ref_src=twsrc%5Etfw">@kmett</a> suggested, the transpose is much more legible <a href="https://t.co/qTN6jwSVqB">pic.twitter.com/qTN6jwSVqB</a></p>&mdash; Mark Jackson-Brown (@Markster3000) <a href="https://twitter.com/Markster3000/status/1356646037348380673?ref_src=twsrc%5Etfw">February 2, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Well, time to do the transpose!