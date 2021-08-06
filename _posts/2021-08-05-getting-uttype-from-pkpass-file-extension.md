---
tags: swift
op_link: https://stackoverflow.com/q/68663474/5133585
op_profile_link: https://stackoverflow.com/users/353911/bill-chan
op_name: "Bill Chan"
title: "Getting UTType from .pkpass File Extension"
---

### Premise

OP is trying to convert a given file extension to a MIME type. They have written this method:

{% highlight swift %}
private class func mime(from utType: UTType) -> String? {
    let mimeType: String

    if let preferredMIMEType = utType.preferredMIMEType {
        mimeType = preferredMIMEType
    } else {
        return nil
    }

    return mimeType
}

public class func convertToMime(fileExtension: String) -> String? {
    var utType: UTType? = UTType(filenameExtension: fileExtension)

        
    var mimeType: String?
    if let utType = utType {
        mimeType = mime(from: utType)
    }
                    
    return mimeType
}
{% endhighlight %}

The tests pass, except on `pkpass`:

{% highlight swift %}
XCTAssertEqual(UTIHelper.convertToMime(fileExtension: "pkpass"), 
               "application/vnd.apple.pkpass")
// Actual: nil
{% endhighlight %}

OP wonders why this is the case, and how they can pass the test.

### Does the Docs Say Anything?

First, I tried to "minimise" the code. The code is essentially doing `UTType(fileExtension: "pkpass")?.preferredMIMEType` and that produces nil. I confirmed this in a playground.

I looked at the documentation of the `UTType` initialiser that I'm calling. I did this by just adding `.init` between `UTType` and the opening bracket in Xcode, and the help panel showed me the initialiser [documentation](https://developer.apple.com/documentation/uniformtypeidentifiers/uttype/3551510-init). Apparently, this is equivalent to calling:

{% highlight swift %}
UTType(tag: filenameExtension, tagClass: .filenameExtension, conformingTo: supertype)
{% endhighlight %}

I was wondering where the `supertype` argument came from, and that is when I realised, the initialiser actually had an optional second argument:

{% highlight swift %}
init?(filenameExtension: String, 
    conformingTo supertype: UTType = .data)
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
{% endhighlight %}

Oh! So this only finds `UTType`s that conforms to `public.data`!

### The UTType Hierarchy of pkpass

So I hypothesised that the reason why `preferredMIMEType` returns nil is because the actual `UTType` for pkpass doesn't conform to `public.data`. Well, if it isn't "data", then does that mean it's sort of like a package of many things? Like how a .jar file or a .framework file or a .app file is actually a whole bunch of files packaged together? I tried finding an example pkpass file online, unarchiving it with The Unarchiver, and it worked, so that has to be it, right? I looked at the list of static properties of `UTType`, and found `UTType.package`, so I tried:

{% highlight swift %}
print(UTType(filenameExtension: "pkpass", conformingTo: .package)?.preferredMIMEType ?? "nil")
{% endhighlight %}

and viola! It worked - printing the expected "application/vnd.apple.pkpass".

### What If I Don't Know The Supertype?

However, since OP is doing a "get the mime type from file extension" function, we don't know what we should pass for the supertype. I tried looking up "UTType hierarchy" trying to find a common supertype for all `UTType`s, but according to [this page](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_conc/understand_utis_conc.html), there are _two_ hierarchies, so what I wanted to find doesn't exist.

Then I remembered that I saw this other initialiser:

{% highlight swift %}
UTType(tag: filenameExtension, tagClass: .filenameExtension, conformingTo: supertype)
{% endhighlight %}

Looking at [its documentation](https://developer.apple.com/documentation/uniformtypeidentifiers/uttype/3551513-init), the `conformingTo` parameter is optional! I wonder what'd happen if I passed nil in there?

{% highlight swift %}
print(UTType(tag: "pkpass", tagClass: .filenameExtension, conformingTo: nil)?.preferredMIMEType ?? "nil")
{% endhighlight %}

This magically prints "application/vnd.apple.pkpass". Though it isn't documented, I think it makes sense to say that passing nil means "conforming to any `UTType`".

### What *Are* the Supertypes of These Things Anyway?

Out of interest, I also checked the superclasses of .jar, .pkpass, .framework and .app files by getting their `UTType`s using that initialiser. Here are the results:

- jar:
    - public.archive
    - public.item
    - public.data
    - public.executable
    - com.pkware.zip-archive
    - public.zip-archive
- pkpass:
    - public.directory
    - com.apple.bundle
    - public.item
    - com.apple.package
- framework:
    - public.item
    - com.apple.bundle
    - public.directory
- app:
    - com.apple.package
    - com.apple.application
    - public.item
    - public.directory
    - public.executable
    - com.apple.bundle
    - com.apple.localizable-name-bundle

Interestingly, .jar files seem to be the odd one out here. .jar files are the only one that has the super type `public.data`. The others all have `public.directory`.