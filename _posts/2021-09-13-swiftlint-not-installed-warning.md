---
tags: swift library xcode
op_link: https://stackoverflow.com/q/69157015/5133585
op_profile_link: https://stackoverflow.com/users/16602527/eric
op_name: "Eric"
title: "The 'SwiftLint Not Installed' Warning"
---

### Premise

OP installed SwiftLint via CocoaPods, and now wants to remove it. They removed the line from their Podfile, and ran `pod install`, but now they get a warning:

> SwiftLint not installed, download from https://github.com/realm/SwiftLint

They wonder how they can get rid of the warning.

I have never used SwiftLint, but I do know that it is a linter for Swift.

### Xcode Plugin?

I know the Swift compiler doesn't check for whether SwiftLint is installed on its own, so the warning must be produced by something else. I guessed that OP must have installed some sort of Xcode plugin that works together with the CocoaPod, or something like that. Linters usually need things like that, as far as I know. So I checked to see if there is an Xcode plugin for SwiftLint, by looking up "swiftlint xcode plugin".

I found [SwiftLintXcode](https://github.com/ypresto/SwiftLintXcode), which is quite old, and I doubt OP is using it. I then went to the [SwiftLint repo](https://github.com/realm/SwiftLint) to see if their README says anything about installing it in Xcode.

### A New Build Phase

To my surprise, the README says that I should add a "Run Script" build phase that runs a script that outputs this exact warning if `swiftlint` is not found on the PATH:

{% highlight shell %}
if which swiftlint >/dev/null; then
  swiftlint
else
  echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
{% endhighlight %}

OP must have followed the instructions in the README, and added this build phase, then forgot about it, I thought.

I tried to do a exact text search for this warning on Google, to see if there is anything else that would produce this, but all the results seem to be about installing SwiftLint, so it seems like this is really a build phase that you add.

Now that's figured out, the answer is simple - just remove the build phase!

### Something Is Weird

Just after I posted the answer, though, I noticed something strange. OP said that they installed SwiftLint via CocoaPods, but CocoaPods never adds anything to your PATH! How would the `which swiftlint` command _ever_ succeed? For the script to work, OP would need to install SwiftLint via HomeBrew or something like that. And according to the README file, the correct script for the new build phase if SwiftLint was installed via CocoaPods should be:

{% highlight shell %}
"${PODS_ROOT}/SwiftLint/swiftlint"
{% endhighlight %}

That seems like it just runs an executable file in the Pods/SwiftLint directory directly. If OP were following the instructions in the README word for word, then they would have used this instead, and when the SwiftLint pod got removed, the Pods/SwiftLint directory would not exist anymore, and they should get a "run script phase failed with exit code..." _error_, rather than a warning...

Then does that mean OP wrongly added the script meant for non-CocoaPods installations? Well, in that case they would have gotten the warning even when the SwiftLint pod is installed, since `swiftlint` is not on their PATH. Does that mean they also installed it with HomeBrew or another package manager? But then removing the _pod_ shouldn't cause the warning to appear!

At this point I was very confused, so I just gave up trying to figure out what exactly OP did.

### Big Reveal

Eventually, OP accepted the answer and explained that this is not actually their project, but a project they downloaded from the Internet. Still, what exactly happened doesn't quite make sense to me. My best guess is that the original author of the project installed SwiftLint both via CocoaPods and via HomeBrew, and added the script that checks if `swiftlint` is on the PATH. OP's machine doesn't have `swiftlint` on the PATH. When OP downloaded the project, the first thing they did was removing the SwiftLint pod, so they didn't realise that the warning exists even when the pod was installed.

That's enough speculation for today.