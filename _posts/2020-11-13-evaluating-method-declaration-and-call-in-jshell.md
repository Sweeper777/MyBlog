---
categories: java
op_link: https://stackoverflow.com/q/64817719/5133585
op_profile_link: https://stackoverflow.com/users/994387/rahman-usta
op_name: "Rahman Usta"
title: "Evaluating Method Declaration + Call in JShell"
---

### Premise

OP can't run the following code programmatically using the JShell API:

{% highlight java %}
void hello(){
    System.out.println("Hello world");
}
                            
hello();
{% endhighlight %}

and asks for what the problem is.

I don't know anything about JShell API. I only know that JShell is a shell for Java.

### Fixing the Code

I did the obvious thing of copy-pasting OP's code into IntelliJ. 

{% highlight java %}
var code= """
          void hello(){
              System.out.println("Hello world");
          }
                                        
          hello();
          """;

JShell shell = JShell.create();
List<SnippetEvent> snippets = shell.eval(code);
{% endhighlight %}

I got some compile errors so I quickly made some changes:

{% highlight java %}
String code = "void hello(){\n" +
        "              System.out.println(\"Hello world\");\n" +
        "          }\n" +
        "                                        \n" +
        "          hello();";

JShell shell = JShell.create();
List<SnippetEvent> snippets = shell.eval(code);
{% endhighlight %}

This shouldn't change the behaviour, hopefully. Judging from the fact that `eval` returns a list of `SnippetEvent`s, I'm guessing it somehow represents the result of executing a snippet of Java code, so I added:

{% highlight java %}
System.out.println(snippets);
{% endhighlight %}

This prints:

```
[SnippetEvent(snippet=MethodSnippet:hello/()void-void hello(){
              System.out.println("Hello world");
          }
                                        
          hello();,previousStatus=NONEXISTENT,status=REJECTED,isSignatureChange=false,causeSnippetnull)]
```

I see that the snippet is `REJECTED`, so I guess this is what the OP is asking about. I don't actually get the compiler error that the OP mentioned though. 

### Reproducing the Problem

I also tried:

- removing the `hello()` call
- removing the `hello` declaration and calling `System.out.println` instead of `hello`
- running the method declaration snippet, then running the method call snippet in another call to `eval`
- running OP's code in the command line version of JShell

All of the above worked, and my code is not `REJECTED`, so it seems like that the CLI JShell is doing something different from the JShell API, and the JShell API doesn't like method declaration + call snippets (?)

### Exploring JShell

The next thing I tried is to see what I can do to a `SnippetEvent`, using IntelliJ's autocomplete. Other than methods that obviously get the stuff that's in the `toString` representation, I found that I can get its `value`:

{% highlight java %}
System.out.println(snippets.get(0).value());
{% endhighlight %}

But it is null. Well, the snippet is rejected after all. I also found that I can get the `snippet()` itself. I thought there are probably more things to explore in a `Snippet` object, so I looked at what methods are available on `Snippet`. I noticed that each snippet has a `kind`. Well, let's see what the `kind` of OP's snippet is:

{% highlight java %}
System.out.println(snippets.get(0).snippet().kind());
{% endhighlight %}

It's apparently a `METHOD`. I looked into the list of enum constants that `Snippet.Kind` has, and found a `STATEMENT` kind. Hmm, I thought, the call `hello();` is definitely of the kind `STATEMENT`. Therefore I hypothesised that `eval` doesn't like snippets with multiple kinds. `kind()` can only return one `Kind` after all.

To test this hypothesis, I thought two kinds of statements:

{% highlight java %}
List<SnippetEvent> snippets1 = shell.eval("var i = 10;");
System.out.println(snippets1.get(0).status()); // VALID
System.out.println(snippets1.get(0).snippet().kind()); // VAR
List<SnippetEvent> snippets2 = shell.eval("System.out.println();");
System.out.println(snippets2.get(0).status()); // VALID
System.out.println(snippets2.get(0).snippet().kind()); // STATEMENT
{% endhighlight %}

If I `eval` them at the same time...

{% highlight java %}
List<SnippetEvent> snippets1 = shell.eval("var i = 10; System.out.println();");
System.out.println(snippets1.get(0).status()); // VALID
System.out.println(snippets1.get(0).snippet().kind()); // VAR
{% endhighlight %}

Now that certainly isn't expected. What about...

{% highlight java %}
List<SnippetEvent> snippets1 = shell.eval("System.out.println(); var i = 10;");
System.out.println(snippets1.get(0).status()); // VALID
System.out.println(snippets1.get(0).snippet().kind()); // STATEMENT
{% endhighlight %}

I then tried a similar experiment on `subKind`, but it didn't confirm my hypothesis either. Hmm... seems like my hypothesis is wrong.

Since I explored pretty much all of the relevant parts of `SnippetEvent`, I trird somewhere else to explore - `JShell`. I see that it has a `diagnostics` method. Well, that can definitely help! It takes a `Snippet`, which I know how to get by now. It returns a `Stream`. Alright, I'll just print everything in the stream then!

{% highlight java %}
List<SnippetEvent> snippets1 = shell.eval(code);
shell.diagnostics(snippets1.get(0).snippet()).forEach(System.out::println);
{% endhighlight %}

Evaluting OP's code gives a single diagnostic:

> WrappedDiagnostic(Invalid method declaration; return type required: 126)

### A Little Clue

I was initially confused by this. What return type am I missing? Then I thought back to the fact that JShell thinks that the whole snippet's kind was `METHOD`, so maybe it thought that `hello();` was a method declaration? I verified this guess by writing a call next to a method declaration, and getting this exact error message:

{% highlight java %}
public class Main {

    public static void main(String[] args) throws FileNotFoundException {
        String code = "void hello(){\n" +
                "              System.out.println(\"Hello world\");\n" +
                "          }\n" +
                "                                        \n" +
                "          hello();";

        JShell shell = JShell.create();
        List<SnippetEvent> snippets1 = shell.eval(code);
        shell.diagnostics(snippets1.get(0).snippet()).forEach(System.out::println);
    }
    main(); // Invalid method declaration; return type required
}
{% endhighlight %}

Aha, so that's why the code snippet got rejected. The JShell probably tried to a stupid thing like that :) But I still don't know why it works on the CLI JShell. It might be that it splits the snippets into smaller parts whenever possible? Anyway, I determined that I have found enough info for me to post an answer.

### Google-fu

A few minutes after posting the answer, I realised I could just googled the [documentation](https://docs.oracle.com/javase/9/docs/api/jdk/jshell/JShell.html#eval-java.lang.String-) for `JShell.eval`, so I did, and oh boy it says it so clearly:

> The input should be exactly one complete snippet of source code, that is, one expression, statement, variable declaration, method declaration, class declaration, or import. To break arbitrary input into individual complete snippets, use `SourceCodeAnalysis.analyzeCompletion(String)`.

At that point, I felt so stupid.

In the end, I looked at the [docs](https://docs.oracle.com/javase/9/docs/api/jdk/jshell/SourceCodeAnalysis.html#analyzeCompletion-java.lang.String-) for `SourceCodeAnalysis.analyzeCompletion(String)`, quickly learned how it works using the docs, and edited my answer.