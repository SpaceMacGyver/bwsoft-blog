---
title: My First Notes
description: Fleshing out the Graph Notes application with actual (basic) notes
slug: graph-notes-02
date: 2024-08-25T09:10:00-05:00
lastmod: 2024-08-25T09:10:00-05:00
params:
  author: Justin Bell
draft: false
weight: 50
categories:
  - Development
  - Showcase
tags:
  - rust
  - graph_notes
contributors: []
pinned: false
homepage: false
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

*This is the second post in a series documenting my progress on the [Graph Notes](/showcase/graph-notes) project.*

I've learned over the course of my career that, when dreaming big, you have to think small.  By this, what I mean is that
making something ambitious, like the world's most powerful note-taking app, is accomplished by taking iterative steps.
Today's baby step will be the introduction of some rudimentary notes.

## Just a single note

That's it.  We can hang our hat up after tracking a simple blurb of text.

No, seriously, we really are starting that small.  I think you'll find, though, as we accomplish this seemingly basic
feat, that we're forced to do some big things along the way.

For starters, let's revisit our interface.  We left off with a skeletal app in the [last article](/blog/graph-notes-01),
and we'll be building on that foundation in today's work.  Let's hunt down **src/app.rs** and make some tweaks to
capture some text.  In particular, I'll revisit the `update` function as follows:

```rust
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            let mut text = String::new();
            ui.label("Note text");
            
            egui::TextEdit::multiline(&mut text).show(ui);
        });
    }
```

{{< figure src="adding-text-edit.png" >}}

Kinda ugly, if you ask me; also, if you try to type into that text area, you should quickly realize that it doesn't actually
let you type into it (or, rather, the text vanishes almost as soon as you enter it) - this is because our state doesn't have
a good place to live, and is recreated with every frame that's rendered.  No es bueno.

We'll deal with aesthetics later; for now, it would be better if our app actually did the thing it is supposed to.  For now,
thinking about where we're at (inside the `App` instance), and the fact that we're dealing with a `&mut self` in our `update`
function, it stands to reason that gives us the best place to put data.  In fact, if you might recall, our `App` doesn't currently
hold any data:

```rust
pub struct App;
```

That seems like a good therapeutic target.  Let's add some data to this structure to let us represent our note:

```rust
pub struct App {
    note: String,
}
```

... and alter our `update` function to use it instead of our ephemeral state:

```rust
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.label("Note text");
            
            egui::TextEdit::multiline(&mut self.note).show(ui);
        });
    }
```

If you try to run this now, you'll also find an angry compiler that wants you to revisit the `default` implementation:

```rust
impl Default for App {
    fn default() -> Self {
        Self {
            note: String::new()
        }
    }
}
```

With that, you should now be able to enter text into the form field, and see it stick.  You can find the commit for
the above changes [here](https://github.com/bellwether-softworks/graph_note/commit/4893a82116ec08073e1f62be50abb3854ecfc44d)
to see before vs. after.

## A little bit more - more notes, more data

That note's pretty useless by itself.  Maybe there's a use-case for a program that lets you just type some stuff into a box,
but our notes are fancier than that.  At a minimum, we'll want to know when the note was created, a title to go along with it,
and I'm guessing we'll want more than one single note in our lonely application.

First, let's build out a separate struct to handle our note's structure, along with a constructor:

```rust
use std::time::SystemTime;

// ...

struct Note {
    created_on: SystemTime,
    title: String,
    text: String,
}

impl Note {
    pub fn new() -> Self {
        Note {
            created_on: SystemTime::now(),
            title: String::new(),
            text: String::new(),
        }
    }
}
```

...tweak our App to use it instead of the standalone String:

```rust
pub struct App {
    note: Note,
}
```

...and update our App's `default` to instantiate it:

```rust
impl Default for App {
    fn default() -> Self {
        Self {
            note: Note::new(),
        }
    }
}
```

Don't forget to visit the `TextEdit` instance to point to our nested `.text` property as well:

```rust
egui::TextEdit::multiline(&mut self.note.text).show(ui);
```

Putting everything together, we should be able to launch the app and see... nothing different.  On the backend, we're now
tracking (but not using) a note creation date and an always-empty title field.  Let's get those things onto the screen to
work with:

```rust
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Title");
                ui.text_edit_singleline(&mut self.note.title);
            });

            ui.label(format!("Created at {:?}", self.note.created_on));

            ui.label("Text");
            egui::TextEdit::multiline(&mut self.note.text).show(ui);
        });
    }
```

{{< figure src="now-with-title-and-time.png" >}}

Aside from the most unreadable time format ever, we're closer.  Let's solve that formatting issue by bringing in
the [time](https://crates.io/crates/time) library:

```bash
$ cargo add time
```

...and modify our format string to use the `OffsetDateTime` struct
[instead of](https://rcos.io/static/internal_docs/time/struct.OffsetDateTime.html#impl-From%3CSystemTime%3E) our `SystemTime` instance:

```rust
ui.label(format!("Created at {:?}", OffsetDateTime::from(self.note.created_on)));
```

{{< figure src="formatted-time.png" >}}

It's a vast (albeit imperfect) improvement from before.  There are more formatting opportunities in this `time` library,
and we can target those at a future time (pun intended), but these steps will lay some groundwork for those changes.

{{< callout context="note" title="Memo to self" icon="outline/bulb" >}}
Storing as a `SystemTime` and continuously casting it to `OffsetDateTime` seems wasteful; we might make
this a target for future refactors...
{{< /callout >}}

### Laying the groundwork for CRUD

Now that we have a rough shape for our data, let's see about having a list of these guys.  In a nutshell, we need to
permit for a list of notes, along with the necessary CRUD operations and user interface to pull it off.  To start, let's
convert our single note instance into a collection, and automatically spawn the first one on app launch - this will give
us parity with our current workflow and allow us to iterate:

{{< tabs "moving to a list" >}}

{{< tab "diff" >}}
```diff
pub struct App {
-    note: Note,
+    notes: Vec<Note>,
}

impl Default for App {
    fn default() -> Self {
        Self {
-           note: Note::new(),
+           notes: vec![Note::new()],
        }
    }
}

impl eframe::App for App {
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Title");
-               ui.text_edit_singleline(&mut self.note.title);
+               ui.text_edit_singleline(&mut self.notes[0].title);
            });

-           ui.label(format!("Created at {:?}", OffsetDateTime::from(self.note.created_on)));
+           ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[0].created_on)));

            ui.label("Text");
-           egui::TextEdit::multiline(&mut self.note.text).show(ui);
+           egui::TextEdit::multiline(&mut self.notes[0].text).show(ui);
        });
    }
}
```
{{< /tab >}}

{{< tab "rust" >}}
```rust
pub struct App {
    notes: Vec<Note>,
}

impl Default for App {
    fn default() -> Self {
        Self {
            notes: vec![Note::new()],
        }
    }
}

impl eframe::App for App {
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Title");
                ui.text_edit_singleline(&mut self.notes[0].title);
            });

            ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[0].created_on)));

            ui.label("Text");
            egui::TextEdit::multiline(&mut self.notes[0].text).show(ui);
        });
    }
}
```
{{< /tab >}}

{{< /tabs >}}

Re-running the app at this point, everything should continue to look and behave as it did previously.  We do have a baked-in
assumption when displaying the note, though, that we are guaranteed to work with a note at index `[0]`, which is a problem that
we can remedy by tracking which note is being displayed at a given time.  For now, we'll track this selection by index:

{{< tabs "list selection" >}}

{{< tab "diff" >}}
```diff
pub struct App {
+   selected_note: usize,
    notes: Vec<Note>,
}

impl Default for App {
    fn default() -> Self {
        Self {
            notes: vec![Note::new()],
+           selected_note: 0,
        }
    }
}

impl eframe::App for App {
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Title");
-               ui.text_edit_singleline(&mut self.notes[0].title);
+               ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
            });

-           ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[0].created_on)));
+           ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[self.selected_note].created_on)));

            ui.label("Text");
-           egui::TextEdit::multiline(&mut self.notes[0].text).show(ui);
+           egui::TextEdit::multiline(&mut self.notes[self.selected_note].text).show(ui);
        });
    }
}
```
{{< /tab >}}

{{< tab "rust" >}}
```rust
pub struct App {
    selected_note: usize,
    notes: Vec<Note>,
}

impl Default for App {
    fn default() -> Self {
        Self {
            notes: vec![Note::new()],
            selected_note: 0,
        }
    }
}

impl eframe::App for App {
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Title");
                ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
            });

            ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[self.selected_note].created_on)));

            ui.label("Text");
            egui::TextEdit::multiline(&mut self.notes[self.selected_note].text).show(ui);
        });
    }
}
```
{{< /tab >}}

{{< /tabs >}}

As with before, we should still have parity with our prior run.

### Adding and listing notes

Let's focus next on the UI for adding new notes.  It's going to start off rough, but I promise in the future that we'll
tidy up the UI to make it easier on the eyes.  For today, the things I think we need to execute this idea include:

1. A button that allows us to add a new note
2. A list of existing notes displaying date and title
3. Selection capability - picking one of our listed notes should make it the currently displayed note

Let's tackle these items in the listed order.  First, I'll just plop the button down above our first `ui.horizontal()` call:

```rust
fn update(&mut self, ctx: &Context, frame: &mut Frame) {
    egui::CentralPanel::default().show(ctx, |ui| {
        if ui.button("Add new note").clicked() {
            self.notes.push(Note::new());
            self.selected_note = self.notes.len() - 1;
        }
        
        ui.horizontal(|ui| {
            ui.label("Title");
            ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
        });

        // ...
    });
}
```

Were you to run in our current state, you'd see that clicking the button updates the displayed timestamp and clears
out our inputs - or, rather, we're displaying the newly-created note, but seemingly lose access to our prior notes in the
process.  It would be nice to know we're not actually getting rid of our previous notes, so let's tackle that list (and
while we're in there, the selection ability):

{{< tabs "see and edit prior notes" >}}
{{< tab "diff" >}}
```diff
- use egui::Context;
+ use egui::{Context, Grid};

fn update(&mut self, ctx: &Context, frame: &mut Frame) {
    egui::CentralPanel::default().show(ctx, |ui| {
        if ui.button("Add new note").clicked() {
            self.notes.push(Note::new());
            self.selected_note = self.notes.len() - 1;
        }
        
+       Grid::new("Notes")
+           .show(ui, |ui| {
+               // Header row
+               ui.label("");
+               ui.label("Date");
+               ui.label("Title");
+               ui.end_row();
+
+               for (index, note) in self.notes.iter().enumerate() {
+                   ui.label(if index == self.selected_note { "*" } else { "" });
+                   ui.label(OffsetDateTime::from(note.created_on).to_string());
+                   ui.label(note.title.as_str());
+                   if ui.button("Edit").clicked() {
+                       self.selected_note = index;
+                   }
+                   ui.end_row();
+               }
+           });
+       ui.separator();
        
        ui.horizontal(|ui| {
            ui.label("Title");
            ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
        });

        // ...
    });
}
```
{{< /tab >}}

{{< tab "rust" >}}
```rust
use egui::{Context, Grid};

fn update(&mut self, ctx: &Context, frame: &mut Frame) {
    egui::CentralPanel::default().show(ctx, |ui| {
        if ui.button("Add new note").clicked() {
            self.notes.push(Note::new());
            self.selected_note = self.notes.len() - 1;
        }
        
        Grid::new("Notes")
            .show(ui, |ui| {
                // Header row
                ui.label("");
                ui.label("Date");
                ui.label("Title");
                ui.end_row();
 
                for (index, note) in self.notes.iter().enumerate() {
                    ui.label(if index == self.selected_note { "*" } else { "" });
                    ui.label(OffsetDateTime::from(note.created_on).to_string());
                    ui.label(note.title.as_str());
                    if ui.button("Edit").clicked() {
                        self.selected_note = index;
                    }
                    ui.end_row();
                }
            });
        ui.separator();
        
        ui.horizontal(|ui| {
            ui.label("Title");
            ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
        });

        // ...
    });
}
```
{{< /tab >}}

{{< /tabs >}}

I threw in a `ui.separator()` call to give us some ability to distinguish between the elements of our app; without it,
things get a bit cramped and confusing.  Not that it's all that great now, but hopefully it makes some sense to look at:

{{< figure src="editable-list.png" >}}

There's much room for improvement.  Note the conditional `*` field to show us our current selection.  The crazy date format.
Why do we display an "Edit" button on the same row as we have currently selected?  Still, this gives us a starting point.

### Removal of notes

Well, let's wrap up our session today with the idea of note deletion.  It introduces some concerns that we didn't have before:

* If we're on a selection, and the list above us shifts, we need to adjust our selection to stay with the current note
* If we're on a selection and that index is removed, where do we end up?  For now, let's try to stay on the current index...
* ...but if that selection was the last index, we should decrement to the last available index
* And, if we delete *all* of our notes, we shouldn't display the note form at all

These rules are arbitrary, but give us some guard rails to keep the app from panicking when we inevitably point to a note
that no longer exists.

{{< callout context="caution" title="Bad practice in action" icon="outline/biohazard" >}}
The "Remove" button will live alongside the "Edit" button for corresponding records.  In it, we'll have to accommodate
many of our above rules, which will make for a lot of business logic - this much commingled UI and logic will become
a pain point in the future, and dealing with this will be a topic for another day.
{{< /callout >}}

{{< tabs "deleting notes" >}}

{{< tab "diff" >}}
```diff
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            if ui.button("Add new note").clicked() {
                self.notes.push(Note::new());
                self.selected_note = self.notes.len() - 1;
            }

+           let mut to_remove = None;
+           let mut to_select = None;

            Grid::new("Notes")
                .show(ui, |ui| {
                    // Header row
                    ui.label("");
                    ui.label("Date");
                    ui.label("Title");
                    ui.end_row();

                    for (index, note) in self.notes.iter().enumerate() {
                        ui.label(if index == self.selected_note { "*" } else { "" });
                        ui.label(OffsetDateTime::from(note.created_on).to_string());
                        ui.label(note.title.as_str());
                        if ui.button("Edit").clicked() {
                            to_select = Some(index);
                        }
+                       if ui.button("Delete").clicked() {
+                           if self.notes.len() == 1 {
+                               // This is the last note; at this point, our selection index is meaningless,
+                               // but can't dip below 0, so we'll just ignore it for now
+                               to_remove = Some(index);
+                           } else if index == self.selected_note && index == self.notes.len() - 1 {
+                               // This is the note at the end of the list; let's decrement and display
+                               // the last available note
+                               to_remove = Some(index);
+                               to_select = Some(self.selected_note - 1);
+                           } else if index < self.selected_note {
+                               // Removing a note before our current selection, so we need to move
+                               // with it accordingly
+                               to_remove = Some(index);
+                               to_select = Some(self.selected_note - 1);
+                           } else {
+                               to_remove = Some(index);
+                           }
                        }
                        ui.end_row();
                    }
                });
+
+           if let Some(to_select) = to_select {
+               self.selected_note = to_select;
+           }
+           if let Some(to_remove) = to_remove {
+               self.notes.remove(to_remove);
+           }
+
            ui.separator();

-           ui.horizontal(|ui| {
-               ui.label("Title");
-               ui.text_edit_singleline(&mut self.notes[self.selected_note].title);
-           });
-
-           ui.label(format!("Created at {:?}", OffsetDateTime::from(self.notes[self.selected_note].created_on)));
-
-           ui.label("Text");
-           egui::TextEdit::multiline(&mut self.notes[self.selected_note].text).show(ui);
+           if let Some(mut selected_note) = self.notes.get_mut(self.selected_note) {
+               ui.horizontal(|ui| {
+                   ui.label("Title");
+                   ui.text_edit_singleline(&mut selected_note.title);
+               });
+
+               ui.label(format!("Created at {:?}", OffsetDateTime::from(selected_note.created_on)));
+
+               ui.label("Text");
+               egui::TextEdit::multiline(&mut selected_note.text).show(ui);
+           }
        });
    }
```
{{< /tab >}}

{{< tab "rust" >}}
```rust
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            if ui.button("Add new note").clicked() {
                self.notes.push(Note::new());
                self.selected_note = self.notes.len() - 1;
            }

            let mut to_remove = None;
            let mut to_select = None;

            Grid::new("Notes")
                .show(ui, |ui| {
                    // Header row
                    ui.label("");
                    ui.label("Date");
                    ui.label("Title");
                    ui.end_row();

                    for (index, note) in self.notes.iter().enumerate() {
                        ui.label(if index == self.selected_note { "*" } else { "" });
                        ui.label(OffsetDateTime::from(note.created_on).to_string());
                        ui.label(note.title.as_str());
                        if ui.button("Edit").clicked() {
                            to_select = Some(index);
                        }
                        if ui.button("Delete").clicked() {
                            if self.notes.len() == 1 {
                                // This is the last note; at this point, our selection index is meaningless,
                                // but can't dip below 0, so we'll just ignore it for now
                                to_remove = Some(index);
                            } else if index == self.selected_note && index == self.notes.len() - 1 {
                                // This is the note at the end of the list; let's decrement and display
                                // the last available note
                                to_remove = Some(index);
                                to_select = Some(self.selected_note - 1);
                            } else if index < self.selected_note {
                                // Removing a note before our current selection, so we need to move
                                // with it accordingly
                                to_remove = Some(index);
                                to_select = Some(self.selected_note - 1);
                            } else {
                                to_remove = Some(index);
                            }
                        }
                        ui.end_row();
                    }
                });

            if let Some(to_select) = to_select {
                self.selected_note = to_select;
            }
            if let Some(to_remove) = to_remove {
                self.notes.remove(to_remove);
            }

            ui.separator();

            if let Some(mut selected_note) = self.notes.get_mut(self.selected_note) {
                ui.horizontal(|ui| {
                    ui.label("Title");
                    ui.text_edit_singleline(&mut selected_note.title);
                });

                ui.label(format!("Created at {:?}", OffsetDateTime::from(selected_note.created_on)));

                ui.label("Text");
                egui::TextEdit::multiline(&mut selected_note.text).show(ui);
            }
        });
    }
```
{{< /tab >}}

{{< /tabs >}}

You'll see that I had to account for the `to_remove` and `to_select` variables - Rust forces us to avoid making changes
to that list of notes while we're inside the loop that renders it, where other languages might happily ignore the potential
for trouble that such a thing creates.  This will be a common pattern in list-based applications, so get used to seeing
this kind of code.

Also while we were at it, I gated the note entry form behind a mutable-get-by-index (`self.notes.get_mut()`) - this way,
we only edit a note if we have access to the one we're referring to, and likewise we make the inside code more terse by
avoiding the deeper path reference to `self.notes[self.selected_index]`.

Running now, you should see that we have the ability to add, edit, and delete notes with impunity.

## In closing

Today, we saw how iterative steps in Rust can get us closer to our vision.  It's still hard on the eyes, and doesn't
even begin to do the things I've promised beyond what you can find in a basic notes app, but it should hopefully demonstrate
the process we'll be taking along the way to flesh out the ideal form for this thing.

All changes made in this article can be found in the
[relevant commit on GitHub](https://github.com/bellwether-softworks/graph_note/commit/bb9669c098191f2e4bcf7891ef32e4c8f6a7ffd0).
