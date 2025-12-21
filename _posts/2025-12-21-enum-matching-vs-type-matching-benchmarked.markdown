---
layout: post
title:  "Enum Matching vs. Type Matching: Benchmarked"
date: 2025-12-21 22:40:00 +0300
categories: support-tips
tags: benchmark performance enums types matching switch case comparison
author: themixedupstuff
commentIssueId: TODO
excerpt: |
    In this post we explore the different approaches to process events from a
    mailbox with a more traditional enum based approach as well as a newer
    type matching method, with benchmarks included!
---
Recently, I have come to the realization that the event delivery system we
designed for the new PAL2 API for OpenTK was not exactly comfortable to use. The
API was designed with an older, more C like approach, where there is an enum
with a listing of all the possible event types, where you would switch on the
event type enum, and cast the event arguments as the correct type manually.

```cs
void OnEventRaised(PalHandle? handle, PlatformEventType type, EventArgs args)
{
    switch (type)
    {
        case PlatformEventType.MouseDown:
        {
            MouseButtonDownEventArgs down = (MouseButtonDownEventArgs)args;
            // Mouse handling code...
        }
        break;
        // ....
    }
}
```

Whilst this approach is pretty conventional, we have to recognize that we are
not writing pure C. The cleanest approach, in my opinion, would be to use a
tagged union. However, tagged unions are not available as a language feature in
C#. The second cleanest approach would be to use the newer type matching
features in C#.

```cs
void OnEventRaised(PalHandle? handle, PlatformEventType type, EventArgs args)
{
    switch (args)
    {
        case MouseButtonDownEventArgs mouseButtonDown:
            // Mouse handling code...
            break;
        // ....
    }
}
```

Although window events should ideally never be a bottleneck in regular
applications, this did raise a question of whichever approach is the fastest. I
find the results will intrigue you as much as it did me. Whilst the findings
aren't significant enough to affect our final decision, it might be important
for someone else in another field or in another application.

## The Premise
Imagine a scenario where you are receiving a significant number of events into
an event queue per second. Unfortunately for our poor program, it is very
important that these events be handled as quickly as possible with a minimum
amount of variance between the time it takes each event be processed. Each event
has something to identify its type, and must be processed by different code
paths.

For our premise, I will assume that each unique branch actually takes the same
amount of time to process. This keeps the benchmark code cleaner overall.

### Different Approaches

#### 1. Matching with an enum with a concrete member defined in the base type
If there happen to be any seasoned C/C++ developers reading this, this is likely
the approach you are most familiar with. Since these languages don't have any
way to identify types at runtime, most codebases which require this behavior
have created their own tagged union type using `union` and `struct`, or perhaps
using C++ inheritance with `class`.

```cs
enum MyEventType { Foo, Bar, Baz }

// Fancy primary constructor to keep the psuedocode short and concise.
class MyEvent(MyEventType type) : EventArgs
{
    public MyEventType Type { get; } = type;
}

// Later

switch (args.Type)
{
    case MyEventType.Foo: // ...
}
```

#### 2. Matching with an enum with a abstract/virtual member defined in the base type
A more object oriented programming approach might be to introduce a virtual
member instead of passing the value in a constructor. What if one wants to
delegate the exact type of the event to a leaf class in the inheritance
hierarchy without using a constructor every step of the way?

```cs
abstract class MyEvent : EventArgs
{
    public abstract MyEventType Type { get; }
}
```

#### 3. Type Matching
The final approach would be to use the newest language features with type
matching in switch case statements.

```cs
class FooEventArgs : EventArgs {}
class BarEventArgs : EventArgs {}
class BazEventArgs : EventArgs {}

// Later

switch (args)
{
    case FooEventArgs foo: // ...
}
```

### Methodology

In order to test the speed of the different approaches I wrote a BenchmarkDotnet
test case which I can run on my own computer as well as share to other people
for them to test. Publishing the findings on Discord and refining with multiple
people on the server we have come up with the following method:

* Create a base enum to use for the enum based approaches.
* Create a base type to derive all the unique event types.
    * Define one field with a concrete member.
    * Define another field with an abstract member of the same enum. This virtual
    member is created in order to factor in the cost of a virtual function
    call within a tight loop.
* Create a set of non-sealed unique classes which represent each event type.
* For each unique class create a new member which can be used to simulate a
workload. In our testing we used a float value to sum.

> **NOTE** <br>
> I have chosen to make the workload be a member of the child class to
> prevent the C# and JIT compiler from doing any sort of optimization feasibly.
> If it were a member of the parent class, the compiler could just chose to not
> do any type resolution at all.

* Create a `sealed` variant of each unique class to test against.
* For each test case:
    * Generate a large number of the events randomly, cache into
    a list. The generated events are of the sealed kind, so the same objects
    may be use for all tests.
    * The events are dispatched through delegate invocation in a loop to
    simulate a more realistic code base.
    * The event handler is expected to sum all of the numbers defined in the
    events.
    * The test case exits with the final result printed out to standard output.

With our test case, we have chosen to use 16 unique events with approximately
32 million instances. However, feel free to extend our example to more event
types and share the results with us on your own time.

## Testing and Results
> <pre>
> BenchmarkDotNet v0.15.8, Linux Fedora Linux 43 (Workstation Edition)
> AMD Ryzen 7 8700G w/ Radeon 780M Graphics 2.91GHz, 1 CPU, 16 logical and 8 physical cores
>  [Host]     : .NET 10.0.1 (10.0.1, 10.0.125.57005), X64 RyuJIT x86-64-v4
> </pre>
>
> | Method                   | Mean     | Error   | StdDev  | Median   | Ratio | RatioSD | Allocated | Alloc Ratio |
> |------------------------- |---------:|--------:|--------:|---------:|------:|--------:|----------:|------------:|
> | MatchOnEnum              | 258.5 ms | 1.86 ms | 1.74 ms | 258.1 ms |  1.03 |    0.03 |     168 B |        1.17 |
> | MatchOnEnumVirtual       | 457.9 ms | 1.76 ms | 1.56 ms | 458.1 ms |  1.82 |    0.05 |     144 B |        1.00 |
> | MatchOnType              | 654.4 ms | 3.94 ms | 3.68 ms | 653.7 ms |  2.60 |    0.07 |     144 B |        1.00 |
> | MatchOnEnumSealed        | 252.2 ms | 4.99 ms | 6.67 ms | 247.9 ms |  1.00 |    0.04 |     144 B |        1.00 |
> | MatchOnEnumVirtualSealed | 454.4 ms | 2.05 ms | 1.92 ms | 453.6 ms |  1.80 |    0.05 |     144 B |        1.00 |
> | MatchOnTypeSealed        | 266.0 ms | 1.24 ms | 1.16 ms | 265.4 ms |  1.06 |    0.03 |     168 B |        1.17 |
>
> <pre>
>   Mean        : Arithmetic mean of all measurements
>   Error       : Half of 99.9% confidence interval
>   StdDev      : Standard deviation of all measurements
>   Median      : Value separating the higher half of all measurements (50th percentile)
>   Ratio       : Mean of the ratio distribution ([Current]/[Baseline])
>   RatioSD     : Standard deviation of the ratio distribution ([Current]/[Baseline])
>   Allocated   : Allocated memory per single operation (managed only, inclusive, 1KB = 1024B)
>   Alloc Ratio : Allocated memory ratio distribution ([Current]/[Baseline])
>   1 ms        : 1 Millisecond (0.001 sec)
> </pre>

As you can see in the results, using `sealed` on the types makes a huge
difference in the performance. Doing so enables the additional optimizations
such as devirtualization and inlining. It is also important to recognize that
it is not always possible or beneficial to make types sealed, especially in a
toolkit like OpenTK. It might be valuable to have a platform specific event
structure which may have details specific to each driver.

Although the non-sealed type matching syntax is significantly slower, matching
on types is significantly more advantageous to the human writing the code. If you
can guarantee that the types will be sealed, there is no need to mess around
with writing an enum as it has very diminishing returns for applications without
significant performance considerations.

This is very easy to see when you consider the test cases where a virtual member
is used. The runtime has to either resolve the actual method to run, or resolve
the type and then execute the concrete method, if it even is concrete to begin
with. The key factor to keep in mind is how you can help the runtime eliminate
which code paths are not present in your code base, and an enum is an excellent
model for that. Although your code base may never run into certain cases, the
runtime does not see the code as you mentally model it. It only has a limited
view into your code to inspect and a list of optimizations it can do safely when
certain conditions are satisfied.

### Improvements made in .NET 10

I had initially done this test under .NET 8.0, which gave the following results.
You can clearly see the difference between the sealed and non-sealed types.
Interestingly, matching on enums with non-sealed types with
an enum is signigicantly slower than type matching on sealed types. The
performance has improved all over the board between .NET versions.

> | Method                   | Mean     | Error   | StdDev  | Ratio | Allocated | Alloc Ratio |
> |------------------------- |---------:|--------:|--------:|------:|----------:|------------:|
> | MatchOnEnum              | 290.6 ms | 0.62 ms | 0.49 ms |  1.22 |     168 B |        1.17 |
> | MatchOnEnumVirtual       | 483.9 ms | 3.78 ms | 3.54 ms |  2.03 |     144 B |        1.00 |
> | MatchOnType              | 677.3 ms | 1.84 ms | 1.72 ms |  2.85 |     144 B |        1.00 |
> | MatchOnEnumSealed        | 238.0 ms | 0.51 ms | 0.47 ms |  1.00 |     144 B |        1.00 |
> | MatchOnEnumVirtualSealed | 447.5 ms | 0.28 ms | 0.23 ms |  1.88 |     144 B |        1.00 |
> | MatchOnTypeSealed        | 273.5 ms | 1.09 ms | 1.02 ms |  1.15 |     168 B |        1.17 |

## Conclusions
Comparing the results, when it comes to absolute raw performance, the clear
winner is still the good old fashioned enum based matching method. When you
factor in `sealed` types, the type matching switch expression which has many
syntactic advantages and is easier to maintain overall becomes a very
competitive option. The biggest takeaway here is that you should avoid virtual
methods in performance critical code, which I believe is well known amongst the
performance concious developers in the field. Please note that benchmarks like
this should not be used to make architectural decisions unless it is absolutely
necessary to improve the performance of a bottleneck.

It is also important to highlight how little the performance of the event
matching matters in the use case of OpenTK. In the constraint of a 16ms frame
time budget (for 60fps), you can handle millions of events, meanwhile
realistically you will recieve less 20 to 30 events at most under most
circumstances. This experiement was more to satisfy my curiousity and i find
the results surprisingly close between some test cases.

I highly suggest reading the source code, as well as the assembly output of the
benchmark code in SharpLab.io for further insights into this test, the results
and more information. Feel free to try and modify the code at home as well as
discuss further improvements in the issue attached to this article.

## Further Reading
* [Benchmark Source Code](https://github.com/utkumaden/OpenTK.EventDeliveryTest)
* [Assembly At Sharplab.io](https://sharplab.io/#gist:b68760cb99a638c08d4d957497a28e1a)
* [C# Language Reference: `sealed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/sealed)
