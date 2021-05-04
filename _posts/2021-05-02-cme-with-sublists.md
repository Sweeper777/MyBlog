---
tags: java
op_link: https://stackoverflow.com/q/67352262/5133585
op_profile_link: https://stackoverflow.com/users/13830331/gian
op_name: "gian_"
title: "ConcurrentModificationException when using sublists"
---

### Premise

OP has written a merge sort, but it throws CME here:

{% highlight java %}
void mergeSort() { 
    mergeSort(0, list.size() - 1);
}

private void mergeSort(int p, int r) {
    if (p < r) {
        int q = (p + r) / 2;
        mergeSort(p, q);
        mergeSort(q + 1, r);
        merge(p, q, r);
    }
}

private void merge(int p, int q, int r) {
    List<Product> left = list.subList(p, q);
    left.add(Product.PLUS_INFINITE);
    List<Product> right = list.subList(q + 1, r);
    right.add(Product.PLUS_INFINITE);
    int i = 0;
    int j = 0;
    for (int k = p; k <= r; ++p) {
        // CME here VVV
        Product x = left.get(i).compareTo(right.get(j)) <= 0 ? left.get(i++) : right.get(j++);
        list.set(k, x);
    }
}
{% endhighlight %}

### No Iterators Huh?

The most common CME questions are asked by people who get CMEs because they are iterator through a list, and removing things from the list:

{% highlight java %}
for (var thing: list) {
    list.remove(0);
}
{% endhighlight %}

But this one has no iterators at all! It must be something else then.

### Semantics of Sublists

The first thing I saw was `subList`, I recalled that `subList` produces a "view" of the original list, rather than a copy, so that might be causing the concurrent modification. I did a thought experiment: if I have two "views", and I modify one of them, the other one wouldn't know. When I try to do another modification on the other one, it notices that the list has been modified without it knowing, and goes crazy (throws CME). Then I thought, iterators are also "views" on lists, so in this regard, sublists are kind of like iterators, aren't they?

My thought experiment seems like something that could very well happen. Now I just need to find the bit of documentation that specifies it. This is quite easy. The [documentation](https://docs.oracle.com/javase/8/docs/api/java/util/List.html#subList-int-int-) says:

> The semantics of the list returned by this method become undefined if the backing list (i.e., this list) is structurally modified in any way other than via the returned list. 

So it doesn't explicitly say "CME", but "undefined" is close enough, I guess. If it is undefined, then CME is definitely one of those things that could happen.

### Reproducing The CME

I tried to make a minimal code snippet to reproduce the exception. I started with OP's code, but with all the non-list operations removed (also changed the list to a list of strings):

{% highlight java %}
ArrayList<String> list = new ArrayList<>(List.of("1", "2", "3", "4"));
List<String> left = list.subList(0, 2);
left.add("");
List<String> right = list.subList(2, 4);
right.add("");
left.get(0);
right.get(0);
list.set(0, "");
{% endhighlight %}

Ran the code, found that it occurs on `left.get(0)`, so I deleted everything after that. Then I reasoned that since `left.add` is being done on the returnd sublist, it does not make the semantics of `left` undefined, so I removed that call too. Well then it must be `right.add` that makes the semantics of `left` undefined!

{% highlight java %}
ArrayList<String> list = new ArrayList<>(List.of("1", "2", "3", "4"));
List<String> left = list.subList(0, 2);
List<String> right = list.subList(2, 4);
right.add("");
left.get(0);
{% endhighlight %}

Having figured this out, I was ready to write an answer.

### There's Still An Error!?

After posting the answer however, OP said that my solution (make copies of the sublist) didn't work - the stack trace is the same. I thought that's not possible. There couldn't possibly still be a CME anymore. There must be something wrong with the OP's merge sort algorithm. I wanted OP to accept my answer, so I tried to reproduce the entire merge sort algorithm in my IDE. I had to change it to sort a list of integers, rather than a list of `Product`s.

{% highlight java %}
public class Main {
  List<Integer> list = new ArrayList<>(List.of(9,7,8,6,1,2,5,3,4));

  public static void main(String[] args) throws IOException {
    Main m = new Main();
    m.mergeSort();
    System.out.println(m.list);
  }

  // the two mergeSort methods are the same...

  private void merge(int p, int q, int r) {
    List<Integer> left = new ArrayList<>(list.subList(p, q));
    left.add(Integer.MAX_VALUE);
    List<Integer> right = new ArrayList<>(list.subList(q + 1, r));
    right.add(Integer.MAX_VALUE);
    int i = 0;
    int j = 0;
    for (int k = p; k <= r; ++k) {
      Integer x = left.get(i).compareTo(right.get(j)) <= 0 ? left.get(i++) : right.get(j++); //147
      list.set(k, x);
    }
  }

}
{% endhighlight %}

After making copies of the sublists, I saw that is an `ArrayIndexOutOfBoundsException` that is thrown at the same line. Clearly, something is wrong with OP's merge sort.

Upon inspecting OP's algorithm, I found that it is a bit different from the merge sort I know. In `merge`, it avoids copying the "tails" of either list by putting a "max element" at the end of each list. I was not sure if that actually works. I spent quite some time figuring out when the for loop should stop, as I thought *that* was the problem. 

After loads and loads of debugging, I noticed that the sublist indices are a bit weird. To split a list into two portions, the end index of the first sublist should be the same as the start index of the second sublist (since the end index is exclusive and the start index is inclusive), but that wasn't the case in OP's code. I thought the OP might not be aware of the fact that the second argument of `sublist` is exclusive. So I tried to add 1 to all the end indices:

{% highlight java %}
List<Integer> left = new ArrayList<>(list.subList(p, q + 1));
List<Integer> right = new ArrayList<>(list.subList(q + 1, r + 1));
{% endhighlight %}

Then it works!