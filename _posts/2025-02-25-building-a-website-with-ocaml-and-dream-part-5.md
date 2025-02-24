---
layout: post
title: Building a Website with OCaml and Dream – Part 5
description: Type-safe HTML and OCaml Server-Side Components with a touch of Assets
  Management
tags: OCaml Dream Webdev
category: Functional Programming
series: Building a Website with OCaml and Dream
series_description: Learn how to build a modular website using OCaml and Dream.
date: 2025-02-25 11:40 +0100
---
## Series
{% series %}

## Introduction

First, I want to give a huge shoutout to [Chukwuma](https://bsky.app/profile/aguluman.bsky.social) for dedicating his time to testing the code from my previous articles on Windows. Based on his feedback, I’ve updated those blog posts to include the necessary information to make the code work on Windows. It may not be perfect yet, but I'll continue refining it as I receive more feedback.

Since this series is about documenting my journey of building a website with OCaml and Dream, I sometimes change my mind about things I’ve previously discussed. `eml` templates are one such case. Generally, I’m not a fan of DSLs for writing HTML in any language or framework — I prefer to stay as close as possible to raw HTML. Initially, I followed this approach while building a Games Index, adapting the concept of `partials` from Rails.

There’s nothing wrong with this approach, and if you prefer it, you can absolutely keep using it. It doesn’t significantly impact how we use Dream to create a website, nor does it introduce major changes in the tooling needed elsewhere in the code. However, I started feeling that type-safe HTML was a better fit for a functional programming language like OCaml. Plus, I was getting frustrated with having to choose between syntax highlighting for OCaml or HTML, but not both. After making the switch, I’m quite happy with the results.

That brings us to the topic of this article:
- Adding the Index view to the Games module as part of the traditional CRUD approach
- My implementation of views using Partials, then evolving into a Component-based structure
- Type-safe HTML with [dream-html](https://github.com/yawaramin/dream-html)
- Asset management (with [Stimulus JS](https://stimulus.hotwired.dev) as a bonus)

As always, if you have any remarks, questions, or suggestions, feel free to reach out. I might be a bad software engineer, but at least I have no ego about it!

## CRUD: Index

### Router, Handler, Domain
First, we need a route to display the list of completed games.
  
```ocaml
let routes =
  [ Dream.get "/static/**" (Dream.static "static")
  ; Dream.get "/" Pages.homepage
  ; Dream.get "/games" Games.index
  ; Dream.get "/games/:id" Games.show
  ]
;;
```
{: file="lib/server/router.ml" }

This leads us to the implementation of the `Games.index` handler:

```ocaml
(* Previous code for the `show` function *)

let index request =
  let* games = Game.all ~request () in
  games |> Views.Games.Index.render |> Dream.html
;;
```
{: file="lib/server/handlers/games.ml" }

Now, we return to our `Domain` module to implement the `all` function.

Since we are displaying a list of completed games, the order of the list is important. We'll sort it in descending order by completion date while keeping the option to modify the ordering if needed. Along the way, I discovered an OCaml quirk: functions cannot have optional arguments unless they also have at least one positional argument. To work around this, I added a `unit` (`()`) argument, even though the named argument `~request` is mandatory.

For the `all` function, we build upon what we implemented in the previous article regarding SQL queries:

```ocaml
let all ~request ?(order_by = "completion_date") ?(direction = `Desc) () =
  let order =
    match direction with
    | `Asc -> order_by ^ " ASC"
    | `Desc -> order_by ^ " DESC"
  in
  let open DB in
  let open Lwt.Syntax in
  let query = (string ->* t) "SELECT * FROM games ORDER BY ?" in
  let all' =
    fun (module Db : Connection) ->
    let* games = Db.collect_list query order in
    Caqti_lwt.or_fail games
  in
  let* result = all' |> Dream.sql request in
  Lwt.return result
;;
```
{: file="lib/server/domain/games.ml" }

### Views

The view is straightforward, building on what we’ve already established. Each game entry in the list will display its cover, title, and rating (if available) while linking to the `show` page we built earlier.

```ocaml
let index_data (games : Game.t list) : Templates.Games.Index.data =
  List.map
    (fun (game : Game.t) : Templates.Games.Partials.IndexGame.data ->
       { uri = "/games/" ^ game.id
       ; title = game.title
       ; rating =
           (match game.rating with
            | None -> "N/A"
            | Some r -> string_of_int r)
       ; cover_url = game.cover_url
       })
    games
;;

let index games =
  Templates.Layouts.Main.layout @@ Templates.Games.Index.render @@ index_data games
;;
```
{: file="lib/server/views/games.ml" }

For the `eml` template to be compiled by `Dune`, we need to add it to the `dune` file:

```lisp
(rule
 (targets show.ml index.ml)
 (deps show.eml.html index.eml.html)
 (action
  (run dream_eml %{deps} --workspace %{workspace_root})))
```
{: file="lib/server/templates/games/dune" }

The template itself uses `Partial` views to promote reusability and reduce code duplication. It contains a little more OCaml code than before, but it remains readable and maintainable:

```html
type game_data = Partials.IndexGame.data
type data = game_data list

let render data =
let li_content = String.concat "\n" @@ List.map Partials.IndexGame.render data in

<h1 class="text-red-800 text-4xl">Mes jeux terminés</h1>
<ul>
    <%s! li_content %>
</ul>
```
{: file="lib/server/templates/games/index.eml.html" }

Since partials are also `eml` templates, they need to be compiled as well. The `dune` file follows the same structure as before:

```lisp
(rule
 (targets indexGame.ml)
 (deps indexGame.eml.html)
 (action
  (run dream_eml %{deps} --workspace %{workspace_root})))
```
{: file="lib/server/templates/games/partials/dune" }

```html
type data =
{ uri: string
; title : string
; rating : string
; cover_url : string
}

let render data =
<li>
    <a href="<%s data.uri %>" class="flex items-center">
        <img src="<%s data.cover_url %>" alt="<%s data.title %>" class="w-1/4">
        <div>
            <%s data.title %>
        </div>
        <div>
            <%s data.rating %>
        </div>
    </a>
</li>
```
{: file="lib/server/templates/games/partials/indexGame.eml.html" }

> [commit a731b6f](https://github.com/Lomig/quest_complete/tree/a731b6f9a9e56bb4cf72280b0578987012f37402)
{: .prompt-tip }

See? No need to stress about my change of mind regarding `eml` templates! They’re still easy to use and remain powerful, so you can continue using them while following this series.

## From Partials to Components in EML Templates
The difference between partials and components is rather subtle. One way to put it (though you may disagree) is that a partial is a snippet of HTML meant for reuse in different places, whereas a component is a piece of _code_ that produces HTML and can contain reusable logic. In my implementation, this distinction is even finer since `eml` templates are essentially glorified OCaml files. But bear with me!

> Here again, keeping to partials is perfectly fine and has served the Rails community well for years. I just wanted to try something new and see how it goes.
{: .prompt-info }

### Component Functor

Our components should meet the following requirements:
* They should be renderable (obviously).
* They should support rendering with encompassing tags — for example, wrapping the output in `<li>...</li>` as in our earlier partial.
* They should be capable of rendering a list of similar components. In the partial example, we needed extra code to render a list of partials.
* They should produce strings for the view so that type handling is not an issue.

This naturally calls for a functor, doesn't it?

```ocaml
open Containers
module Games = Games

module type COMPONENT = sig
  type data

  val render : data -> string
end

module MakeComponent (C : COMPONENT) = struct
  type data = C.data

  let render data = C.render data

  let render_with_tag ~tag data =
    let inner_html = C.render data in
    Printf.sprintf "<%s>%s</%s>" tag inner_html tag
  ;;

  let render_list ~tag data_list =
    data_list
    |> List.map (fun data ->
      match tag with
      | Some tag -> render_with_tag ~tag data
      | None -> render data)
    |> String.concat "\n"
  ;;
end
```
{: file="lib/server/views/components/components.ml" }

As I want Components to be _real_ Ocaml files, we will use a `PPX` to generate the `eml` part of components.

```lisp
; Add to dependencies:
(depends ... ppx_dream_eml)
```
{: file="dune-project" }

```lisp
(library
 (name components)
 (libraries containers dream)
 (preprocess
  (pps ppx_dream_eml)))

(include_subdirs qualified)
```
{: file="lib/server/views/components/dune" }

And voilà, our first component!

```ocaml
type data =
  { uri : string
  ; title : string
  ; rating : string
  ; cover_url : string
  }

let render data =
{% raw %}
  {%eml|
<a href="<%s data.uri %>" class="flex items-center">
    <img src="<%s data.cover_url %>" alt="<%s data.title %>" class="w-1/4">
    <div>
        <%s data.title %>
    </div>
    <div>
        <%s data.rating %>
    </div>
</a>
|}
{% endraw %}
;;
```
{: file="lib/server/views/components/games/card.ml" }

The view now uses the `MakeComponent` functor to render the component as a list:

```lisp
(library
 (name views)
 (libraries containers components templates domain))
```
{: file="lib/server/views/dune" }


```ocaml
(*===========================================================================*
 * Game Index
 *===========================================================================*)
module IndexGameCardComponent = Components.MakeComponent (Components.Games.Card)

let index_data (games : Game.t list) : Templates.Games.Index.data =
  let games_data =
    List.map
      (fun (game : Game.t) : IndexGameCardComponent.data ->
         { uri = "/games/" ^ game.id
         ; title = game.title
         ; rating =
             (match game.rating with
              | None -> "N/A"
              | Some r -> string_of_int r)
         ; cover_url = game.cover_url
         })
      games
  in
  { card_components = IndexGameCardComponent.render_list ~tag:(Some "li") games_data }

let index games =
  Templates.Layouts.Main.layout
  @@ Templates.Games.Index.render
  @@ index_data games
```
{: file="lib/server/views/games.ml" }

```html
type data = { card_components : string }

let render data =
<h1 class="text-red-800 text-4xl">Mes jeux terminés</h1>
<ul>
    <%s! data.card_components %>
</ul>
```
{: file="lib/server/templates/games/index.eml.html" }

> [commit e0b3318](https://github.com/Lomig/quest_complete/tree/e0b3318729816315708793b7385910ba91ca4044)
{: .prompt-tip }

## Dream-html and Type-safe HTML
I should probably call it `PureHTML` instead, as I won’t be using the parts of this library that interact with `Dream` — only its DSL for writing HTML in a type-safe way. Maybe one day, I’ll be convinced by other aspects of this library!

As the project grows, I’ve been thinking about its structure:

* There are quite a few folders now.
* I like to keep things organized, and since we’re working exclusively with OCaml now, do we really need a strict separation between `views` and `templates`?
* If everything is under `views`, I don’t want to mix conceptual structures like `components` or `layouts` with domain-related folders like `games` or `users`.
* Alphabetical ordering isn’t always the best for semantic organization. In Rails, an underscore at the beginning of a folder name prevents it from being picked up by the framework. I’ll mimic this behavior to improve file organization fro, the `dune` files.

*(I don’t have a clean solution for `dune` files themselves yet, so if anyone has ideas, I’d love to hear them!)*

### New Project Structure

```text
lib/
└── server/
    ├── domain/
    │   ├── date.ml
    │   ├── date.mli
    │   ├── dune
    │   ├── game.ml
    │   └── platform.ml
    ├── handlers/
    │   ├── dune
    │   ├── games.ml
    │   └── pages.ml
    ├── views/
    │   ├── _components/
    │   │   ├── games/
    │   │   │   └── card.ml
    │   │   └── components.ml
    │   ├── _layouts/
    │   │   ├── dune
    │   │   ├── layouts.ml
    │   │   └── main.ml
    │   ├── games/
    │   │   ├── index.ml
    │   │   └── show.ml
    │   ├── pages/
    │   │   └── homepage.ml
    │   └── dune
    ├── dune
    └── router.ml
```
{: file="tree.txt" }

If a page becomes too complex to fit neatly into a single view file, we still can use a layout to encapsulate the complexity.

### Components Using `PureHTML`

Components remain conceptually the same, but instead of returning a `string`, they now return a `PureHTML` `node`, making them more type-safe.

```ocaml
open Containers
open Fun
open Pure_html
module Games = Games

module type COMPONENT = sig
  type data

  val node : data -> node
end

module MakeComponent (C : COMPONENT) = struct
  type data = C.data

  let node data = C.node data
  let html = to_string % node

  let node_with_tag ~(tag : node list -> node) data =
    let inner_node = C.node data in
    tag [ inner_node ]
  ;;

  let html_with_tag ~tag = to_string % node_with_tag ~tag

  let node_list ?tag data_list =
    data_list
    |> List.map (fun data ->
      match tag with
      | Some tag -> node_with_tag ~tag data
      | None -> C.node data)
  ;;

  let html_list ?tag data_list =
    String.concat "\n" @@ List.map to_string @@ node_list ?tag data_list
  ;;
end
```
{: file="lib/server/views/_components/components.ml" }

### Type-safe Views with `PureHTML`

Each view now has its own file, and `eml` templates are replaced with `PureHTML` nodes.  
For example, here’s the updated `Games.Index` view:

```ocaml
open Containers
open Domain
open Pure_html
open Pure_html.HTML
module GameCardComponent = Components.MakeComponent (Components.Games.Card)

let extract_data_from (games : Game.t list) =
  let component_of_game : Game.t -> GameCardComponent.data =
    fun game ->
    { uri = "/games/" ^ game.id
    ; title = game.title
    ; rating = Option.map_or ~default:"N/A" string_of_int game.rating
    ; cover_url = game.cover_url
    }
  in
  List.map component_of_game games
;;

let render games =
  let index_card_list =
    extract_data_from games |> GameCardComponent.node_list ~tag:(li [])
  in
  Layouts.Main.layout
    [ h1 [ class_ "text-red-800 text-4xl" ] [ txt "Mes jeux terminés" ]
    ; ul [] index_card_list
    ]
  |> to_string
;;
```
{: file="lib/server/views/games/index.ml" }

> [commit a54bffb](https://github.com/Lomig/quest_complete/tree/a54bffb2243b2c38b8381dc29faefab475aa45ac)
{: .prompt-tip }

This approach moves further from raw HTML but brings in more elegant, type-safe OCaml!

## Managing Assets
Browsers cache assets aggressively, which can cause issues when updates don’t immediately take effect. A common solution is to **fingerprint assets**, ensuring each version has a unique name. There are several approaches to this, but the simplest and most secure method is to name assets after their **MD5 hash**.

However, this introduces a challenge: developers shouldn’t have to manually update asset references every time a file changes. The **asset pipeline** should handle this automatically by providing helpers to generate the correct paths while managing fingerprinting under the hood.

Additionally, I want to use **Stimulus** for my JavaScript needs. As a strong proponent of the **no-build paradigm**, I also want to manage my JS dependencies using **Import Maps** rather than a bundler.

### The Plan 
To achieve this, my approach is:

* **Fingerprint every file** in the `static` folder when the server starts.
* **Map original filenames to fingerprinted versions**, with a utility function to retrieve the correct path.
* **Generate Import Map tags** from both local JS assets and external libraries.

Since my setup must support both **EML templates** and my work with **PureHTML**, I’ll make sure the tag generation works seamlessly with both.

### Fingerprinting Assets
The fingerprinting process follows these steps:
* Take the name of the asset
* Remove the extension
* Compute the md5 hash of the file
* Append the hash to the name
* Append the extension

Here’s how it looks in OCaml:


```ocaml
module Fingerprint = struct
  (* is_fingerprinted is a function based on a simple Regex *)
  (* remove_from is the opposite of add_to, and a fuction based on the same Regex *)
  
  let md5_of_file filename =
    IO.with_in filename (fun ic ->
      let digest = Digest.channel ic (-1) |> Digest.to_hex in
      digest)
  ;;
  
  let add_to filename =
    match is_fingerprinted filename with
    | true -> filename
    | false ->
      let extension = Filename.extension filename in
      let basename = Filename.remove_extension filename in
      let fingerprint = md5_of_file filename in
      basename ^ "-" ^ fingerprint ^ extension
  ;;
  
  let rename_file filename =
    match is_fingerprinted filename with
    | true -> ()
    | false -> Sys.rename filename (add_to filename)
  ;;
end
```

Applying this to all files in `static` is straightforward:
```ocaml
let recursive_file_list dir =
  let rec aux acc = function
    | [] -> acc
    | file :: files ->
      (match Sys.is_directory file with
       | true ->
         let new_files =
           Sys.readdir file |> Array.to_list |> List.map (Filename.concat file)
         in
         aux acc (files @ new_files)
       | false -> aux (file :: acc) files)
  in
  aux [] [ dir ]
;;

let fingerprint ?(path = "static/") () =
  recursive_file_list path
  |> List.fold_left
    (fun asset_map filename ->
      let original_filename = Fingerprint.remove_from filename in
      let fingerprinted_filename = Fingerprint.add_to filename in
      Fingerprint.rename_file filename;
      StringMap.add original_filename fingerprinted_filename asset_map)
  StringMap.empty
;;
```

### Generating Import Maps
To ensure JavaScript dependencies are correctly loaded, the Import Map must:
- Reference fingerprinted assets.
- Include manually imported JS libraries from external sources.
- Generate both the **Import Map JSON** and **preload tags** for better performance.

Some browsers won’t load assets listed in the Import Map unless explicitly imported in the HTML, so we’ll generate both:

- The **Import Map JSON** (for `script type="importmap"`).  
- The **preload modules** (for `<link rel="modulepreload">`).  

Here’s how that works in OCaml:

```ocaml
module ImportMap = struct
  type t = (string * string) list

  let to_json (list : t) =
    `Assoc [ "imports", `Assoc (list |> List.map (fun (k, v) -> k, `String v)) ]
    |> Yojson.Safe.pretty_to_string ~std:true
  ;;

  let to_module (list : t) = List.map (fun (_, v) -> v) list

  let asset_rewrite path original_filename fingerprinted_filename =
    let basename = Filename.basename original_filename |> Filename.remove_extension in
    let directory = String.chop_prefix ~pre:path @@ Filename.dirname original_filename in
    let name_without_underscore =
      Option.get_or ~default:basename @@ String.chop_prefix ~pre:"_" basename
    in
    let module_name =
      match directory, String.equal basename "_index" with
      | Some dir, true -> Filename.basename dir
      | None, true -> "index"
      | Some dir, false -> dir ^ "/" ^ name_without_underscore
      | None, false -> name_without_underscore
    in
    module_name, "/" ^ fingerprinted_filename
  ;;

  let from_asset_map path asset_map =
    StringMap.fold
      (fun k v acc ->
         match Filename.extension k with
         | ".js" -> asset_rewrite path k v :: acc
         | _ -> acc)
      asset_map
      []
  ;;
end

let generate_importmap ?(path = "static/") (import_list : ImportMap.t) =
  let importmap =
    fingerprint ~path () |> ImportMap.from_asset_map path |> List.append import_list
  in
  { list = ImportMap.to_json importmap; modules = ImportMap.to_module importmap }
;;
```

### HTML Helpers
We now need functions to map the Assets and generate the HTML tags for both the **Import Map** and the **preloaded modules**:

```ocaml
let path ?(path = "static/") filename =
  match StringMap.find_opt (path ^ filename) A.asset_map with
  | Some fingerprinted_filename -> "/" ^ fingerprinted_filename
  | None -> "/" ^ path ^ filename
;;

module PureHTML = struct
  open Pure_html
  open HTML

  let importmap_tag =
    let importmap_list = script [ type_ "importmap" ] "%s" A.importmaps.list in
    let preloaded_modules =
      List.map
        (fun file -> link [ rel "modulepreload"; href "%s" file ])
        A.importmaps.modules
    in
    null @@ (importmap_list :: preloaded_modules)
  ;;
end

module String = struct
  let importmap_tag =
    let importmap_list =
      Format.sprintf {|<script type="importmap">%s<script>|} A.importmaps.list
    in
    let preloaded_modules =
      List.map
        (fun file -> Format.sprintf {|<link rel="modulepreload" href="%s">|} file)
        A.importmaps.modules
    in
    List.to_string ~sep:"\n" (fun s -> s) (importmap_list :: preloaded_modules)
  ;;
end
```
### Introducing **RealityAssets**
Clearly, this is a lot of reusable code for any Dream-based web project. So, I’ve bundled it into **[RealityAssets](https://github.com/Lomig/reality_assets)** — a library that:

* Handles **asset fingerprinting** automatically.
* **Generates Import Maps** for local and external JS dependencies.
* Adds **Stimulus** boilerplate.
* Includes a **CLI** for setup and integration.  

## Putting It to the Test

Now that **RealityAssets** is set up, let’s use it.

1. Update the **CSS import** to use the fingerprinted path.
2. Initialize the **Import Map with Stimulus**.

```ocaml
open Pure_html
open HTML
module Assets = Client.Assets

let body_classes =
  "grid h-screen overflow-hidden grid-cols-[auto_minmax(0,1fr)] \
   grid-rows-[minmax(0,1fr)_auto] bg-gray-900 text-white"
;;

let layout ?(page_title = "QuestComplete") data =
  html
    [ lang "fr" ]
    [ head
        []
        [ meta [ charset "UTF-8" ]
        ; meta [ name "viewport"; content "width=device-width, initial-scale=1.0" ]
        ; title [] "%s" page_title
        ; link [ rel "stylesheet"; href "%s" (Assets.path "application.css") ]
        ; Assets.PureHTML.importmap_tag
        ; Assets.PureHTML.js_entrypoint_tag
        ]
    ; body
        [ class_ "%s" body_classes ]
        [ header [ class_ "min-h-0 min-w-0" ] []
        ; main [ class_ "min-h-0 min-w-0 overflow-y-auto" ] data
        ; footer [ class_ "min-h-0 min-w-0" ] []
        ]
    ]
;;
```
{: file="lib/server/views/_layouts/main.ml" }

### Vanilla JS
Next, let’s try some Vanilla **JavaScript**:

```js
// Previous content that loads Stimulus Controllers

console.log("Hello from application.js")
```
{: file="lib/client/javascript/application.js" }

### Stimulus JS
And now, some **Stimulus JS**:

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
    connect() {
        console.log('Hello, Stimulus!', this.element)
    }
}
```
{: file="lib/client/javascript/controllers/hello_world_controller.js" }

```ocaml
let render game =
  let data = extract_data_from game in
  Layouts.Main.layout
    [ h1
        [ class_ "text-red-800 text-4xl"; string_attr "data-controller" "hello-world" ]
        [ txt "%s" data.title ]
    ; img [ src "%s" data.cover_url; alt "%s" data.title; class_ "w-1/4" ]
    ; ul
        []
        [ li [] [ txt "Genre : %s" data.genre ]
        ; li [] [ txt "Date de sortie : %s" data.release_date ]
        ; li [] [ txt "Plateforme : %s" data.platform ]
        ]
    ; br []
    ; br []
    ; p [ class_ "text-bold" ] [ txt "Complété le %s" data.completion_date; txt " !" ]
    ; a [ href "/games" ] [ txt "Back" ]
    ]
  |> to_string
;;
```
{: file="lib/server/views/games/show.ml" }

> [commit 531e7f4](https://github.com/Lomig/quest_complete/tree/531e7f4c4df3e71de3b41885c375b981a099e7e2)
{: .prompt-tip }

### Final Thoughts
With this setup, we now have:

✅ **Automatic asset fingerprinting**
✅ **Import Maps handling both local and external JS**
✅ **Seamless Stimulus integration**
✅ **PureHTML support for easier templating**

One small issue remains: `string_attr` isn’t the safest way to add Stimulus attributes. I’ll propose a **pull request** to add Stimulus tags to `Dream_html`, similar to its existing **HTMX module**. If it’s not accepted, I’ll create a separate module for myself.

And that’s it! 🚀

{% include image name="hello_world_with_stimulus.png" caption="One of our pages with Vanilla and Stimulus JS 'Hello World'" %}

## Next Steps
As always, I have plenty of ideas, and I rarely end up following my own roadmap—but that’s half the fun, right? That said, there are still a few critical pieces to tackle before I can call this a functional proto-framework:

* **WebSockets** – Real-time functionality is essential for many modern web applications, and I’ll need to explore the best way to integrate it into my setup.
* **Authentication** – WebAuthn is already in place, but there are still improvements to be made for a seamless authentication flow.
* **Database Migrations** – Managing schema changes effectively is a must-have for any serious web project.

Once these are in place, I’ll have a strong foundation to build on. Whether this eventually turns into a full-fledged framework or just a personal toolkit remains to be seen, but one thing is certain—I’m enjoying the process!

Stay tuned for the next post, and as always, feel free to reach out if you have any thoughts, questions, or suggestions.
