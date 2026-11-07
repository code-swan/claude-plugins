---
name: codeswan
description: Use when a question is about this organisation's own systems — which services exist, what they do, who owns them, what depends on what, or what breaks if something changes — down to the code, such as who calls a function or which code handles a route. A checkout shows ONE repository. For who uses or consumes something, what breaks if it changes, whether we already have something that does a job, and who owns or works on it, ask the CodeSwan catalog first, even with the code open — it covers every repository, and the calls and events no text search can see. An empty grep is not evidence that nothing exists.
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
| roughly what it is called        | `search` (`kinds: ["component"]` to stay on services)  |
| only what it does                | `search` — it ranks by meaning                         |
| nothing yet — want the full list | `search` or `repository_search`, no `query`            |
| the component, want detail       | `get_component` — its business `features` included; `includeDiagrams: true` adds diagram source |
| the component, want its edges    | `get_dependencies` / `get_dependents`                  |
| the full chain behind a request, or everything X eventually reaches | `get_dependencies` with `depth: 10`, once (`get_dependents` the same, the other way) |
| a topic, flag, feature or endpoint by name/key | `get_entity` with its `kind` (`topic`, `feature_flag`, `feature`, `api`) |
| the code — a function, a file, a route's handler | the `code_graph_*` tools (section below) |

**`search` finds every kind of thing** — components, APIs, data stores, topics and queues,
business features, feature flags — and `kinds` narrows it. Use it to FIND something; when you
already hold a name, key or topic, call the lookup tool for it directly (`get_component`, or
`get_entity` — which services consume a topic is `get_entity` with `kind: "topic"`, not
`get_dependents`).

**To list everything, leave out `query`.** `search` and `repository_search` then
return every component (or API, topic, flag … per `kinds`) or repository, one page at a time: the reply carries `total`, and
`nextSkip` when there is more to fetch. Do not call `get_service_graph` just to learn what
exists — it returns every edge as well.

**You do not need an id first.** Every component-scoped tool takes a name as well as an
id, so "tell me about the payment service" is one call. Search only when you do not know
_which_ component you want — never to turn a name you already have into an id.

## What to trust, and what to check

The catalog is the first stop for architecture, dependencies and ownership. It covers
services whose source is not checked out locally, and connections that never appear as a
matching string in any file — a gRPC or HTTP call, or an event published to a topic
another service subscribes to.

`get_dependents` lists API callers, event subscribers, and components that declare the
dependency.

**Do not rebuild a consumer list by searching the source code.** (This is about listing who
calls or depends on something. Planning a change or finding a bug is different — see the next
section.) It costs several times more
and is less accurate, because code search finds _mentions_ rather than dependencies — a
service that merely names another in a log line or a config comment is not a consumer of
it. On a real blast-radius question, an agent that answered from the catalog and then
re-verified by grepping took 42 turns and returned five false positives; the catalog
alone took 8 turns and got it right.

**A catalog hit narrows where to look; it is not proof that nobody else is affected,** and
a route on an edge is matched, not proven exact. For a rename, a contract change, or a thin
or empty result, verify before you narrow scope: open each consumer's `callSite` (the
`calls` in `get_dependents`) or ask the owning team. A value that lives in code — a status
or enum value, a constant — has no consumer list here: search the code of every repository
for it, or ask every team that may use it.

**The catalog knows structure, not behaviour.** It does not hold values inside code (status
and enum values, constants, formatters, config literals), the reasoning behind a piece of
code, or exact dependency versions. To find where a wrong value or behaviour comes from,
take the call chain from the catalog, then read the code at each hop — do not pick the
location from structure alone.

Use the code graph (below), then code search and local files, for what the catalog
cannot answer: implementation detail, the reasoning behind a piece of code, and specific
library versions.

## A full chain: one call, not hop by hop

Request chains in a real system often run past three hops. `get_dependencies` walks the
upstreams of the upstreams on the server: pass `depth: 10` once. The walk stops by itself when a
level adds nothing, so a high value costs nothing on a short chain, and you cannot miss a branch
the way you do by following hops yourself. `get_dependents` takes the same `depth` for the
reverse direction.

- Entries carry the `hop` they were reached at. For "what calls what over HTTP" keep the `api`
  entries and, past hop 1, only those whose `viaPath` is all `api`: an `event` entry is a
  subscription through a topic, not a call.
- `calls` are listed at hop 1 only. To see where a deeper call is made, ask for that service's
  own `get_dependencies`.
- If `truncated` is true the walk stopped at the cap: call again on the entries marked
  `moreUpstreams`. Never report a truncated walk as the whole chain. (`get_dependents` has no
  `truncated` flag: if its deepest level is still full, call it again from those entries.)

## Planning a change, or finding where a bug lives

"Which repositories must change?" and "where does this wrong value come from?" are not "who
calls X?" questions. The catalog gives you the call chain and the owners, not the answer.
Before you name any repository:

1. **Take the chain from the catalog** — as above.
2. **Search every repository for the literal itself** with `code_smart_search` (a checkout
   shows only one repository; this searches the default branch across the org's repositories)
   — the route path, the field name, the topic, the status value — not only the repositories on
   the chain. Contract and spec repositories, deploy and gateway config, and frontends mention a
   route, topic or field without calling or subscribing to it, so no catalog edge points at them.
3. **Open the code before you name a repository.** A service the chain points at is where to
   look, not proof the fault is there: a wrong value is often made in the service it calls.
4. **Do not read "nobody calls this route" off the catalog.** A route on an edge is matched, not
   proven exact, so the callers of a service may not show the route you are changing. Open each
   caller's `callSite` before you decide nothing else is affected.

A plan that names a repository you did not open, or stops at what the catalog listed, is the
usual way this goes wrong.

## The code itself (when the code graph is on)

Each scanned repository also has a **code graph** — its definitions and the calls between
them, read from the source at scan time. Use these tools for call and definition questions
about code you do not have checked out: they beat text search, which finds mentions, not
calls.

| You want                                         | Call                       |
| ------------------------------------------------ | -------------------------- |
| where something is defined (or by part of its name in one repository) | `code_graph_symbols` |
| who calls X, or what X calls, hop by hop         | `code_graph_trace`         |
| what changing X would touch                      | `code_graph_trace` with `mode: "impact"` — ask it directly |
| which function handles a route or a topic        | `code_graph_entry_points`  |
| what a file defines (`path`), how a repository is organised (no `path`) | `code_graph_structure` |

**Across services:** `code_graph_trace` (mode `impact`, or callees) takes
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

- **Nothing found for a name** — `search` again with what it _does_, or with a
  shorter term. A catalog that has never scanned the repo
  will genuinely not have it.
- **Tools missing entirely** — the connection is not authorised. The first call opens a
  browser sign-in; report that rather than working around it.
- **Only the `code_graph_*` tools are missing** — the code graph is off for this
  organisation; that is a setting, not an error. Answer from the catalog, and say that
  code-level detail needs the code graph turned on.
- **An empty result is an answer.** "Nothing depends on this" is a finding: report it
  together with what the catalog cannot see (above), and verify it as described there
  before you act on it. For a plain question, do not re-run it as a code search to get a
  more satisfying-looking one.
