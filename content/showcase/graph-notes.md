---
title: "Graph Notes"
description: "An application that allows for expressive note-taking"
params:
  author: Justin Bell
summary: ""
date: 2024-08-11T07:00:00-05:00
lastmod: 2024-08-11T07:00:00-05:00
draft: false
weight: 999
toc: false
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

*I'll periodically update this page to reflect the project's evolution; keep an eye on it for updates.*

## Overview

For years, I've envisioned a note-taking system that allows for expression (e.g. via Markdown) and complex mind-mapping
(graph database).  I originally assumed I'd build something with NodeJS and React (this was back in the mid 2010s), and
got particularly excited when I discovered [Neo4j](https://neo4j.com/) as it seemed like the perfect fit for my mental model.  Well, as fads
are wont to do, the architecture of my app kept changing in my mind as cool new things erupted - different ways of modeling
state, different approaches to UX, even my evolving tastes and ideas around how best to model documents.

Then, several years later, I discovered [Obsidian](https://obsidian.md/).  Someone had independently come to many of the
same ideas I'd been tossing around and fleshed it out into a robust and feature-rich application.  In fact, I've been using
it for a couple of years to handle my assorted note-taking and creative writing ventures, and I have to say it handles about
95+% of what I want from my note-taking experience.  For me, though, it's greatest strength is also its most formidable
weakness - its storage format, which consists of a folder structure and documents comprised of Markdown.  Effectively,
inside this hierarchical graph structure of folders, individual nodes are comprised of documents, and links are basically
inferred from the documents' contents.

Instead, what I'd like to create is first and foremost a database of graph nodes, within which can be assorted resources
that will predominantly include textual content.  It's the graph, not the individual documents themselves, that I believe
have lasting value, a value system that's heavily influenced by Neo4j's [Cypher DSL](https://neo4j.com/docs/cypher-manual/current/introduction/cypher-overview/)
and its mechanics.  Rather than relying on parsing out a semi-structured text document for meaning, the meaning resides
in the nodes' and the edges' attributes.  Furthermore, once the data exists, the query grammar allows for an expressive
means of retrieving data and relationships in unplanned and perhaps unexpected ways.  Basically, I'm solving a different
problem than Obsidian - rather than creating a document database that results in a graph, I'm creating a graph that may or
may lead to new discoveries from captured ideas.

## Goals

At a glance, what I'm hoping to get out of a note-taking app would include:

* Rich-text editing capabilities
* Arbitrary classification systems
* Node versioning, revision history
* Robust query engine leveraging graph system
* Easy backups and export
* Means of importing from existing sources

### Stretch goals

A future version of this application would be multi-user and offer collaborative tools.

* Ownership model with access control, rights management
* Concurrent editing capabilities between multiple editors
* Schema-based enforcement of structure to ensure graphs are well-formed for specific use-cases

Additionally, while the above is with respect to note-taking and document management, there may be other viable uses
for the engine, and it may have opportunity to be adapted to custom analysis and reporting.

## The toolkit

I've been daily-driving the Rust programming language for several years now, and when I first started off (late 2010's),
the ecosystem of third-party libraries was still rather nascent.  There were plenty of existing tools in the Crates repository
system, but quite often I'd find that the thing I desired to use had yet to be created.  Today, as I reevaluate the scene,
I'm now finding the pieces that I believe will lead to success for my vision, and I'm as confident as ever that this is the
appropriate way to move forward.

### Rust

First, developing in Rust is simultaneously fulfilling and confidence-inspiring.  I can know as I write that, if any issues
exist with my application, it's almost certainly a fault with my own planning and not a shortcoming of the language; I get
the benefits of performance, compile-time quality assurance, and durable shelf-life of any code I produce.  There are
plenty of reasons to choose this language - though, to be clear, it's not to say that choosing one language over another
is so clear-cut and objectively correct.  In my own case, though, this is the environment that (for me) offers the confluence
of features, opportunity, and developer experience that allow me to create a feature-rich application in the first draft.

### egui

Language aside, there's the matter of graphical frameworks.  I've explored multiple options, each with its respective
merits and considerations, and feel that as of this moment the one that gets me closest to my vision is [egui](/tags/egui).
It's still a developing framework and imposes some limitations, particularly when compared to HTML with its rich layouting
and rich ecosystem of functionality and customization.  An alternative direction I could have pursued may have been to
leverage [Tauri](https://tauri.app/) to build a decoupled backend/frontend architecture that could take advantage of decades'
worth of frontend innovation, with the added advantage of being able to outsource or crowdsource the frontend to other UI
developers.

When it comes to rapid prototyping with a predictable ecosystem, though, I've found egui to be an unmatched experience.
Its immediate-mode interface allows for expression without the costs of reconciling application state with the
human-machine-interface woes that come with an event-driven architecture.  Its greatest shortcoming, the fact that it's
still a work-in-progress with some missing features and functionality, will likely resolve itself with enough time and a
growing userbase to drive innovation.

Additionally, this choice allows for consistent and predictable behavior across major platforms - Windows, Linux, Mac,
and web (via WASM) are all well-supported and should behave largely identically.

### [egui_graphs](https://github.com/blitzarx1/egui_graphs/tree/master)

Building off of my choice to move forward with egui, a component of my vision that's been missing until recently has been
the means by which I could visualize graphs.  I had been convinced for some time that I'd end up needing to roll my own
framework to make it happen, but someone else has done the heavy lifting for me and produced a reusable component for my
visualization needs.

I'm still sizing up how best to leverage this tool, and will explore that notion during development via iteration.

### [SurrealDB](https://github.com/surrealdb/surrealdb?tab=readme-ov-file)

This multi-paradigm database server has much to offer, and is effectively the linchpin of this project.  It's not the only
graph database option available, but this one in particular offers me several advantages:

* Can be nested within my app rather than living as an external process
* Data modeling with structured schema
* Rich query/DML grammar
