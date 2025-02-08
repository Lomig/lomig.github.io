---
layout: post
title: Building a Website with OCaml and Dream – Part 4
description: Storing data. Retrieving data. Serving Data.
tags: OCaml Dream Webdev
category: Functional Programming
series: Building a Website with OCaml and Dream
series_description: Learn how to build a modular website using OCaml and Dream.
mermaid: true
date: 2025-02-08 22:12 +0100
---
## Series
{% series %}

## Introduction

In this article, we'll continue building our game collection website using OCaml and PostgreSQL. Our goal is to create a platform where users can track completed games, including completion dates and reviews. This part focuses on setting up database connectivity and implementing our first database-driven features.

## Database Schema

Let's start with our database schema. We're using a single `games` table to store our game collection (we'll think about migrations another day):

```mermaid
erDiagram
    games {
        ULID id "DEFAULT gen_monotonic_ulid()"
        VARCHAR title "NOT NULL, CHECK (LENGTH(TRIM(title)) > 0)"
        ENUM platform "NOT NULL"
        VARCHAR genre "NOT NULL"
        DATE release_date "NOT NULL"
        INTEGER igdb_id "NOT NULL"
        VARCHAR cover_url "NOT NULL"
        DATE completion_date "NOT NULL, CHECK (completion_date >= release_date)"
        SMALLINT rating "CHECK (rating BETWEEN 0 AND 10)"
        TEXT review
        TIMESTAMPTZ created_at "DEFAULT CURRENT_TIMESTAMP"
        TIMESTAMPTZ updated_at "DEFAULT CURRENT_TIMESTAMP"
    }
```

For testing purposes, we'll insert a single record:

```sql
INSERT INTO games
  (title, platform, genre, release_Date, igdb_id, cover_url, completion_date)
VALUES
  ('Final Fantasy VII', 'PC', 'RPG', '1998-06-24', 207021, 'https://images.igdb.com/igdb/image/upload/t_cover_big/co4xgb.jpg', '1998-08-01');
```

## Setting Up Database Connectivity

We'll use [Caqti](https://github.com/paurkedal/ocaml-caqti) as our PostgreSQL driver and integrate it with Dream's built-in SQL functionality.

### Environment Configuration

First, create a `.env` file to store your database connection string:

```
DATABASE_URL = postgres://my_user:my_password@postgres__url
```
{: file=".env" }

Add the required dependencies to your `quest_complete.opam.template`:

```
pin-depends: [
  [ "reality_tailwindcss.1.0.0" "git+https://github.com/Lomig/reality_tailwindcss.git#main" ]
  [ "simple_dotenv.1.0.0" "git+https://github.com/Lomig/simple_dotenv.git#main" ]
]
```
{: file="quest_complete.opam.template" }

Update your `bin/dune` file to include the necessary libraries:
```lisp
(executable
 (public_name quest_complete)
 (name main)
 (libraries dream simple_dotenv server))

; Tailwind Auto-generated Configuration
; [...]
```
{: file="bin/dune" }

### Database Connection Pool
Set up the Dream SQL pool in your `bin/main.ml`:

```ocaml
open Server

let () = Dotenv.load ()

let () =
  Dream.run
  @@ Dream.logger
  @@ Dream.sql_pool (Sys.getenv "DATABASE_URL")
  @@ Dream.router Router.routes
;;
```
{: file="bin/main.ml" }

> [commit 1be97ca](https://github.com/Lomig/quest_complete/tree/1be97ca5d215c5dea4f08bca41b98d00059ed3f2)
{: .prompt-tip }

## Understanding Caqti Queries

Caqti uses a type-safe approach to database queries. Let's break down how it works:

### Request Definition

A request is a function that will generate the actual raw SQL query. I'll use the name `query` for the variable as `Dream` already has a concept of request.

```ocaml
module T = Caqti_type

let query =
  let open Caqti_request.Infix in
  (T.unit ->! T.string) "SELECT title FROM games LIMIT 1"
;;
```

`(T.unit ->! T.string)` indicates:
* No input parameters (`T.unit`)
* Returns exactly one record (`->!` infix operator)
* Returns a string value (`T.string`)
* The type of the request will then be ``(unit, string, [ `One ]) Caqti_request.t``

Other operators:
* `->.` returns zero records.
* `->?` returns zero or one record.
* `->*` returns zero or more records.

For multiple parameters or return values, use tuples:
* `t2 string int` represents a tuple `(string, int)`.
* `t3 int string string` represents a 3-tuple `(int, string, string)`.
* `Caqti` supports tuples up to size 12.

Nullable SQL fields are represented as `option` types in OCaml.

### Executing a Query

> `Dream` and `Caqti` use `Lwt` to deal with asynchronicity. A DB response is a promise, and we must handle it properly.
{: .prompt-info }

A DB connection module executes the query. To that avail, DB Connection Modules implement:
* a `find` function to return a record
* a `find_opt` function to return an option of a record
* a `collect_list` function to return a list of records
* functions to deal with transactions


```ocaml
open Lwt.Syntax

let first_game_title (module Db : Caqti_lwt.CONNECTION) =
  let* record_and_error = Db.find query () in
  Caqti_lwt.or_fail record_and_error
;;
```

`find` will return a tuple of the expected response and an error, so after awaiting the resolve of the query promise, we create a new promise with `Caqti` to handle the potential error received.

### Testing in a Web Page

We can display this in a real webpage:

```ocaml
open Lwt.Syntax

module type DB = Caqti_lwt.CONNECTION

module T = Caqti_type

let query =
  let open Caqti_request.Infix in
  (T.unit ->! T.string) "SELECT title FROM games LIMIT 1"
;;

let first_game_title (module Db : DB) =
  let* record_and_error = Db.find query () in
  Caqti_lwt.or_fail record_and_error
;;

let homepage request =
  let* game_title = Dream.sql request first_game_title in
  Views.Pages.homepage game_title |> Dream.html
;;
```
{: file="lib/server/handlers/pages.ml" }

```ocaml
let homepage title =
  Templates.Layouts.Main.layout @@ Templates.Pages.Homepage.render title
;;
```
{: file="lib/server/views/pages.ml" }

```html
let render title =
<h1 class="text-red-400">My First Game</h1>
<p>
    <%s title %>
</p>

```
{: file="lib/server/templates/pages/homepage.eml.html" }

> [commit 43b360e](https://github.com/Lomig/quest_complete/tree/43b360e2098794ea94952f0f93b1017e5b1fa9ec)
{: .prompt-tip }

And we have data coming from our database!

{% include image name="hello_world_with_db.png" caption="Our homepage with the title of the first game inserted in DB" %}

## Domain
As mentioned at the start of this series, the goal is to build something close to the MVC pattern. However, models only hold a special place in the domain, so I opted to create a dedicated Domain folder to house all modules related to my business logic.

Within this `Domain`, I need a way to represent a game.

### Types

Since platform is stored as an Enum in the database, I don’t want to handle it as a raw string in my domain model. Instead, I define it explicitly:

```ocaml
type t =
  | PC
  | PS5
  | PS4
  | XboxXS
  | XboxOne
  | Switch
  | Mobile
  | Other

let of_string = function
  | "PC" -> PC
  | "PlayStation 5" -> PS5
  | "PlayStation 4" -> PS4
  | "Xbox Series X/S" -> XboxXS
  | "Xbox One" -> XboxOne
  | "Nintendo Switch" -> Switch
  | "Mobile" -> Mobile
  | _ -> Other
;;

let to_string = function
  | PC -> "PC"
  | PS5 -> "PlayStation 5"
  | PS4 -> "PlayStation 4"
  | XboxXS -> "Xbox Series X/S"
  | XboxOne -> "Xbox One"
  | Switch -> "Nintendo Switch"
  | Mobile -> "Mobile"
  | Other -> "Autre"
;;
```
{: file="lib/server/domain/platform.ml" }

Similarly, `release_date` and `completion_date` deserve better than simple strings, so I built a module to handle them properly. Below is the type signature; you can find the full implementation on GitHub if you're interested:

```ocaml
type day
type year

type month =
  | January
  | February
  | March
  | April
  | May
  | June
  | July
  | August
  | September
  | October
  | November
  | December

type t =
  { day : day
  ; month : month
  ; year : year
  }

type week_day =
  | Monday
  | Tuesday
  | Wednesday
  | Thursday
  | Friday
  | Saturday
  | Sunday

val string_of_week_day : week_day -> string
val int_of_week_day : week_day -> int
val week_day_of_int : int -> week_day
val string_of_month : month -> string
val int_of_month : month -> int
val month_of_int : int -> month
val day_of_the_week : t -> week_day
val to_string : t -> string
val to_short : t -> string
val to_long : t -> string
val to_iso : t -> string
val of_iso : string -> (t, string) result
val is_leap_year : year -> bool
val days_in_month : year -> month -> int
val days_in_year : year -> int
val zero : t
```
{: file="lib/server/domain/date.mli" }

Finally, my `Game` module is structured as follows:

```ocaml
type t =
  { id : string
  ; title : string
  ; platform : Platform.t
  ; genre : string
  ; release_date : Date.t
  ; igdb_id : int
  ; cover_url : string
  ; completion_date : Date.t
  ; rating : int option
  ; review : string option
  ; created_at : Ptime.t
  ; updated_at : Ptime.t
  }
```
{: file="lib/server/domain/game.ml" }

### Custom Caqti Type for the Domain
With this structured record, I no longer need to deal with raw tuples in Caqti. Additionally, I want Caqti to handle the encoding and decoding of `Date` and `Platform` seamlessly. Thankfully, the library allows us to define custom types:

```ocaml
module DB = struct
  include Caqti_type.Std
  include Caqti_request.Infix

  module type Connection = Caqti_lwt.CONNECTION

  let t =
    let encode game =
      Ok
        ( game.id, game.title, Platform.to_string game.platform, game.genre
        , Date.to_iso game.release_date, game.igdb_id, game.cover_url
        , Date.to_iso game.completion_date, game.rating, game.review
        , game.created_at, game.updated_at )
    in
    let decode
          ( id, title, platform, genre, release_date, igdb_id, cover_url
          , completion_date, rating, review, created_at, updated_at )
      =
      Ok
        { id; title; platform = Platform.of_string platform; genre
        ; release_date = Result.get_or ~default:Date.zero @@ Date.of_iso release_date
        ; igdb_id; cover_url
        ; completion_date = Result.get_or ~default:Date.zero @@ Date.of_iso completion_date
        ; rating; review; created_at; updated_at
        }
    in
    custom
      ~encode
      ~decode
      (t12
         string string string string
         string int string string
         (option int) (option string) ptime ptime) [@@ ocamlformat "disable"]
end
```
{: file="lib/server/domain/game.ml" }

The implementation is somewhat lengthy due to the number of fields and the necessary encoding/decoding logic, but overall, it's clear and straightforward.

That being said, this seems like a great opportunity to explore `ppx` to automate the Caqti type definitions. I might look into it in the future—though maybe I’m just overcomplicating things.

### Querying the Domain

To fetch a game by its `id`, the query structure follows our earlier test:


```ocaml
let find_by_id ~request id =
  let open DB in
  let open Lwt.Syntax in
  let query = (string ->? t) @@ "SELECT * FROM games WHERE id = ?" in
  let find_by_id' id =
    fun (module Db : Connection) ->
    let* game_option = Db.find_opt query id in
    Caqti_lwt.or_fail game_option
  in
  let* result = id |> find_by_id' |> Dream.sql request in
  Lwt.return result
;;

```
{: file="lib/server/domain/game.ml" }

And that's it! It is now up to a `Handler` to call this function.

## Showing a Game

To display a game, we need to add a route with a parameterized URL. This route will be handled by a new `Handler` function:

```ocaml
open Handlers

let routes =
  [ Dream.get "/static/**" (Dream.static "static")
  ; Dream.get "/" Pages.homepage
  ; Dream.get "/games/:id" Games.show
  ]
;;
```
{: file="lib/server/router.ml" }

Since handlers rely on domain logic, they should have access to the `Domain` modules. This is reflected in our `dune` configuration:
```lisp
(library
 (name handlers)
 (libraries dream domain views))
```
{: file="lib/server/handlers/dune" }

The `Games` handler module exposes a `show` function to display a single record. The game `id` is extracted from the request using `Dream.param`:

```ocaml
open Domain
open Lwt.Syntax

let show request =
  let id = Dream.param request "id" in
  let* game = Game.find_by_id ~request id in
  match game with
  | None -> Dream.empty `Not_Found
  | Some game -> game |> Views.Games.show |> Dream.html
;;
```
{: file="lib/server/handlers/games.ml" }

### Rendering the View
The view layer processes the retrieved game data and sends it to the template. It ensures that only relevant, formatted data is exposed to the template:

```ocaml
open Containers
open Domain

let show_data (game : Game.t) : Templates.Games.Show.data =
  { title = game.title
  ; release_date = Date.to_short game.release_date
  ; rating =
      (match game.rating with
       | None -> "N/A"
       | Some r -> string_of_int r)
  ; platform = Platform.to_string game.platform
  ; genre = game.genre
  ; cover_url = game.cover_url
  ; completion_date = Date.to_long game.completion_date
  ; review = Option.get_or ~default:"Pas encore de critique" game.review
  }
;;

let show game =
  Templates.Layouts.Main.layout @@ Templates.Games.Show.render @@ show_data game
;;
```
{: file="lib/server/views/games.ml" }

### A Basic (and Ugly) Template
The following template renders the game details. Right now, it’s quite basic (and not very pretty), but it gets the job done:

```html
type data =
{ title : string
; release_date : string
; rating : string
; platform : string
; genre : string
; cover_url : string
; completion_date : string
; review : string
}


let render data =
<h1 class="text-red-800 text-4xl">
    <%s data.title %>
</h1>
<img src="<%s data.cover_url %>" alt="<%s data.title %>" class="w-1/4">
<ul>
    <li>
        Genre : <%s data.genre %>
    </li>
    <li>
        Date de sortie : <%s data.release_date %>
    </li>
    <li>
        Plateforme : <%s data.platform %>
    </li>
</ul>
<br>
<br>
<p class="text-bold">Complété le <%s data.completion_date %> !
</p>
```
{: file="lib/server/templates/games/show.eml.html" }

> [commit e2e0e17](https://github.com/Lomig/quest_complete/tree/e2e0e17f84105cc2804e4fd0d8ed7d8dc0f6e2ca)
{: .prompt-tip }

## Next Steps

At this stage, we have everything needed to build a functional website — routes, data persistence, and basic rendering. However, there’s still a lot to refine:

* **CSS**: The template desperately needs styling.
* **Database Migrations**: We need a way to handle schema evolution.
* **JavaScript**: Being a professional Ruby dev, I’m leaning toward **Hotwire** over HTMX (spoiler alert).
* **PPX for DB Setup**: Investigating how to leverage **PPX** for reducing boilerplate.
* **WebSockets**: Exploring real-time capabilities.
* **Writing OCaml Instead of JavaScript**: Finding ways to avoid writing JS altogether.

There’s plenty to explore ahead! 🚀
