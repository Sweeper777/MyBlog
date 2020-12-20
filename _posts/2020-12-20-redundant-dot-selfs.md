---
tags: swift language-feature
op_link: https://stackoverflow.com/q/65369113/5133585
op_profile_link: https://stackoverflow.com/users/14531220/zeytin
op_name: "zeytin"
title: "Redundant '.self's"
---

### Premise

OP found that you can add `.self` to any expression in Swift, as many times as you like. They wonder repeating how many times is preferred.

My first reaction was that one should not use `.self` after an expression at all. This is different from the `.self` after a type name, which is the way to get a metatype object from a type name. `.self` after an expression is just redundant. 

To argue that it is redundant, I need to show that the reason the feature exists is not to make your code look prettier or anything like that. I then thought that maybe this feature is just there for the completeness of other features. For example, I hypothesized that if `self` is a member of everything, then it is a key path of everything, which is the reason why the keypath expression `\.self` is valid.

But before I went any further, I suspected that this is a duplicate. `self` is a pretty common thing, so someone else probably asked about it before. I looked up `swift .self`, and found this [SO post](https://stackoverflow.com/q/26835013/5133585). That seems to be about the `self` used on its own or as a prefix, so I hesitated to close it. However, [one of the answers](https://stackoverflow.com/a/26835100/5133585) mentions a quote:

> The self Property Every instance of a type has an implicit property called `self`, which is exactly equivalent to the instance itself. 

That sounds like evidence for my hypothesis. I scrolled down to find out where the quote is from, as the source could mention reasons why this feature exists. I need the reason in anyway. The source is apparently a really old version (Swift 2) of the language guide. I went to the [current version](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html) and tried to find the quoted text, but can't find it anywhere. It seems like that part has been removed between now and then. :(

Now that I am in the language guide, I thought I might as well check out what the [language *reference*](https://docs.swift.org/swift-book/ReferenceManual/AboutTheLanguageReference.html) has to say about `.self`. They are on the same site after all.

So I Cmd+F'ed for `.self` in all the pages of the language reference. [The first thing I found](file:///Library/Apple/System/Library/StagedFrameworks/Safari/WebInspectorUI.framework/Resources/Main.html#ID563) disproved my hypothesis: `self` is hardcoded into the syntax of key path expressions, and it even has a special name - the identity key path:

> The *path* can refer to `self` to create the identity key path (`\.self`). The identity key path refers to a whole instance, so you can use it to access and change all of the data stored in a variable in a single step.

```
key-path-expression → \ type opt . key-path-components
key-path-components → key-path-component | key-path-component . key-path-components
key-path-component → identifier key-path-postfixes opt | key-path-postfixes
key-path-postfixes → key-path-postfix key-path-postfixes opt
key-path-postfix → ? | ! | self | [ function-call-argument-list ]
```

I continued looking, and eventually I found what this `.self` thing is called - a ["Postfix Self Expression"](https://docs.swift.org/swift-book/ReferenceManual/Expressions.html#ID401). Unfortunately, that section of the language reference doesn't say anything about why it was added into the language. Looking at the bright side, I now know what this is called! 

Using the name of this, I then searched `use of postfix self swift` to find [this swift evolution proposal](https://forums.swift.org/t/grammar-of-postfix-self-expression/39316/2), where they discussed changing `.self` after a type name to `.type`. In a comment, [Jorden Rose](https://forums.swift.org/u/jrose) mentioned that `.self` exists because of historical, Objective-C reasons. That sounds like exactly what I need to make my point!