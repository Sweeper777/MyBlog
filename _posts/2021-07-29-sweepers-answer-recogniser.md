---
tags: misc
layout: post
title: "Sweeper's Answer Recogniser"
---

### Introduction

I was thinking about the thousands of answers I posted on Stack Overflow, and was wondering if I actually have a distinct "writing style". Then the shower thought of "I could try to create an neural network that identifies my SO answers" popped into my head.

In Xcode's Developer Tools menu, there is this thing called Create ML. It's basically a GUI app that trains CoreML models for you. The user just needs to provide the training data and click a few buttons. No ML knowledge required at all - exactly the kind of tool I like. I planned on using this to create a CoreML model that identifies whether a Stack Overflow answer is written by me or not.

For the training data, I needed two groups - one for "my answers" and the other for "not my answers". I called these "Yes" and "No" respectively. I know I can easily get the text data of the answers on Stack Overflow using [SEDE](https://data.stackexchange.com), but I'm not very familiar with the SEDE's database schemas, so I first looked for existing queries that finds all the answers of a user.

### Getting Data

I searched for "all answers", and after looking through the pages, found [this query](https://data.stackexchange.com/stackoverflow/query/1259936/find-all-answers-from-specified-userid). I changed it slightly to give me the bodies of the answers only:

{% highlight sql %}
SELECT 
    Ans.Body
FROM Posts as Ans
WHERE Ans.OwnerUserId = ##UserId## and PostTypeId = 2
ORDER BY Ans.CreationDate DESC
{% endhighlight %}

I tried running the query using my own user ID, and was pleasantly surprised by the fact that the answers' bodies are all in HTML instead of markdown. HTML can be easily parsed by e.g. Jsoup, but I don't know any library that can parse markdown, and I'd have to find such a library if it were in markdown.

Anyway, I downloaded the results as CSV, and opened it in a text editor to see what it looked like. It was a mess. Since there was only one column, it wasn't really "comma separated" at all. It was just a list of my answers in HTML, each one surrounded in quotes. The worst thing about it though, is that each answer spanned multiple lines, and the delimiter between each answer is also a newline character. It was really hard to tell where each answer started and ended. Since each answer was surrounded by quotes, I thought the quotes inside each answer would be escaped by backslashes or something like that, but nope, they are not.

### Cleaning It

Then I realised that I can split the file contents by the regex `"\n"`, i.e. try to match the ending quote of an answer, the newline that separates the answers, and the starting quote of the next answer. I'd still need to remove the starting quote from the first answer and the ending quote from the last answer, but I didn't really care at this point. The file format is just shitty. I tried the regex out in VSCode, and it reported more matches than the number of answers that I have. I suspected that this is because some of my answers also included this sequence of characters, probably in code blocks. I noticed that all my answers would all start with a `<p>` tag, so I used another regex, `"\n"(?!<)`, to find those extra matches that the first regex had found, which would not have been the start of an answer. 

As I thought, all those extra matches are in code blocks. I decided to use Jsoup to remove all the code blocks and blockquotes from my answers before giving them to Create ML, because those things aren't really a part of my "writing style". I wanted the NN to determine whether answer is mine based on the writing alone. Since I was going to delete the code blocks anyway, I deleted the code blocks that the extra matches are in, as I found them.

Then I just wrote some Kotlin code to split the CSV file into a bunch of smaller text files, so that Create ML can work with it, 

{% highlight kotlin %}
fun splitCSV() {
    val answers = Files.readString(Paths.get("QueryResults.csv")).split(Regex("\"\\n\""))
    answers.forEachIndexed { index, s ->
        var answer = s
        if (answer.startsWith("\"")) {
            answer = answer.drop(1)
        }
        if (answer.endsWith("\"")) {
            answer = answer.dropLast(1)
        }
        Files.writeString(Paths.get("TrainingData/Yes/$index"), answer)
    }
}
{% endhighlight %}

Then I removed the code blocks ans blockquotes of the html, and extracted the text content. I didn't actually know how to use Jsoup before this, so I quickly looked up a Jsoup tutorial. Apparently getting a `Document` object as simple as calling `Jsoup.parse`. Then I tried selecting all the `<pre>` and `<blockquote>`, and called `remove`. 

{% highlight kotlin %}
Jsoup.parse(html).select("pre").remove()
{% endhighlight %}

I thought I should check the docs of `remove` to see if it actually works the way I expect. That is when I noticed that it says I should not use `remove` to _clean_ HTML. That is when I remembered, that I saw a `clean` method when browsing through the methods in the `Jsoup` class. It apparently takes a `Whitelist`. After looking at some of the existing whitelists, I figured that `basic` without `pre` and `blockquote` is probably good enough, so this is what I did:

{% highlight kotlin %}
fun parseHTML() {
    var i = 0
    for (file in File("TrainingData/Yes/").walkTopDown()) {
        if (!file.isFile) {
            continue
        }
        val html = Files.readString(file.toPath())
        val doc = Jsoup.parse(Jsoup.clean(html, Whitelist.basic().removeTags("pre", "blockquote")))
        Files.writeString(Paths.get("TrainingData/Yes/$i.txt"), doc.text())
        i++
    }
}
{% endhighlight %}

### Getting Not-My-Answers

Now I needed to collect data for the "No" category - answers written by "not me". The hard part is that I had to be as unbiased as I can, and choose a "representative" sample of answers that is not written by me, for the NN to work well (I think). Since there are so many people that are not me, it's very hard to do this. If I only choose Jon Skeet's answers for example, then the NN would probably do something weird on other people's answers, instead of outputting a definitive "no". In the end, I just chose the 5000 most recent answers that have a score of 7. 5000, because that's roughly how many answers there are in the "Yes" group.

{% highlight sql %}
SELECT top 5000
    Ans.Body
FROM Posts as Ans
WHERE Ans.OwnerUserId <> ##UserId## and PostTypeId = 2 and Ans.Score = 7
ORDER BY Ans.CreationDate DESC
{% endhighlight %}

I downloaded the CSV again, put it through the same Kotlin scripts as the "Yes" group. I also downloaded some of Jon Skeet's answers as testing data. Now I can finally start training the model. 

### Training

I saw that there were 3 algorthms that I can use: Maximum Entropy, Conditional Random Field, and Transfer Learning. I don't know what any of them meantm, so I tried all 3. The first was the quickest to train, but the other two were a _lot_ slower. Though, no matter how long they took, none of them passed the tests completely. Even the best one only passed a little more than half of the tests. Maybe my writing style is very similar to Jon Skeet's? :-)

After some manual testing, I found that it is much more likely to give wrong results when it is someone else's answer in a tag that I answer in frequently as well, or when it is my answer in a tag that I don't usually answer in., and is more likely to give a definite "No" when the answer is about something that I am not familiar with. What I think has happened here is that the model has just picked up on the terms that often appears in my answers, not because I like to use those terms, but because they are related to the technology. Oh well, unintened consequences are inevitable.

You can download the CoreML models here:

- [Maximum Entropy](/assets/2021-07-29/SweepersAnswerRecogniser1.mlmodel) 
- [Conditional Random Field](/assets/2021-07-29/SweepersAnswerRecogniser2.mlmodel)
- [Transfer Learning](/assets/2021-07-29/SweepersAnswerRecogniser3.mlmodel)