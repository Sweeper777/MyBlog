---
tags: java
op_link: https://stackoverflow.com/q/68406016/5133585
op_profile_link: https://stackoverflow.com/users/13123964/jsteltze
op_name: "jsteltze"
title: "Graphics2D fillRect vs clearRect"
---

### Premise

OP was using the `Graphics2D` API to create a `BufferedImage`. They did:

{% highlight java %}
Color someHalfTransparentColor = new Color(Integer.parseInt("77affe07", 16), true);
BufferedImage bi = new BufferedImage(10, 10, BufferedImage.TYPE_4BYTE_ABGR);
Graphics2D g = bi.createGraphics();
g.setColor(someHalfTransparentColor);
g.fillRect(0, 0, 10, 10);
g.dispose();
        
System.out.println(Integer.toString(bi.getRGB(0, 0), 16));
{% endhighlight %}

They found that the output was `77b0ff06`, rather than the hex value of `someHalfTransparentColor`, with which they called `setColor`.

On the other hand, if they used `setBackground` and `clearRect`, rather than `setColor` and `fillRect`, the output is expected:

{% highlight java %}
g.setBackground(someHalfTransparentColor);
g.clearRect(0, 0, 10, 10);
{% endhighlight %}

So they ask what exactly is the difference between these two methods.

### The First Very Bad Guess

The first thing I did was of course to reproduce OP's behaviour. The code that OP provided could be easily copy and pasted into a `main` method and run, so that was very nice of them :) I reproduced the behaviour successfully and started reading the code. I first noticed that OP gets the most top left pixel of the image. I thought that this might be reason - the edges of the rectangle usually are not the same color as the center of the rectangle. From experience, I knew that some image editors will somehow "blur" the edges of a rectangle, so I tried to print out pixel at the center of the image instead:

{% highlight java %}
System.out.println(Integer.toString(bi.getRGB(5, 5), 16));
{% endhighlight %}

Still the same output. (Now thinking back, I was probably thinking about the artefacts created when doing bicubic scaling.)

Looking at the documentations for [`clearRect`](https://docs.oracle.com/javase/7/docs/api/java/awt/Graphics.html#clearRect(int,%20int,%20int,%20int)) and [`fillRect`](https://docs.oracle.com/javase/7/docs/api/java/awt/Graphics.html#fillRect(int,%20int,%20int,%20int)), `fillRect` didn't say anything relevant to this question, but `clearRect` did say something that caught my eye:

> This operation does not use the current paint mode.

Well, that's definitely a _difference_, but it raises more questions. _What is a paint mode?_ And what paint mode _does_ `clearRect` use, if not the "current" one?

### What Is a "Paint Mode"?

I simply did a text search for "paintmode" on the page, and found the [`setPaintMode`](https://docs.oracle.com/javase/7/docs/api/java/awt/Graphics.html#setPaintMode()) method. To my surprise, it doesn't take any parameters! My interpretation of what this method is therefore "changes the graphics into a mode called 'paint'", rather than "sets the 'paint mode' of the graphics to something". I'm not sure if this is the same thing that the documentation of `fillRect` is talking about, but I looked at its documentation anyway:

> Sets the paint mode of this graphics context to overwrite the destination with this graphics context's current color. This sets the logical pixel operation function to the paint or overwrite mode. All subsequent rendering operations will overwrite the destination with the current color.

Okay... so this is the "paint mode", but more importantly, seeing the word "paint" here is when I realised, we are painting a _half transparent_ color _on another color_, and that's what `fillRect` is doing. Naturally, the resulting color is a mixture of initial color of the image, and `someHalfTransparentColor`. The "paint" mode is probably "mix what was already there with the color we are drawing", and now I just need to find the other mode (that `clearRect` uses)!

I looked for another `setXXXMode` method, and found `setXORMode`. I thought, this must be the mode where it alternates between two colors! I tried to make it alternate between the initial color of the image, and whatever `getColor` is:

{% highlight java %}
g.setXORMode(new Color(0, true));
{% endhighlight %}

This produced the same color as `someHalfTransparentColor`, but without the alpha component. I felt like I was very close.

### Stepping Into clearRect

Running out of ideas, I went brute force and used the debugger to step into `clearRect` and see exactly what happens in there. I wanted to see exactly how it "ignores the current paint mode". The debugger brought me to `SunGraphics2D.class`, where I was shown this:

{% highlight java %}
public void clearRect(int x, int y, int w, int h) {
    // REMIND: has some "interesting" consequences if threads are
    // not synchronized
    Composite c = composite;
    Paint p = paint;
    setComposite(AlphaComposite.Src);
    setColor(getBackground());
    fillRect(x, y, w, h);
    setPaint(p);
    setComposite(c);
}
{% endhighlight %}

It temporarily sets the paint to background color, _and the composite to_ `AlphaComposite.Src`, then sets them back to what they were originally. This is interesting, because it doesn't call any of the methods that mentions "paint mode" (`setPaintMode` or `setXORMode`). Are the paint and composite part of the "paint mode" perhaps? The documentation doesn't say.

I looked at what `AlphaComposite.Src` does. Apparently, it implements the [`SRC`](https://docs.oracle.com/javase/7/docs/api/java/awt/AlphaComposite.html#SRC) rule, which is:

> The source is copied to the destination (Porter-Duff Source rule). The destination is not used as input.

That seems to match my expectations, assuming that "destination" are the pixels that we are drawing on top of, and "source" are the pixels that we are drawing. I scrolled up to find that the default composite rule is [`SRC_OVER`](https://docs.oracle.com/javase/7/docs/api/java/awt/AlphaComposite.html#SRC_OVER), which does take the input into account. 

I tried to substitute in some numbers into the equations given in the documentation there to check, but after scrolling up to the top, I realised that the process is much more complicated than that, so I decided to just _assume_ that the maths all work out. I've already invested too much time into this.

I tried to reproduce the behaviour of `clearRect` using `fillRect` by first setting the composite to `AlphaComposite.Src`, and it worked!

{% highlight java %}
g.setComposite(AlphaComposite.Src);
g.setColor(someHalfTransparentColor);
g.fillRect(0, 0, 10, 10);
{% endhighlight %}

This difference in `fillRect` and `clearRect` is technically undocumented though (which is quite rare for something in the standard library), unless composite is part of the "paint mode".