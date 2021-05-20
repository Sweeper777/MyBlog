---
tags: swift locale
op_link: https://stackoverflow.com/q/67597756/5133585
op_profile_link: https://stackoverflow.com/users/1199106/s-basnagoda
op_name: "S.Basnagoda"
---

### Premise

OP wants to only allow the user to enter one single Katakana in a `UITextField`. They also want it to work with the Romaji keyboard.

They tried to override `shouldChangeCharactersInRange`:

{% highlight swift %}
func textField(_ textField: UITextField, shouldChangeCharactersIn range: NSRange, replacementString string: String) -> Bool {
    if string.rangeOfCharacter(from: .init(charactersIn: 
        "アイウエオカキクケコサシスセソタチツテトナニヌネノハヒフヘホマミムメモヤユヨラリルレロワヲン"
        )) != nil || string == ""{
        return true
    } else {
        return false
    }
}
{% endhighlight %}

But this didn't work.

### How Romaji Keyboards Work

I can immediately see why that would not work. The way the Romaji keyboard works is, you type latin letters, and the keyboard (IME) will gradually change those letters into Hiragana. It will also give you a list of things that can be spelled with what you already typed, and you can tap on one of the options to replace what you already typed with that option. That list of things will include the Katakana counterparts of the Hiragana you typed, and that is how you type Katakana. So to type Katakana with the Romaji keyboard, you must be allowed to type Latin letters and Hiragana, otherwise you can't type Katakana at all. OP's code literally only allows Katakana to be entered.

### Fixing the Code With markedTextRange

I copied and pasted OP's attempt into Xcode to start doing some trial and error. That is when I realised OP also didn't check the length of the resulting string, so I immediately added that check. I also didn't like `rangeOfCharacter` for some reason, and changed to use `contains`.

{% highlight swift %}
let katakanas =
    "アイウエオカキクケコサシスセソタチツテトナニヌネノハヒフヘホマミムメモヤユヨラリルレロワヲン"
let newString = NSString(string: textField.text ?? "").replacingCharacters(in: range, with: string)
return newString.count == 1 && katakanas.contains(newString.first!)
{% endhighlight %}

Now I need to also allow Latin letters and Hiragana, but only in the cases when the user is using them to enter a Katakana. OP mentiones the property `markedTextRange`, which seems to refer to the highlighted part of the text when I start typing using the Romaji keyboard. The highlighted part is where the list of options would be generated from. When I select an option, a part of the text would be unhighlighted, or in the API's terminology, unmarked.

I thought I could just check `markedTextRange` to distinguish between "entering an a Latin letter because they actually wanted to enter a Latin letter", and "entering a Latin letter because they wanted to enter a Katakana" - the former we need to disallow, the latter we need to allow. If `markedTextRange` is nil, we know the user is definitely not trying to enter a Katakana.

So I added a disjunct:

{% highlight swift %}
return (newString.count == 1 && katakanas.contains(newString.first!))
    || textField.markedTextRange != nil
{% endhighlight %}

I still can't type any non-Katakana! I used the debugger to see what's wrong. 

Apparently, when I start typing the first character, `shouldChangeCharactersInRange` gets called, but at that point, `markedTextRange` is nil, not (0, 1)! I guess this is understandable, as `shouldChangeCharactersInRange` is meant for asking the delegate "should I change this?" _before actually changing the text_, so naturally `markedTextRange` wouldn't change. I wish I can get more information about what this incoming change, like how it would change `markedTextRange` etc, in `shouldChangeCharactersInRange`, but alas that's not available.

### How About EditingChanged?

I then tried to listen for the `editingChanged` control event. I know that will happen _after_ the text has been changed, so `markedTextRange` is probably also updated too.

{% highlight swift %}
@objc func textFieldChanged() {
    let newString = textfield.text ?? ""
    if !(newString.count == 1 &&
            katakanas.contains(newString.first!)) && textfield.markedTextRange == nil {
        textfield.text = ""
    }
}
{% endhighlight %}

Since the text has already been changed, it's quite hard to revert back to the previous version, so I could do is to set it to empty when the text is not a Katakana... Then I realised that with this design, if you enter a Katakana, then starts to enter some other rubbish, the entire text would be deleted. 

### Other Extensions

Therefore, I added some code in `shouldChangeCharactersInRange` to disallow the user from entering any more characters when the text is already valid:

{% highlight swift %}
if let firstChar = textField.text?.first,
    (katakanas.contains(firstChar)) && string != "" {
    return false
}
return true
{% endhighlight %}

Now that I've done this, I thought it'd be a great idea if I disallow the user from entering any more, if there is already a single Hiragana in the text field, since a single Hiragana is enough for the corresponding Katakana option to show up, and the user can select that without typing anything extra.

At the time of writing the answer, I couldn't do it for some reason, but it is really as simple as changing the condition to:

{% highlight swift %}
if let firstChar = textField.text?.first,
    (katakanas.contains(firstChar) || hiraganas.contains(firstChar)) && string != "" {
{% endhighlight %}

After all that, I thought, OP could have allowed the user to enter Hiragana as well, since the conversion from Hiragana to Katakana can be done with a simple `StringTransform.hiraganaToKatakana`, but maybe this is an app meant to teach people how to type Katakana, who knows...