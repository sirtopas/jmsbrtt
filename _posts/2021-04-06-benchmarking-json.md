---
layout: post
title:  "Benchmarking JSON Serializers in .NET"
categories: dev benchmarking .net json
---
## Intro
When it was [introduced](system-text-json), Microsoft set out some impressive performance stats for the new [System.Text.Json](json-announcement):


<table><thead><tr><th class="x-hidden-focus" align="left">Scenario</th><th align="left">Speed</th><th align="left">Memory</th></tr></thead><tbody><tr><td class="" align="left"><strong>Deserialization</strong></td><td align="left">2x faster</td><td align="left">Parity or lower</td></tr><tr><td align="left"><strong>Serialization</strong></td><td align="left">1.5x faster</td><td align="left">Parity or lower</td></tr><tr><td align="left"><strong>Document</strong>&nbsp;(read-only)</td><td align="left">3-5x faster</td><td align="left">~Allocation free for sizes &lt; 1 MB</td></tr><tr><td align="left"><strong>Reader</strong></td><td align="left">2-3x faster</td><td align="left">~Allocation free (until you materialize values)</td></tr><tr><td align="left"><strong>Writer</strong></td><td align="left">1.3-1.6x faster</td><td class="" align="left">~Allocation free</td></tr></tbody></table>

Having used [Newtonsoft.JSON](newtonsoft) for many years, I was intrigued and impressed by the stats for `System.Text.Json` - it showed a massive improvement in speed and throughput:

> For the most common payload sizes, System.Text.Json offers about 20% throughput increase in MVC during input and output formatting with a smaller memory footprint... The primary goal was performance and we see typical speedups of up to 2x over Json.NET

## Starting out
I'm fairly new to benchmarking; I've setup timers on code where performance is key but never in anger. Ideally, we should all be benchmarking where we can - it helps us understand how the code is being executed and where time is being wasted. If we benchmark more, we create a standard in our code and we can be sure that future changes we make won't affect the performance of our application.

Another reason I've not put benchmarking as high up as things like Unit Testing is the frameworks available to do it. When researching recently however, I found [BenchmarkDotNet](https://benchmarkdotnet.org/) - a simple and easy to use benchmarking framework. BenchmarkDotNet has excellent documentation and helps prevent benchmarking mistakes you might make. The output it produces from benchmarks is incredible:

- Markdown
- HTML
- CSV
- XML
- JSON
- Plots (with R)

[Steve Gordon](https://www.stevejgordon.co.uk/introduction-to-benchmarking-csharp-code-with-benchmark-dot-net) has a fantastic write-up on benchmarking in .NET using the framework and shows us why it's important and what we gain from doing it.

## Setup

Okay, enough overview let's see some code. 

I'm using .NET5 and Visual Studio for benchmarking so I'll create a new Console project to get us started:

![VS New Project](/jmsbrtt/assets/benchmark-new-project.png)

After we've been through the wizard, we're left with a nice shiny Console app ready for benchmarking. Next, we'll add some NuGet packages to help: **BenchmarkDotNet** and **Newtonsoft.Json**:

![VS NuGet](/jmsbrtt/assets/benchmark-nuget.png)

With those installed, we're ready to create a benchmark! 

## First benchmark

The first thing I'll do is create a benchmark class:

```csharp
using BenchmarkDotNet.Attributes;

namespace JsonBenchmark
{
    [MemoryDiagnoser]
    public class JsonBenchmarks
    {
        private const string _jsonString = "{\"username\" : \"user1\", \"fileId\" : 12, \"roles\" : [\"admin\", \"report\"]}";
        private static readonly JsonParser _parser = new JsonParser();

        [Benchmark(Baseline = true]]
        public void NewtonSoftJson()
        {
            _parser.NewtonSoftParseJson(_jsonString);
        }
    }
}
```

It's a simple class but let's go through some of the details:

- `[MemoryDiagnoser]`
    - MemoryDiagnoser is one of BenchmarkDotNet's [Diagnosers](https://benchmarkdotnet.org/articles/configs/diagnosers.html) that allows measuring the number of allocated bytes and garbage collection frequency.
- `_jsonString`
    - The test JSON string we'll be using to benchmark
- `JsonParser` 
    - This is the class that will be performing the deserialization of JSON from which the benchmarker can benchmark
    - More on this later
- `[Benchmark(Baseline = true]`
    - First, this sets the method as a `Benchmark` which allows BenchmarkDotNet to pick it up and run it when asked
    - Secondly, I've marked this as a [Baseline](https://benchmarkdotnet.org/articles/features/baselines.html). Baselines help us set a benchmark from which other benchmarks will be compared. As `Newtonsoft.Json` is the package I've used most, it'll be our baseline compared to `System.Text.Json` and will show how they compare to each other

As you can see, the class is small and simple. The only benchmark here is asking our JsonParser to serialize a JSON string into an object and measure performance. 

Next up is our `JsonParser` class:

```csharp
using System.Text.Json;

namespace JsonBenchmark
{
    public class JsonParser
    {
        public Employee NewtonSoftParseJson(string jsonString)
        {
            return Newtonsoft.Json.JsonConvert.DeserializeObject<Employee>(jsonString);
        }
    }
}
```

Again, this is another very simple class with a single method `NewtonSoftParseJson`. This method takes in the JSON string as a parameter and deserializes it using Newtonsoft.Json into our dummy object `Employee` which looks like this:

```csharp
using System.Collections.Generic;

namespace JsonBenchmark
{
    public class Employee
    {
        public int FileId { get;  set; }

        public string Username { get; set; }

        public IEnumerable<string> Roles { get; set; }
    }
}
```

So far, so good. We've created a benchmarking class which performs a single benchmark on `Newtonsoft.Json` which deserializes a string into a single `Employee` object. The only thing left to do it tie it all together in `Program.cs`:

```csharp
using BenchmarkDotNet.Running;

namespace JsonBenchmark
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var summary = BenchmarkRunner.Run<JsonBenchmarks>();
        }
    }
}
```

Inside `Main()` we've added a single line which calls the `Run` method on BenchmarkDotNet while passing in our Benchmark class. All that's left to do is run the project without the debugger and wait for our first results! The output of the benchmarks get generated to your `bin/Release` folder in the various formats; here is the markdown output:

``` ini

BenchmarkDotNet=v0.12.1, OS=macOS 11.2.3 (20D91) [Darwin 20.3.0]
Intel Core i7-8559U CPU 2.70GHz (Coffee Lake), 1 CPU, 8 logical and 4 physical cores
.NET Core SDK=5.0.201
  [Host]     : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT
  DefaultJob : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT


```

|         Method |     Mean |     Error |    StdDev | Ratio |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------- |---------:|----------:|----------:|------:|-------:|------:|------:|----------:|
| NewtonSoftJson | 1.671 μs | 0.0202 μs | 0.0157 μs |  1.00 | 0.7420 |     - |     - |   3.03 KB |

This looks a little daunting at first so let's go through some of these numbers:

- Mean      : Arithmetic mean of all measurements
- Error     : Half of 99.9% confidence interval
- StdDev    : Standard deviation of all measurements
- Ratio     : Mean of the ratio distribution ([Current]/[Baseline])
- Gen 0     : GC Generation 0 collects per 1000 operations
- Gen 1     : GC Generation 1 collects per 1000 operations
- Gen 2     : GC Generation 2 collects per 1000 operations
- Allocated : Allocated memory per single operation (managed only, inclusive, 1KB = 1024B)
**1 us      : 1 Microsecond (0.000001 sec)**

With that in mind, we can say that `Newtonsoft.Json` took ~1.6 microseconds to deserialize the object with 3KB of memory allocated. 

## Comparing

Let's try adding another benchmark for `System.Text.Json` and compare the results. We'll add a new Benchmark without the Baseline decorator:

```csharp
[Benchmark]
public void SystemTextJson()
{
    _parser.SystemTextJsonParseJson(_jsonString);
}
```

And a new method to deserialize with `System.Text.Json`:

```csharp
public Employee SystemTextJsonParseJson(string jsonString)
{
    return JsonSerializer.Deserialize<Employee>(jsonString);
}
```

Now we have two benchmarks, we can run the operation again and compare the results:

|         Method |       Mean |    Error |   StdDev |     Median | Ratio |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|--------------- |-----------:|---------:|---------:|-----------:|------:|-------:|------:|------:|----------:|
| SystemTextJson |   500.8 ns |  6.53 ns |  5.45 ns |   499.5 ns |  0.29 | 0.0095 |     - |     - |      40 B |
| NewtonSoftJson | 1,658.8 ns | 30.89 ns | 70.36 ns | 1,627.6 ns |  1.00 | 0.7420 |     - |     - |    3104 B |

So far Microsoft look to be telling the truth - not only did `System.Text.Json` take only 500ns compared to 1658ns of `Newtonsoft.Json`, but the allocated memory is 40B compared to the 3.1KB of `Newtonsoft.Json`. We can also now see the effect of using a Baseline benchmark in the Ratio column: `System.Text.Json` took less than 30% of the time to complete than `Newtonsoft.Json` did! 

## Bigger Files

So far we've found `System.Text.Json` to be much faster in our benchmarks but that was using a relatively small JSON file. Let's take it up a notch with more data to deserialize. Instead of passing in a string, I've created a ~200KB JSON file to store _multiple_ `Employee`s (around 2000 of them). I'll also have to change the benchmark we created to retrieve this data and pass it to the benchmarking methods:

```csharp
private string _jsonString = File.ReadAllText("employee-data.json");
```

**note** - always perform non-benchmarking tasks *outside* of your benchmarks. In this case, we don't care how well `File.ReadAllText` performs as it's outside of our benchmark. We want to perform this action in outside of any Benchmark and then pass the result as a string into those benchmarks as before. 

|         Method |     Mean |     Error |    StdDev | Ratio | RatioSD |    Gen 0 |   Gen 1 | Gen 2 | Allocated |
|--------------- |---------:|----------:|----------:|------:|--------:|---------:|--------:|------:|----------:|
| SystemTextJson | 1.092 ms | 0.0200 ms | 0.0273 ms |  0.35 |    0.02 |  25.3906 |       - |     - | 111.79 KB |
| NewtonSoftJson | 3.118 ms | 0.0581 ms | 0.1105 ms |  1.00 |    0.00 | 156.2500 | 50.7813 |     - | 753.66 KB |

Even with the larger file, `System.Text.Json` is still performing much faster with much less memory allocated. Similar to our single Employee file, it's taking almost 30% of the tie `Newtonsoft.Json` is taking.

## Notes

After our basic benchmarking, it looks like Microsoft are almost underselling the new JSON functionality - they stated it was 2x faster and at least on par with memory allocation. They're normally quite vocal about improvements and will often overstate them as opposed to giving conservative estimates... is there anything I've missed?

In the migration documents for [`Newtonsoft.Json` -> `System.TextJson`](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-migrate-from-newtonsoft-how-to?pivots=dotnet-5-0) there are some interesting pieces of information:

> During deserialization, Newtonsoft.Json does case-insensitive property name matching by default. The System.Text.Json default is case-sensitive, which gives better performance since it's doing an exact match.

Simply put: `Newtonsoft.Json` is case-insensitive by default whereas `System.Text.Json` is case-sensitive by default which gives better performance anyway... this sounds like a good case for benchmarking. 

We can force `System.Text.Json` to be case-insensitive very simply by creating some `JsonSerializerOptions` and passing them into the `JsonSerializer` in our `JsonParser.cs` class:

```csharp
var options = new JsonSerializerOptions() { PropertyNameCaseInsensitive = true };
```

We've now told `System.Text.Json` to be case-insensitive, better matching the default functionality of `Newtonsoft.Json`. Let's benchmark it!

|         Method |     Mean |     Error |    StdDev | Ratio | RatioSD |    Gen 0 |   Gen 1 | Gen 2 | Allocated |
|--------------- |---------:|----------:|----------:|------:|--------:|---------:|--------:|------:|----------:|
| SystemTextJson | 2.370 ms | 0.0339 ms | 0.0300 ms |  0.83 |    0.02 | 113.2813 | 46.8750 |     - | 510.52 KB |
| NewtonSoftJson | 2.836 ms | 0.0559 ms | 0.0854 ms |  1.00 |    0.00 | 160.1563 | 58.5938 |     - | 753.66 KB |

Some interesting output after the change. Our massive improvements are slipping and now show minor speed improvements whilst keeping some good memory allocation wins. 

## Conclusion

`System.TextJson` deserialization is fast. Faster than `Newtonsoft.Json` in my benchmarks in every way. I'd love to hear your views on the benchmarks and any suggestions for improvements. In the future benchmarking *massive* files might also be fun.

## Source
If you want to check out or clone the source code used in this article, check out the [repo](repo)!

[system-text-json]: https://devblogs.microsoft.com/dotnet/try-the-new-system-text-json-apis/
[json-announcement]: https://nuget.org/packages/System.Text.Json
[repo]: https://github.com/sirtopas/JsonBenchmarking/
