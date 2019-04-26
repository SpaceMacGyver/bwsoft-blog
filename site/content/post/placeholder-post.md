---
title: 'Using what Rust taught me in C#'
date: 2019-03-19T10:30:00.000Z
description: Part 1 of the the Lessons Learned from Rust series
---
## A [not-so-]brief introduction to Rust

Prior to Rust, JavaScript (more specifically, ES6 and everything that's come since) has been my go-to language for development. JS has a great blend of language features, opportunity for expression, and an ecosystem of supporting libraries that has actually been faulted for having an explosive rate of growth and breadth. One of its shortcomings, if you wish to consider it as such, would be that it's _too_ flexible at times, often addressed with intermediate languages or frameworks like TypeScript or Flow.

For those that need the behavior of JavaScript but can't bring themselves to use it directly, there are other languages that can be compiled into the web's native language. The previously mentioned TypeScript is one such language, but further down the line would be such things as [Scala](https://www.scala-js.org/), or for the braver of heart systems languages like C or C++ through [Emscripten](https://emscripten.org/).

Finally, if one is willing to make an almost clean cut with JavaScript and use WebAssembly, other options pop up, including [Rust](https://rustwasm.github.io/book/). As Rust eschews the use of managed memory in favor of compile-time guards on object lifetimes and cleanup, it's a good fit for the constraints of WebAssembly (which doesn't offer automatic garbage collection, instead preferring to provide a low-level interface to the web). Rust is not alone in this regard, but has a [strong community backing the effort](https://www.arewewebyet.org/) to break into the web development sphere and offer a safe programming experience.

This article isn't meant to sell you on doing web development in Rust; that's a topic for another day. Instead, I'd like to focus on the compile-time guards, which act not only as a fence to constrain "proper" developer behaviors, but also are a new Rust developer's best guide to idiomatic development.

You see, Rust's compiler is pedantic. Where pretty much every other compiled language allows you to write "bad" code, Rust won't let you move forward. Bad, in this context, is pretty much anything that leads to a segfault, race condition, or [poorly written branching logic](https://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch/), among other things.

It also babies you through errors. Where an error in other compilers would slap you on the wrist with an arcane error, Rust's compiler actually tries to tell you what you did wrong. It's not perfect, but it takes a fairly educated guess as to what you either meant to write, or what you're supposed to be writing. In this regard, the compiler serves as living documentation for how to use the language.

Rust's answer to safe development is complex, but not complicated.  Several design decisions were made to eliminate multiple classes of bugs at the root, and you can start using them today in C# with very little additional tooling.

## Just say no to null

`null` has been regarded [one of the more expensive mistakes](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare) in software development.  It is ambiguous in meaning, and because `null` coexists with actual data it needs to be accounted for when processing data.

Rust answers nullability by removing it entirely from the language.  Before you suffer a panic attack at the thought of losing the ability to represent the lack of data, though, know that the language (or rather, the standard libraries that accompany Rust) offers a superior alternative, the [`Option` type](https://doc.rust-lang.org/std/option/enum.Option.html).

The Option removes ambiguity from the equation.  Accessing its contents, should it in fact have any, is a deliberate decision done either with branching logic ([pattern matching](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring/destructure_enum.html) or [if let](https://doc.rust-lang.org/rust-by-example/flow_control/if_let.html)), or by explicitly unwrapping it when you, as the developer, know beyond all doubt that you will always have a value to work with.  Until you actually need to consume the value, though, it's easy to work with Options without having to guard against missing values, using the `.map()` statement:

<iframe height="400px" width="100%" src="https://repl.it/repls/VainTidyMedia?lite=true" scrolling="no" frameborder="no" allowtransparency="true" allowfullscreen="true" sandbox="allow-forms allow-pointer-lock allow-popups allow-same-origin allow-scripts allow-modals"></iframe>

In the above illustration, the same `.map()` statement is applied to options both with and without a value.  If you're in C# and working with value types, the nullable value type lets you do this as well:

<iframe height="400px" width="100%" src="https://repl.it/repls/IndolentFrontMonads?lite=true" scrolling="no" frameborder="no" allowtransparency="true" allowfullscreen="true" sandbox="allow-forms allow-pointer-lock allow-popups allow-same-origin allow-scripts allow-modals"></iframe>

It gets a bit more grim when dealing with null object references, though:

<iframe height="400px" width="100%" src="https://repl.it/repls/IndolentFrontMonads?lite=true" scrolling="no" frameborder="no" allowtransparency="true" allowfullscreen="true" sandbox="allow-forms allow-pointer-lock allow-popups allow-same-origin allow-scripts allow-modals"></iframe>



The good news for C# developers is that they can start using this tool today.  Multiple solutions exist, and the one that I would recommend is [language-ext](https://github.com/louthy/language-ext), for reasons that will become apparent in part two of this series.  That last example could be rewritten as follows if using the Option type from this library:

```c#
using System;
using LanguageExt;

namespace csharp_playground
{
    class Program
    {
        class Tom
        {
            public string Bob { get; set; }

            public static Tom AsUppercase(Tom original) => new Tom() {Bob = original.Bob.ToUpper()};
            public override string ToString() => $"Bob = {Bob}";
        }
        
        static void Main(string[] args)
        {
            Option<Tom> someTom = new Tom() {Bob = "cat"};
            Option<Tom> noTom = Option<Tom>.None;

            var foo = someTom.Map(Tom.AsUppercase);
            var bar = noTom.Map(Tom.AsUppercase);
            
            Console.WriteLine($"Foo: {foo}");
            Console.WriteLine($"Bar: {bar}");
        }
    }
}
```
