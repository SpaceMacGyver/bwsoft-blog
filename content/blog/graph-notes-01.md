---
title: Graph Notes - Creating the project
description: Just getting my feet wet, provisioning some resources to get the ball rolling...
params:
  author: Justin Bell
date: 2024-08-15T08:00:00.000-05:00
draft: false
weight: 50
categories:
  - Development
  - Showcase
tags:
  - rust
  - graph_notes
contributors: []
---

My primary goal for today is to get some scaffolding in place.  At the end of this, I'll have a repository, some commits
for you to explore, and enough of a project that you could pull down and run if so inclined.

First, the repo, which can be found on GitHub: [https://github.com/bellwether-softworks/graph_note](https://github.com/bellwether-softworks/graph_note).
The initial commit contains the results of running `cargo new`, followed by a commit to supply the license (MIT).

I'll mention up-front that this project's structure may evolve over time as need dictates.  For now, a single application
suits my needs, but I can also envision a future where things are decoupled (e.g. a project for business logic, one for
an API server, another for the front-facing application).  Today, I'm keeping it simple.

I've talked before about [creating a new project with egui](/blog/practical-rust-desktop-gui-development-with-egui-pt.-1/#eframe).
In that article, I relied on a template project to wire a lot of stuff up; today, I'm instead going to do it by hand, in part
to give that perspective to anyone looking for what it takes to inject egui into an existing project.  That said, I'll be
referring to the template project's [**Cargo.toml**](https://github.com/emilk/eframe_template/blob/18a8e76c2525862216410a0db877833090db3ff5/Cargo.toml)
file to help inform my dependency selections:

```bash
$ cargo add serde@1 --features derive
$ cargo add egui
$ cargo add egui_extras
$ cargo add eframe --no-default-features --features accesskit,default_fonts,glow
```

For today, I'm developing a local executable, but I'm likely going to revisit the Cargo.toml to add support for WASM targets.

Also, while we're in the template project, we'll want to look at how they structure the app:

* **src/main.rs** is the entry point for our executable; it wires up the core app and passes that to eframe for execution
* **src/app.rs** is where the egui application is defined, and provides the lifecycle hooks needed to run our program loop

I'll start first by adapting the [`fn main()` implementation from the template](https://github.com/emilk/eframe_template/blob/18a8e76c2525862216410a0db877833090db3ff5/src/main.rs#L5-L25)
to our own needs, starting simple; the full file contents for the moment are as shown below:

```rust
mod app;

#[cfg(not(target_arch = "wasm32"))]
fn main() -> eframe::Result {
    let native_options = eframe::NativeOptions::default();
    
    eframe::run_native(
        "Graph Notes",
        native_options,
        Box::new(|cc| Ok(Box::new(app::App::new(cc)))),
    )
}
```

You can see I'm referencing a yet-to-be-created `app::App` struct; let's head over to **src/app.rs** and flesh out the
minimum requirements for our nascent egui application:

```rust
use eframe::Frame;
use egui::Context;

pub struct App;

impl Default for App {
    fn default() -> Self {
        Self
    }
}

impl App {
    pub fn new(_cc: &eframe::CreationContext<'_>) -> Self {
        // Access to the CreationContext during instantiation allows us to take
        // ownership of some aspects of our app, such as wiring up fonts.
        Default::default()
    }
}

impl eframe::App for App {
    fn update(&mut self, ctx: &Context, frame: &mut Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.label("Greetings from the skeleton app.");
        });
    }
}
```

The `eframe::App` trait offers us more hooks to take advantage of, but shown here is just enough code to be able
to render out the following:

{{< figure src="skeleton-app.png" >}}

Nothing fancy thus far, but this gives us enough of a starting point that we can start fleshing out some ideas in our
next go.
