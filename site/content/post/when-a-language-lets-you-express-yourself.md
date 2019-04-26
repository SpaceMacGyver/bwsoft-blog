---
title: When a language lets you express yourself
date: 2019-04-26T16:37:53.246Z
description: Part 2 of the the Lessons Learned from Rust series
---
As promised in the last post in [this series](https://www.bellwether-softworks.com/post/placeholder-post/), I will give a rationale for why [language-ext](https://github.com/louthy/language-ext) belongs in your toolkit.

## First, a bit of background

When LINQ first hit the scene, it was a big deal.  C# developers to this point were largely limited to imperative code when walking over enumerable sequences.  Think about how you might average a list of integers without LINQ:

```c#
int[] items = new int[] {1, 3, 5, 88, 623};
int accumulator = 0;

foreach (int i in items)
{
  accumulator += i;
}

int result = accumulator / items.Length;
```

With the LINQ enumerable extensions, we can skip the mutation and loop:

```c#
int[] items = new int[] {1, 3, 5, 88, 623};
int result = items.Sum() / items.Length;
```

To be fair, it's not that there's no mutation at all; under the hood, at some point, machine code is represented in terms of imperative instructions and mutable memory.  However, you as a developer should have the opportunity to step back and speak in terms of abstractions.  Rather than spelling out to the computer how it should do every little thing, modern languages give you the opportunity to express ideas.

Ideas can be shared quickly and conversationally.  They can be diagrammed easily and don't require a CS degree to understand.  An idea can outlive the lines of code that make it happen.

You, the programmer, are living in a world where you no longer need to micro-manage your code.

Okay, so the skeptics among you might be thinking:

> That's great and all.  I see what you did with that array, but not everything's an enumerable object.

## Options to the rescue

I introduced the `Option` in the first post in the earlier post.  At the time, I sold it as the solution to the problem of nullability.  This by itself is actually a weak case to be made for language-ext, with [C# 8 bringing nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/nullable-reference-types) to the masses.

However, that's not its only benefit.  The language-ext types, including Option, can help you get the same kind of declarative code for other types too.

First, let's consider a scenario where we're trying to split a `key=value` string.  The key we care about is "name", and in the event we have a valid (non-empty) value for the name we return it; if not, we return "no name".

Here's how I might write this in C#:

```c#
static string oldFashionedWay(string pair)
{
    if (!string.IsNullOrEmpty(pair))
    {
        var parts = pair.Split('=');

        if (parts.Length == 2 && parts[0] == "name" && !string.IsNullOrEmpty(parts[1]))
        {
            return parts[1];
        }
    }

    return "no name";
}
```

The important parts here are:

* We only return a name if we meet all of the following conditions:
  - Input is not null
  - Input can be split into two parts using `=`
  - The first split part must equal "name"
  - The second split part must not be empty
* Otherwise, return "no name"

Because there are multiple ways we could get into a "no name" condition, it makes sense to have that be the fallthrough case.  One possible improvement we could make to my illustration would be to get around the null/empty check with a [safe navigation operator](https://en.wikipedia.org/wiki/Safe_navigation_operator#C%23):

```c#
static string oldFashionedWay(string pair)
{
    var parts = pair?.Split('=');

    if (parts?.Length == 2 && parts?[0] == "name" && !string.IsNullOrEmpty(parts?[1]))
    {
        return parts[1];
    }

    return "no name";
}
```

Personally, I'm okay with this code.  However, with Options, we could maybe offer a solution that is both clever and readable (two things that don't normally go hand-in-hand in programming):

```c#
static string getName(Option<string> maybePair) => maybePair
    .Map(s => s.Split('='))
    .Find(pair => pair.Length == 2)
    .Find(pair => pair[0] == "name" && pair[1].Length > 0)
    .Map(pair => pair[1])
    .IfNone("no name");
```

The major downside is that we had to have our input wrapped in an `Option<string>` beforehand, but this is relatively simple if we've imported the `LanguageExt.Prelude` namespace:

```c#
var validName = Some("name=bob");
var invalidName = Some("name=");
var notAPair = Some("name");
var notAName = Some("other=value");
var noValue = None;
```

Besides that, you might be thinking that the Option-based version looks tedious.  It's more dense, but I would argue that the workflow is clearer when piped this way.  I see this as a point that will probably be hotly contested, but before you give up on it, there are some extractions that I think will make things a bit clearer:

```c#
static string getName(Option<string> maybePair) => maybePair
    .Map(SplitPair)
    .Find(HasLength(2))
    .Find(HasValidName)
    .Map(pair => pair[1])
    .IfNone("no name");

static string[] SplitPair(string input) => input.Split('=');
static Func<string[], bool> HasLength(int count) => pair => pair.Length == count;
static bool HasValidName(string[] pair) => pair[0] == "name" && pair[1].Length > 0;
```

At the expense of having to write more code, the `getName` function now reads as a chain of terse and self-explanatory steps.  The logic that backs those extracted functions have the added benefit of being reusable for other situations, in addition to being named in a way that is self-documenting.

The full source for my example can be found [here](https://gitlab.com/snippets/1851612).
