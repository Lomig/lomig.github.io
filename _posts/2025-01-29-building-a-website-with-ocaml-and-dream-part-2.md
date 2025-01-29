---
layout: post
title: Building a Website with OCaml and Dream – Part 2
description: Preparing Your Web App for CSS Integration
tags: OCaml Dream Webdev
category: Functional Programming
series: Building a Website with OCaml and Dream
series_description: Learn how to build a modular website using OCaml and Dream.
date: 2025-01-29 19:56 +0100
---
## Series
{% series %}

## Introduction
In this article, we will explore how to integrate [TailwindCSS](https://tailwindcss.com) into an OCaml/Dream project. Tailwind CSS is a modern CSS framework that has gained significant popularity. While there are multiple ways to set it up, we'll aim for a reusable and practical approach rather than repeating manual configurations for each new project.

Unfortunately, there isn’t a ready-made solution to integrate Tailwind CSS into OCaml/Dream projects satisfying enough to me, so we’ll need to implement a custom solution. Let’s dive in.

## Installation
I prefer minimal JavaScript in my projects, so using npm to install Tailwind CSS wasn’t my first choice. Luckily, standalone binaries for Linux, Windows, and macOS are available. We’ll create a script to download the appropriate binary for our system and make it executable during the build process.

###Creating the Download Script

Our script will:
1. Detect the current operating system and CPU architecture.
2. Download the correct Tailwind CSS executable from GitHub.
3. Handle potential redirects during the download process.
`dune` can help us with that!

Here is the implementation:

```ocaml
open Lwt.Syntax

let system_to_version () =
  match ExtUnix.All.uname () with
  | { ExtUnix.All.Uname.sysname = "Darwin"; machine = "arm64"; _ } ->
    "tailwindcss-macos-arm64"
  | { ExtUnix.All.Uname.sysname = "Darwin"; machine = "x64"; _ } ->
    "tailwindcss-macos-x64"
  | { ExtUnix.All.Uname.sysname = "Linux"; machine = "x86_64"; _ } ->
    "tailwindcss-linux-x64"
  | { ExtUnix.All.Uname.sysname = "Linux"; machine = "aarch64"; _ } ->
    "tailwindcss-linux-arm64"
  | _ -> "tailwindcss-windows-x64.exe"
;;

let rec download_with_redirects ~max_redirects uri target =
  if max_redirects <= 0
  then Lwt.fail_with "Too many redirects"
  else
    let* resp, body = Cohttp_lwt_unix.Client.get uri in
    let code = Cohttp.Response.status resp |> Cohttp.Code.code_of_status in
    match code with
    | 200 ->
      let stream = Cohttp_lwt.Body.to_stream body in
      Lwt_io.with_file ~mode:Lwt_io.output target (fun chan ->
        Lwt_stream.iter_s (Lwt_io.write chan) stream)
    | 301 | 302 | 307 | 308 ->
      (match Cohttp.Header.get (Cohttp.Response.headers resp) "location" with
       | Some location ->
         let new_uri = Uri.of_string location in
         download_with_redirects ~max_redirects:(max_redirects - 1) new_uri target
       | None -> Lwt.fail_with "Redirection response missing Location header")
    | _ -> Lwt.fail_with (Printf.sprintf "Failed to download: HTTP %d" code)
;;

let () =
  let target = Filename.concat Sys.argv.(1) "tailwindcss" in
  let version = system_to_version () in
  let base_url =
    "https://github.com/tailwindlabs/tailwindcss/releases/latest/download/"
  in
  let uri = Uri.of_string (base_url ^ version) in
  Lwt_main.run @@ download_with_redirects ~max_redirects:5 uri target
;;
```
{: file="bin/tailwind_download.ml" }

* `ExtUnix.All.uname ()` provides a thin binding over `uname`
* `Cohttp` will be used to download `Tailwind` from Github
* Async download from `Cohttp` forces us to use `Lwt`, a library to handle promises
  * `Lwt_main.run` would execute the promise, wait for its resolve and return its success
  * `Lwt.fail_with` makes the promise fail;
  * `let* x = ... in` is a binding operator; its JS equivalent would be `const x = await ...`
  * `Lwt_` methods are used in place of their synchronous counterparts

> Note: The script is designed to work for macOS (arm64 and x64), Linux (x86_64 and arm64), and Windows. If you find any issues with any OS/CPU combinations, feel free to suggest corrections.
{: .prompt-warning }

### Adding to the Build Configuration
Update your `dune` file to include this new script:
```lisp
(executable
 (public_name tailwind_download)
 (name tailwind_download)
 (libraries extunix cohttp-lwt-unix))
```
{: file="bin/dune" }

Run the script to download the Tailwind CSS binary:
```console
dune exec tailwind_download /a_path/somewhere
```
> [commit 1f77e79](https://github.com/Lomig/quest_complete/tree/1f77e7901eded97f5af8407b1b68cdc2de2e85d8)
{: .prompt-tip }

### Automating the Download
To automate the download during the build process, add a rule to the `dune` file:

```lisp
;; Declaration of our 2 executables
;; ...

(rule
 (target tailwindcss)
 (deps
  (:exe tailwind_download.exe))
 (action
  (progn
   (run echo "Downloading Tailwind CSS...")
   (run ./tailwind_download.exe %{project_root}/bin)
   (run chmod +x %{target})))
 (mode fallback))
```
{: file="bin/dune" }

This rule ensures that:
1. The Tailwind CSS binary is downloaded if it’s not already present.
2. The file is marked as executable.
3. The rule only runs if the binary is missing (`(mode fallback)`).

Build the project:
```console
dune build
```
> [commit 320c2c1](https://github.com/Lomig/quest_complete/tree/320c2c1e7ad43975d914845ceaee9b086e3d379e)
{: .prompt-tip }

## Compiling CSS
With Tailwind CSS installed, we’ll compile CSS files as part of the build process.

### Setting Up Stylesheets
Create a new `static/` directory for static assets and a `client/stylesheets/` folder for your CSS files. Add the following to your main stylesheet:
```css
@import "tailwindcss";
```
{: file="lib/client/stylesheets/application.css" }

### Adding the Compilation Rule
Update the dune file to include a rule for compiling CSS:

```lisp
(rule
 (target application.css)
 (deps
  (:tailwindcss %{project_root}/bin/tailwindcss)
  (:input %{project_root}/lib/client/stylesheets/application.css)
  (source_tree %{project_root}/lib/client/stylesheets)
  (source_tree %{project_root}/lib/server/templates))
 (action
  (chdir
   %{project_root}/lib
   (progn
    (run echo "Building CSS...")
    (ignore-outputs
     (run
      %{tailwindcss}
      -i
      %{input}
      -o
      %{project_root}/../../static/application.css))
    (run cp %{project_root}/../../static/application.css %{target})))))
```
{: file="bin/dune" }

To be able to test it, let's add a Tailwind class to our Hello World, and make use of the generated CSS file.

```html
let render _ =
<p class="text-red-400">Hello world!</p>
```
{: file="server/templates/pages/homepage.eml.html" }


```html
let layout ?(title="QuestComplete") content =
<!DOCTYPE html>
<html>

<head>
    <title>
        <%s! title %>
    </title>
    <link rel="stylesheet" href="static/application.css">
</head>

<body>
    <%s! content %>
</body>

</html>

```
{: file="server/templates/layouts/main.eml.html" }

### Serving Static Files
Finally, serve the static folder through your Dream application:

```ocaml
open Handlers

let routes =
  [ Dream.get "/static/**" (Dream.static "static")
  ; Dream.get "/" Pages.homepage
  ]
;;
```
{: file="server/router.ml" }

Now, you can use Tailwind CSS classes in your templates and see the changes reflected in your application.

```console
dune build
dune exec quest_complete
```
> [commit d82ca17](https://github.com/Lomig/quest_complete/tree/d82ca173d4888bdf9bda48e09a7043870aa60972)
{: .prompt-tip }

## Far Away, far away from Rails...
That's a lot of boilerplate. For reference, adding Tailwind (using the CLI too) to a Rails Project after its init looks like this:
```console
bundle add tailwindcss-rails
rails tailwindcss:install
```
Done. The download, the configuration, the CSS files, all has been handled.

Our own version of it works beautifully, but that's not quite like the smooth experience of such a popular framework.
Or is it?

Introducing... [reality_tailwindcss](https://github.com/Lomig/reality_tailwindcss).
* It goes with `Dream`, hence the name.
* It is opiniated, as it uses the structure I settled upon in this blog (`lib/{client;server};static`{: .filepath})
* There's no unit nor integration tests yet.
But it works!

And we get the smooth Rails experience:
```console
opam pin reality_tailwindcss.1.0.0 git+https://github.com/Lomig/reality_tailwindcss.git#main
reality_tailwindcss install
```
(I'll check how to make a real opam package one day instead of relying on pins)

> [commit c13630a](https://github.com/Lomig/quest_complete/tree/c13630a77298b8c6aa71d77832919844f2ab5145)
{: .prompt-tip }

## Tailwind 4.0

As Tailwind 4.0 has CSS-only configuration, and can `@import` other CSS files with relative paths, we have everything we need regarding CSS now.

In Part 3, we will deal with secrets, as we will need to use credentials for Databases. Stay tuned!
