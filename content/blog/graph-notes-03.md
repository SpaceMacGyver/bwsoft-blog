---
title: Refactoring in Rust
description: "Refactors and other tweaks to make the future an easier place"
slug: graph-notes-03
params:
  author: Justin Bell
categories:
  - Development
  - Showcase
tags:
  - rust
  - graph_notes
date: 2025-01-26T09:19:57-06:00
lastmod: 2025-01-26T09:19:57-06:00
draft: false
weight: 50
contributors: []
pinned: false
homepage: false
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

*This is the third post in a series documenting my progress on the [Graph Notes](/showcase/graph-notes) project.*

In my last article, I made some decisions I knew I'd come to regret.  Well, it's still a bit early in the project to
take on major cleanup (what with the fact that we just don't yet have that much code), but it's nevertheless a great
opportunity to lay out some best practices, clear organization, and perhaps give a bit of forethought into how we might
want this project to trend.

## Using the tools

First, I'll lead with an assumption that you, as a reader, are a brand-new developer who just woke up one day and decided
to write some Rust.  In reality, I'd expect most readers to already have some general understanding of code quality tools,
but for the sake of this discussion, I'll treat you as ignorant.  If you already know this stuff, feel free to skip around
as you see fit.

### cargo check

You get this one for free simply by having installed Rust (assuming you did so via the normal routes; if you have some
special way of installing it, you're on your own here).  This tool does the same compile-time checking that the `cargo build`
command does, but without a lot of the overhead associated with actually compiling (which, let's be honest, can sometimes
take a while with Rust codebases).  It's a great tool to spot-check yourself if you're not ready for a full build but want
to know whether you have glaring issues in your code; but, it also performs some rudimentary linting, which is why we're here
now.

With my code in its
[current state](https://github.com/bellwether-softworks/graph_note/commit/bb9669c098191f2e4bcf7891ef32e4c8f6a7ffd0),
running the `cargo check` command in my project folder reveals some issues:

```
warning: unused variable: `frame`
  --> src/app.rs:45:41
   |
45 |     fn update(&mut self, ctx: &Context, frame: &mut Frame) {
   |                                         ^^^^^ help: if this is intentional, prefix it with an underscore: `_frame`
   |
   = note: `#[warn(unused_variables)]` on by default

warning: variable does not need to be mutable
   --> src/app.rs:102:25
    |
102 |             if let Some(mut selected_note) = self.notes.get_mut(self.selected_note) {
    |                         ----^^^^^^^^^^^^^
    |                         |
    |                         help: remove this `mut`
    |
    = note: `#[warn(unused_mut)]` on by default

warning: `graph_note` (bin "graph_note") generated 2 warnings (run `cargo fix --bin "graph_note"` to apply 1 suggestion)
```

You might notice that adding the `--fix` argument would have automatically addressed one issue.  Myself, I prefer not to
use that flag, as going through and addressing these things by hand can function as a training tool.  Periodically, as the
language itself evolves and the tooling with it, new standards and suggestions are introduced through Rust's code quality
tools, and this can be a great way to keep in lockstep with the language's growth.

I'll apply the fixes as suggested, which gets me
[this commit](https://github.com/bellwether-softworks/graph_note/commit/4d4f83ac3bd496f4504eb8b4c48d79a74383ed9d).

### cargo clippy

If you don't already have this tool installed (it, unlike the prior one, doesn't come with Rust), you may need to
[install it](https://doc.rust-lang.org/clippy/installation.html) now.  It builds on what `cargo check` does by adding
even more compile-time checks; you could run it instead of `cargo check` if you are the pedantic sort who always
chases down compiler warnings, but I prefer to do so infrequently and treat its output as technical debt.  Whichever
route you prefer, though, this tool is an essential way by which the Rust team communicates "idiomatic Rust" to developers
in an ongoing fashion.

Because our project is still pretty green, and we've already addressed the `cargo check` recommendations, we have nothing
of note from this tool right now.  However, keep this one around; we'll come back to it later, as it will help us to identify
antipatterns and best practices we may otherwise be unaware of.

### IDE integration

Since I have no way of knowing what kind of IDE you're using, I won't get too specific here.  Suffice to say, I would
highly recommend having an IDE that gives you real-time feedback on code quality issues; Visual Studio Code with the
[rust-analyzer](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer) extension is a good route
to take, but I personally prefer to use [RustRover](https://www.jetbrains.com/rust/) or [Helix](https://helix-editor.com/)
for my Rust coding needs.  No matter which of those routes (or any other that has [LSP](https://en.wikipedia.org/wiki/Language_Server_Protocol) support)
you choose to take, you should see in real-time when you've botched something; consider this intentional goof, for instance:

{{< figure src="move-error.png" >}}

Note that I don't have to compile my code to know there's a problem.  To add to this, I could have my RustRover configured
to add automatic `cargo check` runs for every time my files are saved; and, because my editor is auto-saving frequently,
this means I get rapid feedback on pressing code-quality issues.

## Fearless Refactoring

This point isn't exclusive to Rust - other strongly-typed languages afford you the chance to do what I'm about to showcase.
That said, when taking into account how Rust handles its exhaustiveness checking or other forms of correctness, and when
backed by a good revision control strategy*, I should be able to make arbitrary changes to my code and let the resulting
errors dictate my next course of action.

{{< callout context="note" title="Be committed to committing!" >}}
You should never get to a place in your code where you're afraid to make a change.  One of the ways to ensure that you can
do this is to have a good strategy - commit often,
commit [atomic](https://www.reddit.com/r/learnprogramming/comments/w3mvl2/comment/igx1igf/).  I'll surely have a lot more
to say about this in the future, but it's crucial to have frequent and meaningful checkpoints to capture and describe our work.
{{</ callout >}}

So, for the sake of discussion, let's say I
[regret my earlier decision](http://localhost:1313/blog/graph-notes-02/#a-little-bit-more---more-notes-more-data)
to store timestamps as raw `SystemTime` values.  I even made myself a note reminding myself to fix this one day.  Well,
that day is today.

### Defining our intentions

First, I like what SystemTime offers us, but think it's an opinionated choice that doesn't lend itself well to long-term
interests.  There are a number of different ways we could go, including ISO8601-formatted dates, but I'll stick with
the simpler UNIX epoch-based time since 1970-01-01.  And, I doubt that we need more than second-based resolution for our
timestamps, so I'll make the decision to use a 64-based unsigned integer to store this - this can be changed someday if
necessary, but let's not entertain the idea of backdating our notes for the time being.

So, to sum up what I think I want: to have the storage of my timestamps be represented as a `u64`, but to somehow keep
all of our current `SystemTime`-based behavior.  The data model doesn't need to "know" how we'll handle the values, and
the business logic shouldn't need to "know" how we store our data.  Having this clear separation of concerns sets us up
for future changes - as said before, we might want to change our mind about this `u64` business down the road, and I want
to set us up for future success.

So, let's begin by first modeling our ideal date structure:

```rust
pub struct Timestamp(pub u64);

impl std::fmt::Display for Timestamp {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        let as_systemtime = OffsetDateTime::from_unix_timestamp(self.0 as i64)
            .expect("should be valid datetime");

        write!(f, "{}", as_systemtime.to_string())
    }
}

impl Timestamp {
    pub fn now() -> Self {
        let secs_since_epoch = SystemTime::now()
            .duration_since(SystemTime::UNIX_EPOCH)
            .expect("should produce valid time")
            .as_secs();

        Self(secs_since_epoch)
    }
}
```

We have two immediate concerns regarding timestamps - storage, and display.  The storage will be handled by wrapping our
seconds-since-epoch value in a named newtype called `Timestamp`.  The second we handle with the `std::fmt::Display` trait,
which gives us the `.to_string()` method and the `{}` format string glyph when using things like the `format!()` macro.
I've reproduced the original formatting logic into this code, giving us parity with our original behavior.

Additionally, I've handled the constructor (appropriately named "now" to indicate intent), which handles the conversion
from `SystemTime` to a numeric value.

With these things in place, we no longer need for our app to know about or understand `SystemTime` - let's start by replacing
that type declaration in our `Note` struct:

```diff
  struct Note {
-     created_on: SystemTime,
+     created_on: Timestamp,
      title: String,
      text: String,
  }
```

As a consequence of this change, we should now start to see some issues in our code.  For me, I have three red underlines
jump out at me in RustRover:

1. Inside our `Note` constructor, we instantiate the `created_on` field with `SystemTime::now()`
2. Our note listing code displays a `ui.label()` call with a call to `OffsetDateTime::from()` that now takes the wrong type as input
3. Likewise, our selected note's code displays an identical `ui.label()` call

I can fix #1 by replacing one type name with another, since I construct with an identical function name:

```diff
  impl Note {
      pub fn new() -> Self {
          Note {
-             created_on: SystemTime::now(),
+             created_on: Timestamp::now(),
              title: String::new(),
              text: String::new(),
          }
      }
  }
```

The `ui.label()` calls on #2 and #3 can be changed as follows...

```diff
  for (index, note) in self.notes.iter().enumerate() {
      ui.label(if index == self.selected_note { "*" } else { "" });
-     ui.label(OffsetDateTime::from(note.created_on).to_string());
+     ui.label(note.created_on.to_string());
      ui.label(note.title.as_str());
```

```diff
- ui.label(format!("Created at {:?}", OffsetDateTime::from(selected_note.created_on)));
+ ui.label(format!("Created at {}", selected_note.created_on));
```

Putting it all together, our red squigglies go away, and the code runs and performs identically to how it did previously.

## In summary

I'll just say that not all refactors work this cleanly.  I frequently refer to this practice of changing my code in vivo
as "pulling on a thread" - the allusion being to when you snag a thread in a knotted ball of yarn (or as happens to me,
a bunched up pile of extension cord) and you're forced to unknot it in all the places it gets hung.  In some cases,
I've spent _days_ on such refactors, because what inevitably happens is the "simple" change you make forces you to subsequently
change several other things downstream of your code.  I'm not ambitious enough to show that to you today, but I promise
it's extremely gratifying when you:

* Fix something
* Doing that breaks something else
* You find all the things it broke, and loop back up to the top
* Eventually, all the red squigglies are gone, and everything miraculously works

It sounds like a terrible workflow, but it's one that has been incredibly successful for me in the past.  And, you have
the added comfort of knowing that if you bungle things up badly enough, you've left yourself an escape hatch in the form
of your last good commit (you did commit as I suggested, right?).

See commit [6817fa2](https://github.com/bellwether-softworks/graph_note/commit/6817fa201fb5c5c72cf6d8b5ce57ed46b40350cf)
for the results of our refactor, and the preceding [4d4f83a](https://github.com/bellwether-softworks/graph_note/commit/4d4f83ac3bd496f4504eb8b4c48d79a74383ed9d)
for the things addressed in our `cargo check` efforts.
