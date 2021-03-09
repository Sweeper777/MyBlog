---
tags: c# linq
op_link: https://stackoverflow.com/q/66432389/5133585
op_profile_link: https://stackoverflow.com/users/1251376/ssilas777
op_name: "ssilas777"
title: "LINQ Theta Join"
---

### Premise

OP tries to join these two tables (`BitTable` and `BitMask`):

| Id| Name           | BitId |
| --| -------------- |-------|
| 1 | TestData1      |   1   |
| 2 | TestData2      |   2   |
| 7 | TestData3      |   4   |

| DataId| DataMask| 
| --| -----------|
| 12 | 3|   
| 13 | 3|   
| 14 | 6|

into this:

| Id| Name           |BitId |DataId |DataMask|
| --| -------------- |-------|------|-------|-------|
| 2 | TestData2      |  2    |   14  |    6  |
| 7 | TestData3      |  4    |   14  |    6  |

In SQL, it would be:

{% highlight sql %}
SELECT * FROM BitTable bt
INNER JOIN BitMask bm ON bm.DataMask & bt.BitId = bt.BitId
WHERE bm.DataMask & 4 = 4
{% endhighlight %}

OP asks how this can be done in LINQ.

I have never done a join operation in LINQ before.

### Query Syntax Might Be Easier...

In OP's attempt, I saw that they are using the method syntax. I thought that isn't a very good idea for translating SQL. SQL is much more similar to the LINQ query syntax, so translating it to query syntax should be much easier. That is exactly what I did. Since I was too lazy to create the types that represent the tables, I just used `int` arrays. All the query cares about is `BitId` and `DataMask`, so two `int` arrays should be enough to "simulate" the query.

{% highlight c# %}
int[] bitTable = { 1, 2, 4 };
int[] bitMask = { 3, 3, 6 };
{% endhighlight %}

My first try in translating the query was:

{% highlight c# %}
var query = from bitId in bitTable
            join dataMask in bitMask on (dataMask & bitId) == bitId
            where (dataMask & 4) == 4
            select bitId;
{% endhighlight %}

### Not In Scope!?

I got the error that I need the contextual keyword `equals`, and that `dataMask` is not in scope on the left hand side of `equals`. Aha, so LINQ joins are always in the form of "something _equals_ something" and I can't put any arbitrary condition there, I thought

Luckily, the condition here _is_ in the form of "something equals something", I just need to rewrite it using the word `equals`, and put `dataMask` on the right hand side, because it's not in scope on the left for some reason. That's weird... I thought.

My second attempt:

{% highlight c# %}
join dataMask in bitMask on bitId equals (dataMask & bitId)
{% endhighlight %}

Now it's saying that `bitId` isn't in scope on the right hand side of `equals`! After seeing this, it finally clicked for me. It seems like a LINQ join must be in the form:

{% highlight c# %}
from x in table1
join y in table2 on f(x) equals g(y)
select ...
{% endhighlight %}

### The Join Overloads

I doubted this at first, as I thought LINQ was a really powerful tool, which should support joins on any condition, so I looked up the list of overloads of the [`Enumerable.Join`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.join?view=net-5.0) method, hoping to find more powerful overloads that are only available in method syntax.

To my surprise, it has only got 2 overloads:

{% highlight c# %}
public static IEnumerable<TResult> Join<TOuter,TInner,TKey,TResult> (
    this IEnumerable<TOuter> outer, 
    IEnumerable<TInner> inner, 
    Func<TOuter,TKey> outerKeySelector, 
    Func<TInner,TKey> innerKeySelector, 
    Func<TOuter,TInner,TResult> resultSelector);
{% endhighlight %}

and 

{% highlight c# %}
public static IEnumerable<TResult> Join<TOuter,TInner,TKey,TResult> (
    this IEnumerable<TOuter> outer, 
    IEnumerable<TInner> inner, 
    Func<TOuter,TKey> outerKeySelector, 
    Func<TInner,TKey> innerKeySelector, 
    Func<TOuter,TInner,TResult> resultSelector,
    IEqualityComparer<TKey>? comparer);
{% endhighlight %}

The only difference is an extra `IEqualityComparer`! This signature shows that join condition are really constrained to the form `f(x) equals g(y)`. `outerKeySelector` corresponds to `f`, and `innerKeySelector` corresponds to `g`.

Oh well, I thought, if it can't be done, it can't be done. Guess I'll resort to using the old-fashioned where-clauses to do joins...
