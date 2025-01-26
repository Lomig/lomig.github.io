---
layout: post
title: Building a Website with OCaml and Dream – Part 1
description: Learn how to build a modular website using OCaml and Dream, starting with project setup, routing, and templating.
date: 2025-01-26 17:38 +0100
tags: OCaml Dream Webdev
category: Functional Programming
series: Building a Website with OCaml and Dream
series_description: Learn how to build a modular website using OCaml and Dream.
---
## Series
{% series %}

## Introduction

I am an object-oriented programming (OOP) developer, and my primary working language is Ruby. However, I have always had a profound love for functional programming languages, especially OCaml. It was the first programming language I learned at [school](https://en.wikipedia.org/wiki/Classe_préparatoire_aux_grandes_écoles), though back then it was still Caml Light. Later, during my computer science studies in college, functional programming remained central to my education, and I continued exploring it through languages like Scheme and LISP. Eventually, OOP became my professional focus, but I never stopped longing for the functional side of programming.

To bridge the gap and reignite my passion for functional programming, I set myself two challenges:

1. Dive deeply back into OCaml, evangelize it, (and perhaps even work professionally with it one day in a distant future!)
2. Create approachable documentation and tutorials for functional programming languages, particularly OCaml. My goal is to demonstrate that anyone can use OCaml effectively through down-to-earth, step-by-step tutorials and practical blog posts—the kind of resources I wish I had when I started.

### My Background

Besides being a Senior Software Engineer working mainly in Ruby and Go, I also dedicate a part of my working time *teaching* software engineering, which is why I hope I can bring something to the table with such a project.

### The App

I love lists. I make lists of things I enjoy, things I’ve watched, books I’ve read, and games I’ve played. For this series of blog posts, I’ll create a very simple web app that allows me to share the games I’ve finished, when I finished them, and how much I enjoyed them.

Here’s the planned stack:

- I’ll use `Dream` as the framework for the website.
- I’ll experiment with incorporating some Ruby on Rails technology, like [Hotwire](https://hotwired.dev), just to see if it’s feasible.
- If I can use OCaml for frontend development, I’ll explore that as well.

The idea is to document this experiment and help others who may hesitate to try OCaml or face roadblocks they can’t overcome.

### Acknowledgement

I’ve been an OOP programmer for many years, with Ruby as my main language. This means I’ve developed certain habits that may be difficult to set aside. Even within OOP, my views are often considered opinionated. When it comes to functional programming, I may take unconventional approaches, think outside the box, or adopt practices that are not standard in the functional programming world. I apologize in advance and invite you to share advice if you have expertise in this area—because I’m learning as I go.

### Prerequisites

* I’ll assume you have `OPAM` and `Dune` installed and working. This will be my only assumption about your setup.
* I'll assume you know the basics about `OCaml` syntax. I don't know much more myself!
* I chose `Containers` as my standard library

## Hello World!

### Creating the Project
To start, let's use dune to create a new project:

```terminal
dune init project quest_complete
```
> [commit 06b662d](https://github.com/Lomig/quest_complete/tree/06b662d52fe514e0851cb53187e7377723e5e8be)
{: .prompt-tip }

This will create the following structure:
```
quest_complete/
├── _build/
├── bin/
│   ├── dune
│   └── main.ml
├── lib/
│   └── dune
├── test/
│   ├── dune
│   └── test_quest_complete.ml
├── dune-project
└── quest_complete.opam
```
Next, we'll add the [Dream library](https://aantron.github.io/dream/) and some other dependencies. Run the following command to install them:

```terminal
opam install containers dream
```
Here's the content of the dune-project file with some basic information filled in:

```lisp
(lang dune 3.17)

(name quest_complete)

(generate_opam_files true)

(source
 (github Lomig/quest_complete))

(authors "Lomig <lomig@hey.com>")

(maintainers "Lomig <lomig@hey.com>")

(license GPL-3.0-only)

(package
 (name quest_complete)
 (synopsis "Track and celebrate the games you’ve conquered.")
 (description "QuestComplete is your ultimate companion for tracking completed video games. Keep a record of your gaming victories, relive your favorite quests, and celebrate your progress in style. Level up your gaming organization with QuestComplete!")
 (depends ocaml containers dream)
 (tags
  ("website" "games" "completion")))
```
{: file="dune-project" }

> [commit 85e5867](https://github.com/Lomig/quest_complete/tree/85e5867a8cf38c2879360e32b7594946810ca2c8)
{: .prompt-tip }

### Setting Up a Basic Web Server

The `Dream` documentation is straightforward, so let's configure it. Update the `dune` file for the executable:

```lisp
(executable
 (public_name quest_complete)
 (name main)
 (libraries quest_complete dream))
```
{: file="bin/dune" }

Now, let's create a minimal web server in `main.ml`:
```ocaml
let () =
  Dream.run
  @@ Dream.logger
  @@ Dream.router [
    Dream.get "/" (fun _ ->
      Dream.html ("hello world"));
  ]
```
{: file="bin/main.ml" }

This snippet does the following:

1. Starts a Dream web server.
2. Adds middleware to log server activity.
3. Defines an array of routes.
4. Sets up a `GET` route for the root (`/`) that returns an HTML response.

Routes follow this structure:
`Dream.VERB PATH -> (fun REQUEST -> RESPONSE)`

Finally, build and run your project:

```terminal
dune exec quest_complete
```

> [commit c9c110c](https://github.com/Lomig/quest_complete/tree/c9c110c8c2cdef87cc37c1eac86f60054eb3051c)
{: .prompt-tip }

### Organizing Code (MVC-ish Style)
To make the project more modular, we'll structure it in an MVC-like way.

#### Router
We'll organize server and client code into separate directories under `lib`. Let's move the routes to a dedicated `router.ml` file inside the `server` directory. Update the `dune` files accordingly.

```lisp
(library
 (name server)
 (libraries dream))
```
{: file="lib/server/dune" }

```ocaml
let routes =
  [
    Dream.get "/" (fun _ -> Dream.html ("hello world"));
  ]
;;
```
{: file="lib/server/router.ml" }


Update the main executable to use the new router:
```lisp
(executable
 (public_name quest_complete)
 (name main)
 (libraries dream server router))
```
{: file="bin/dune" }


```ocaml
open Server

let () =
  Dream.run
  @@ Dream.logger
  @@ Dream.router Router.routes
```
{: file="bin/main.ml" }


Rebuild and run:
```terminal
dune exec quest_complete
```
Everything should still work!

> [commit 73265f4](https://github.com/Lomig/quest_complete/tree/73265f4a8894bbe4406ad506b6081dc1b33cf339)
{: .prompt-tip }

#### Handlers
`Dream` introduces the concept of `handlers`, which are essentially controllers. Let's create a dedicated space for them under `lib/server/handlers`.

##### New files
```lisp
(library
 (name handlers)
 (libraries dream))
```
{: file="lib/server/handlers/dune" }


```ocaml
(* pages will deal with static pages of the website *)
let homepage _ = Dream.html ("hello world")
```
{: file="lib/server/handlers/pages.ml" }


##### Updates

```lisp
(library
 (name server)
 (libraries dream handlers))
```
{: file="lib/server/dune" }


```ocaml
open Handlers

let routes =
  [
    Dream.get "/" Pages.homepage
  ]
;;
```
{: file="lib/server/router.ml" }


Check the server again:
```terminal
dune exec quest_complete
```
> [commit 39d490a](https://github.com/Lomig/quest_complete/tree/39d490ad0fe675fbd4aa6b1497dee3ae8c36ee7b)
{: .prompt-tip }

#### Views
Views handle the interface with the user. We'll move the presentation logic into a views directory.

##### New files
```lisp
(library
 (name views))
```
{: file="lib/server/views/dune" }


```ocaml
let homepage = "hello world"
```
{: file="lib/server/views/pages.ml" }


##### Updates
```lisp
(library
 (name handlers)
 (libraries dream views))
```
{: file="lib/server/handlers/dune" }


```ocaml
let homepage _ = Views.Pages.homepage |> Dream.html
```
{: file="lib/server/handlers/pages.ml" }

Rebuild and run the project:

```terminal
dune exec quest_complete
```
> [commit cfa44de](https://github.com/Lomig/quest_complete/tree/cfa44dede2068788a38aac33993811ec09cfd5b1)
{: .prompt-tip }

#### Raw Templating
Mixing HTML with logic can get messy. Dream supports templating, which we'll use to separate concerns.

##### New files
```lisp
(library
 (name templates)
 (libraries dream))

(include_subdirs qualified)
```
{: file="lib/server/templates/dune" }


In OCaml, each `ml` file transposes to a module; but as there will be no file directly in the `templates` directory and we need to nest sub-directories to be considered as nested modules, we use the `(include_subdirs qualified)` stanza.

Consider this folder structure:
```
templates/
├── foo/
│   ├── bar.ml
│   └── ber.ml
├── buzz/
│   └── bazz.ml
└── dune
```
* if `dune` contains the stanza `(include_subdirs qualified)`, we will get access to modules `Templates.Foo.Bar`, `Templates.Foo.Ber` and `Templates.Buzz.Bazz`
* if `dune`  contains the stanza `(include_subdirs unqualified)`, we will get access to modules `Templates.Bar`, `Templates.Ber` and `Templates.Bazz`


```ocaml
let render _ = "hello world"
```
{: file="lib/server/templates/pages/homepage.ml" }


##### Changed files
```lisp
(library
 (name views)
 (libraries templates))
```
{: file="lib/server/views/dune" }


```ocaml
let homepage = Templates.Pages.Homepage.render None
```
{: file="lib/server/views/pages.ml" }

> [commit 00be062](https://github.com/Lomig/quest_complete/tree/00be0626bc7c61c322bf8a81074ad8a5990d66a4)
{: .prompt-tip }

#### Embedded ML Templating
I want to serve some `HTML` though, and for that I'll use what `Dreams` provide: a templating system. The main idea is:
* Writing some code mixing HTML and OCaml (think Moustache or ERB for example)
* On build, having a preprocessor that would translate the template in pure HTML, wrapped up in OCaml functions which `Dream` can use.
* We will rename `templates/pages/homepage.ml` to `templates/pages/homepage.eml.html` to mark it as being Embedded ML within HTML.

##### New files
```lisp
(rule
 (targets homepage.ml)
 (deps homepage.eml.html)
 (action
  (run dream_eml %{deps} --workspace %{workspace_root})))
```
{: file="lib/server/templates/pages/dune" }


A `rule` will map `deps` to `targets` using the `action`.
* `deps` are files in the current directory
* `targets` will be generated in the `_build` directory on build

```html
let render _ =
<!DOCTYPE html>
<html>

<head>
    <title>Homepage</title>
</head>

<body>
    <p>Hello world!</p>
</body>

</html>
```
{: file="server/templates/pages/homepage.eml.html" }


Our usual check it still works:
```terminal
dune exec quest_complete
```

> [commit 01bdf25](https://github.com/Lomig/quest_complete/tree/01bdf25e5af1895440672af77d772077f07ff977)
{: .prompt-tip }

#### Layouts
To avoid duplicating boilerplate, create a layout template:

##### New files
```lisp
(rule
 (targets main.ml)
 (deps main.eml.html)
 (action
  (run dream_eml %{deps} --workspace %{workspace_root})))
```
{: file="lib/server/templates/layouts/dune" }

```html
let layout ?(title="QuestComplete") content =
<!DOCTYPE html>
<html>

<head>
    <title><%s! title %></title>
</head>

<body>
    <%s! content %>
</body>

</html>
```
{: file="server/templates/layouts/main.eml.html" }


##### Changed files

```html
let render _ =
<p>Hello world!</p>
```
{: file="server/templates/pages/homepage.eml.html" }


```ocaml
let homepage = Templates.Layouts.Main.layout @@ Templates.Pages.Homepage.render None
```
{: file="lib/server/views/pages.ml" }


> [commit 989cbb5](https://github.com/Lomig/quest_complete/tree/989cbb5d6ef2cbe0ce8e9ca644e8e69a5dd5390e)
{: .prompt-tip }

Now you have a static, basic website in OCaml! In the next part, we'll explore database connections, add CSS, and more.
