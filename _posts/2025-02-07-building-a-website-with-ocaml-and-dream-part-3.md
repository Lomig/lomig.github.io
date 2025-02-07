---
layout: post
title: Building a Website with OCaml and Dream – Part 3
description: Keys, Tokens... Web Apps need secrets.
tags: OCaml Dream Webdev
category: Functional Programming
series: Building a Website with OCaml and Dream
series_description: Learn how to build a modular website using OCaml and Dream.
date: 2025-02-07 20:10 +0100
---
## Series
{% series %}

## Introduction
The next logical step in our journey is introducing some form of record storage in a database. If I were to use SQLite, everything would work out-of-the-box. However, despite the growing trend of using SQLite in production, I remain committed to PostgreSQL. This means I need to manage credentials securely without exposing secrets on GitHub.

Additionally, databases aren’t the only scenario where we need to store and expose internally static data for external sources.

## Choosing the Right Approach
There are three main ways to propagate secrets in a web application:

1. Secrets as Files on the Filesystem
  * Common in Kubernetes, where pods fetch secrets from a vault and store them as local files.
  * Offers no protection against accidental commits exposing secrets in Git.
  * Does not prevent unauthorized access to secrets on disk.
  * Difficult to share secrets securely among team members.
2. Secrets as Environment Variables
  * Setting secrets via environment variables (`DATABASE=some_url dune exec my_website`) is not user-friendly for local development, especially with multiple secrets.
  * Scripts can help, but there’s no standard approach for managing secrets in development.
  * A `.env` file (ignored by Git) can store local secrets and load them into environment variables at runtime.
  * Sharing secrets remains a challenge (password vaults, private gists, unsecured Slack messages?).
  * Managing versioned secrets is painful.
  * Can pollute the environment namespace and expose secrets unintentionally.
3. Secrets as an Encrypted HashMap
  * Secrets are pushed and versioned alongside the code in Git.
  * Requires one of the previous two methods to retrieve the encryption key.
  * Needs a mechanism to differentiate environments (dev, staging, prod, etc.).

Interestingly, Rails has built-in secret management, but in most projects I’ve seen, the .env approach is still the norm. Let’s explore that approach!

> Because of the way OCaml programs are started, any Env Variable set from within the App does not leak externally
{: .prompt-info }


## The `.env` File
A .env file is a simple text file (**ignored by Git**) containing key-value pairs:
```text
# This is a comment
KEY=VALUE
```
{: file=".env" }

Since we need to parse this text format, it’s time to use Regular Expressions.

## Regular Expressions in OCaml
The OCaml standard library’s regex capabilities are said to be outdated, so I opted for [Re](https://github.com/ocaml/ocaml-re). It offers familiar regex syntax (Perl, POSIX, etc.) while also providing a more expressive API in pure OCaml.

To use Regular Expressions in Ocaml, we typically need those steps:
1. Define the expression
2. Compile it
3. Use the compiled expression to match a string

## Matching Environment Variables

### Defining the Regular Expression
We need a regex that matches:
* The start of a line
* An alphanumeric key (possibly including _)
* The `=` character
* A value consisting of at least one character

Additionally, we want to capture both the key and the value separately.

A `PCRE` equivalent would be:
```perl
/^((_|[[:alnum:]])+)=(.+)$/
```

In `Re`, the implementation is more structured and readable:

```ocaml
open Containers

let regexp =
  let open Re in
  seq
    [ start
    ; group @@ rep1 (alt [ alnum; char '_' ])
    ; char '='
    ; group @@ rep1 any
    ; stop
    ]
  |> compile
;;
```

> The `let open Re in` statement allows us to use `Re` functions without explicit module prefixes, improving readability.
{: .prompt-info }

### Using the Regular Expression

We need to extract key-value pairs from each line. In OCaml, the `Option` monad is used instead of `nil`/`null`, as those values are impossible – it also allows us to avoid exceptions.

A basic approach:
```ocaml
(* Previous code *)

let to_key_value_pairs line =
  match Re.exec_opt regexp line with
  | None -> None
  | Some matches -> Some (Re.Group.get matches 1, Re.Group.get matches 2)
;;
```

A more idiomatic approach using `Option.map`:
```ocaml
(* Previous code *)

let to_key_value_pairs line =
  Re.exec_opt regexp line
  |> Option.map (fun matches -> Re.Group.get matches 1, Re.Group.get matches 2)
;;
```

## Loading Environment Variables

Now, let’s implement:
1. A function to process lines from the `.env` file, extracting key-value pairs and adding them to the environment.

2. A function to load the `.env` file if it exists and process its contents.

```ocaml
open Containers
open Server

let regexp =
  let open Re in
  seq
    [ start
    ; group @@ rep1 (alt [ alnum; char '_' ])
    ; char '='; group @@ rep1 any
    ; stop
    ]
  |> compile
;;

let to_key_value_pairs line =
  Re.exec_opt regexp line
  |> Option.map (fun matches -> Re.Group.get matches 1, Re.Group.get matches 2)
;;

let add_env_variables list =
  let rec aux = function
    | [] -> ()
    | None :: rest -> aux rest
    | Some (key, value) :: rest ->
      (match Sys.getenv_opt key with
       | Some _ -> aux rest
       | None -> Unix.putenv key value);
      aux rest
  in
  aux list
;;

let load_env () =
  match Sys.file_exists ".env" with
  | true ->
    IO.with_in ".env" IO.read_lines_l |> List.map to_key_value_pairs |> add_env_variables
  | false -> print_endline "No .env file to parse"
;;

let () =
  load_env;
  Dream.run
  @@ Dream.logger
  @@ Dream.router Router.routes
;;
```
{: file="bin/main.ml" }

## Reusability
Obviously, this functionality can be extracted into a reusable library. The final implementation is available [here](https://github.com/Lomig/simple_dotenv)!

## Next steps for our little Website
In the future, I might transition to a dedicated secret management system instead of relying on environment variables. However, in the next blog post, we'll shift our focus to databases—exploring how to integrate and manage them effectively!
