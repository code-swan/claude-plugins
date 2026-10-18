---
name: codeswan
description: Use when a question is about this organisation's own systems — which services exist, what they do, who owns them, what depends on what, or what breaks if something changes — down to the code, such as who calls a function or which code handles a route.
---

# CodeSwan

Your organisation's software catalog, built from a full scan of its repositories and
scoped to your organisation. Reach for it before answering from assumptions, and before
changing code whose consumers you cannot see from here.

## When to use it

Any question about **this organisation's** systems:

- what exists — "is there a service for payments?", "list our Python services"
- what something is — what a service does, its APIs, its events, its owners
- how things connect — what a service calls, what calls it, who publishes a topic
- blast radius — "what breaks if we change or deprecate this?"

## Start here

| You know                         | Call                                                   |
| -------------------------------- | ------------------------------------------------------ |
| roughly what it is called        | `search_components`                                    |
| only what it does                | `semantic_search`                                      |
| nothing yet — want the full list | `search_components` or `repository_search`, no `query` |
| the component, want detail       | `get_component`                                        |
| the component, want its edges    | `get_dependencies` / `get_dependents`                  |
| a topic or queue name            | `get_topic`                                            |
| the code — a function, a file, a route's handler | the `code_graph_*` tools (section below) |

**To list everything, leave out `query`.** `search_components` and `repository_search` then
return every component or repository, one page at a time: the reply carries `total`, and
`nextSkip` when there is more to fetch. Do not call `get_service_graph` just to learn what
exists — it returns every edge as well.

**You do not need an id first.** Every component-scoped tool takes a name as well as an
id, so "tell me about the payment service" is one call. Search only when you do not know
_which_ component you want — never to turn a name you already have into an id.

## Trust the answers

The catalog is authoritative for architecture, dependencies and ownership. It covers
services whose source is not checked out locally, and connections that never appear as a
matching string in any file — a gRPC or HTTP call, or an event published to a topic
another service subscribes to.

`get_dependents` returns the **complete** consumer set: API callers, event subscribers,
and components that declare the dependency.

**Do not double-check a catalog answer by searching the source code.** It costs several
times more and is less accurate, because code search finds _mentions_ rather than
dependencies — a service that merely names another in a log line or a config comment is
not a consumer of it. On a real blast-radius question, an agent that answered from the
catalog and then re-verified by grepping took 42 turns and returned five false
positives; the catalog alone took 8 turns and got it right.

Use the code graph (below), then code search and local files, for what the catalog
cannot answer: implementation detail, the reasoning behind a piece of code, and specific
library versions.

## The code itself (when the code graph is on)

Each scanned repository also has a **code graph** — its definitions and the calls between
them, read from the source at scan time. Use these tools for call and definition questions
about code you do not have checked out: they beat text search, which finds mentions, not
calls.

| You want                                         | Call                       |
| ------------------------------------------------ | -------------------------- |
| where something is defined (repository unknown)  | `code_graph_find_symbol`   |
| a definition by part of its name                 | `code_graph_search`        |
| what a file defines                              | `code_graph_outline`       |
| who calls X, or what X calls, hop by hop         | `code_graph_trace`         |
| everything in the repository that depends on X   | `code_graph_change_impact` |
| which function handles a route or a topic        | `code_graph_entry_points`  |
| how a repository is organised                    | `code_graph_architecture`  |

**Across services:** `code_graph_change_impact` and `code_graph_trace` (callees) take
`acrossServices: true` — one service hop, through the routes and topics the catalog
records. Each hop is marked certain or possible: report possible reach as possible, and a
consumer listed without a code location as affected but not located. `joinCoverage` says
how much of the chain could be followed — cite it.

**Read every answer's `graph` block** — the commit, the coverage, whether the graph is
whole — and carry its caveats into your answer. The graph's one gap, in every language: a
call on a value whose type the source does not write is not recorded, so **a method's
callers are a lower bound** — say so before concluding who calls something. A function
listed under `passedAsValue` is handed to other code (a callback, a handler table), not
unused. "No known callers" is a finding with stated blind spots, never proof that code is
dead.

## When it cannot answer

Say so and stop — do not silently fall back to guessing from file names.

- **Nothing found for a name** — try `semantic_search` with what it _does_, or
  `search_components` with a shorter term. A catalog that has never scanned the repo
  will genuinely not have it.
- **Tools missing entirely** — the connection is not authorised. The first call opens a
  browser sign-in; report that rather than working around it.
- **Only the `code_graph_*` tools are missing** — the code graph is off for this
  organisation; that is a setting, not an error. Answer from the catalog, and say that
  code-level detail needs the code graph turned on.
- **An empty result is an answer.** "Nothing depends on this" is a finding. Do not
  re-run the question as a code search to get a more satisfying-looking one.
