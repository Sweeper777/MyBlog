---
tags: c#
op_link: https://stackoverflow.com/q/69410717/5133585
op_profile_link: https://stackoverflow.com/users/58678/hippy
op_name: "hIpPy"
title: "The Performance of Case Insensitive HashSets"
---

### Premise

OP tried to measre the performance of `HashSet.Contains` using both a case sensitive hash set, and a case insensitive one.

{% highlight c# %}
[Test]
[Explicit]
public void Perf_HashSet()
{
    var hashSet = new HashSet<string>();
    for (int i = 0; i < size; i++)
    {
        hashSet.Add(Guid.NewGuid().ToString().ToLowerInvariant());
    }
    var x = Guid.NewGuid().ToString();
    var sw = Stopwatch.StartNew();
    for (int i = 0; i < iterations; i++)
    {
        var contains = hashSet.Contains(x.ToLowerInvariant());
    }
    sw.Stop();
    Console.WriteLine(sw.ElapsedMilliseconds);
}

[Test]
[Explicit]
public void Perf_HashSet_CaseInsensitive()
{
    var hashSetCaseInsensitive = new HashSet<string>(StringComparer.InvariantCultureIgnoreCase);
    for (int i = 0; i < size; i++)
    {
        hashSetCaseInsensitive.Add(Guid.NewGuid().ToString().ToLowerInvariant());
    }
    var x = Guid.NewGuid().ToString();
    var sw = Stopwatch.StartNew();
    for (int i = 0; i < iterations; i++)
    {
        var contains = hashSetCaseInsensitive.Contains(x);
    }
    sw.Stop();
    Console.WriteLine(sw.ElapsedMilliseconds);
}
{% endhighlight %}

`Perf_HashSet` took 43ms, but `Perf_HashSet_CaseInsensitive` turned out to be a lot worse - 903ms. OP wonders why. After all, it is only one extra step of checking whether a character is the uppercase/lowercase counterpart of another character. That shouldn't take that long.

### Why The Invariant Culture?

I inspected the code and see what OP used to implement a case insensitive hash set. Apparently, it is `StringComparer.InvariantCultureIgnoreCase`. On the other hand, the case sensitive version is just created with the parameterless constructor of `HashSet<T>`, which uses the default equality comparer of `string` if I recall correctly.

The mere sight of `StringComparer.InvariantCultureIgnoreCase` already rings alarm bells in my head. That uses the invariant culture's rules to compare the strings. I remembered that `string.Equals` is not culture sensitive at all, so is gotta be a lot faster. OP must have misunderstood "invariant culture" as "culture-insensitive", I thought. The first thing I did was to go to their respective documentations ([1](https://docs.microsoft.com/en-us/dotnet/api/system.stringcomparer.invariantcultureignorecase?view=net-5.0), [2](https://docs.microsoft.com/en-us/dotnet/api/system.string.equals?view=net-5.0#System_String_Equals_System_String_)) to check if I remembered correctly. And indeed, I did remember correctly.

### It's Hard To Write a Correct Benchmark

The second thing that stood out to me was the way OP wrote the benchmarks. They seemed to have simply used `Stopwatch` and NUnit. I don't know what the `[Explicit]` attribute does, but I don't expect it to help much with producing an accurate benchmark. I would assume that the CLR, like the JVM, has a "warmup period", where everything runs relatively slowly at the beginning. I'm not sure if OP's code has taken that into account. There are also some comments pointing out that the GC would need to clean up the strings created by `ToLower`, which might also affect the performance.

I'm not sure if the worse performance is solely caused by using a culture-sensitive comparison, or whether it is by the mistakes in OP's benchmarking code.

### My Own Benchmarks

Therefore, I have decided to write my own benchmarks using [BenchmarkDotNet](https://benchmarkdotnet.org). This is probably not a correct benchmark either, but I just wanted to do something similar to OP's code.

If I use `OrdinalIgnoreCase` instead of `InvariantCultureIgnoreCase`...

{% highlight c# %}
public class HashSetBenchmarks {
    private HashSet<string> caseSensitiveHashSet;
    private HashSet<string> caseInsensitiveHashSet;
    private string x;
    private const int size = 25;

    [Benchmark]
    public bool CaseSensitiveHashSet() {
        return caseSensitiveHashSet.Contains(x);
    }
    
    [Benchmark]
    public bool CaseInsensitiveHashSet() {
        return caseInsensitiveHashSet.Contains(x);
    }

    [GlobalSetup]
    public void SetUp() {
        caseSensitiveHashSet = new HashSet<string>();
        caseInsensitiveHashSet = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        for (int i = 0; i < size; i++)
        {
            caseSensitiveHashSet.Add(Guid.NewGuid().ToString().ToLowerInvariant());
            caseInsensitiveHashSet.Add(Guid.NewGuid().ToString().ToLowerInvariant());
        }
        x = Guid.NewGuid().ToString();
    }
}

class Program
{
    static void Main(string[] args)
    {
        BenchmarkRunner.Run<HashSetBenchmarks>();
    }
}
{% endhighlight %}

The results were:

    BenchmarkDotNet=v0.13.1, OS=macOS Big Sur 11.4 (20F71) [Darwin 20.5.0]
    Intel Core i5-8279U CPU 2.40GHz (Coffee Lake), 1 CPU, 8 logical and 4 physical cores
    .NET SDK=5.0.103
    [Host]     : .NET 5.0.3 (5.0.321.7212), X64 RyuJIT
    DefaultJob : .NET 5.0.3 (5.0.321.7212), X64 RyuJIT


|                 Method |     Mean |    Error |   StdDev |   Median |
|----------------------- |---------:|---------:|---------:|---------:|
|   CaseSensitiveHashSet | 20.95 ns | 0.451 ns | 0.715 ns | 20.56 ns |
| CaseInsensitiveHashSet | 19.76 ns | 0.407 ns | 0.340 ns | 19.64 ns |

If I use `InvariantCultureIgnoreCase`...

|                 Method |      Mean |     Error |    StdDev |
|----------------------- |----------:|----------:|----------:|
|   CaseSensitiveHashSet |  14.76 ns |  0.318 ns |  0.413 ns |
| CaseInsensitiveHashSet | 902.78 ns | 18.007 ns | 19.268 ns |

For some reason the Median column is gone from the output. This is probably something I can configure, but I don't really care. The output is very clear - the culture sensitive comparison is the sole reason for the slow performance.