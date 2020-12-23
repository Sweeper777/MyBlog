---
tags: swift language-feature
op_link: https://stackoverflow.com/q/65404303/5133585
op_profile_link: https://stackoverflow.com/users/8510707/mrd2242
op_name: "mrd2242"
---

### Premise

OP wonders what the difference between these three code snippets are:

{% highlight swift %}
let someInt: Int? = 42
if let x = someInt {
  print(x)
}
// ----
let someInt: Int? = 42
if case .some(let x) = someInt {
  print(x)
}
// ----
let someInt: Int? = 42
if case let x? = someInt {
  print(x)
}
{% endhighlight %}

and which one is preferred. They also wonder why this code "doesn't work":

{% highlight swift %}
let someInt: Int? = 42
var x: Int?
if case x? = someInt {
  print(x)
}
{% endhighlight %}

### How Am I Supposed To Answer This?

At first, I didn't really know what to say for this question, other than "yes, they are the same", so I thought I should start from "why the fourth code snippet doesn't work". I have never seen a pattern like that, so I thought it must be violating some syntax rules. I just need to find the grammar production rules for the syntax used for these patterns. 

### Looking Up Production Rules

Naturally, I went to the [Patterns](https://docs.swift.org/swift-book/ReferenceManual/Patterns.html) page in the Swift Reference. There I found syntax for the enum case pattern and the optional pattern:

```
enum-case-pattern → type-identifier(opt) . enum-case-name tuple-pattern(opt)
optional-pattern → identifier-pattern ?
```

But I couldn't find anything about optional binding in that page. That's when I realised optional binding isn't actually a pattern. After all, you can't use it in places where patterns can normally be used, such as in a switch statement.

I can't do a full text search in the whole Swift Reference, so I went to "[Branch Statements](https://docs.swift.org/swift-book/ReferenceManual/Statements.html#ID434)" where they talk about if statements. They've gotta mention something about optional binding there, I thought. I looked at the production rules for the if statement:

```
if-statement → if condition-list code-block else-clause(opt)
condition-list → condition | condition , condition-list
condition → expression | availability-condition | case-condition | optional-binding-condition
case-condition → case pattern initializer
optional-binding-condition → let pattern initializer | var pattern initializer
```

`case x? = someInt` is not a valid `case-condition` because `x?` is not a valid optional pattern, is what I thought the explanation was. It was much later that I realised I was wrong.

### Explaining The Three Same Code Snippets

With the fourth code snippet being taken care of, I thought of something that I could say other than "they are the same" for the first three code snippets, and that is "it is recommended to use optional binding in this case". To argue that, I just need to find a quote in the Swift Guide/Reference that says something to that effect.

Searching for "Optional Binding" on the "Statements" Swift Reference page gave me a link to [Optional Binding](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html#ID333), which is in the Swift Guide. I got the quote I wanted there.

The enum case pattern works because `Optional` is an enum. That's easy to explain. Now I just need to explain the optional pattern. This should be easy as a quote from the Swift Reference that talks about what it does should do the job. So I went back to [Optional Pattern](https://docs.swift.org/swift-book/ReferenceManual/Patterns.html#ID520), and had a little read at what it says there. Apparently, optional patterns are just syntactic sugar for enum case patterns. Great! This is very useful.

### My Theory Was Wrong!

After posting the answer, a comment from the OP made me realise that I forgot to explain why the fourth code snippet doesn't work. Quickly, I copied the necessary production rules into my answer to show the OP that the code is syntactically wrong. Just as I was doing the production rules for the optional pattern...

```
optional-pattern → identifier-pattern ?
identifier-pattern → identifier
```

**Hang on!** So `x?` is a valid optional pattern after all!

I immediately copied the fourth code snippet into an Xcode Playground, and **it does compile**, but it prints nothing. I thought the OP probably wanted it to print 42, and "doesn't work" means "prints nothing" here.

Just when I was about to scroll down from "Identifier Pattern" section, I saw a section called "Value Binding Pattern". After some rethinking about what the optional pattern does ("`x?` matches `.some(x)`"), I finally realised that I was totally wrong, and `let x?` actually has 3 nested patterns:

```
Value Binding Pattern: let x?
    Optional Pattern: x?
        Identifier Pattern: x
```

with the outermost pattern being a value binding pattern, rather than an optional pattern, like I first thought.