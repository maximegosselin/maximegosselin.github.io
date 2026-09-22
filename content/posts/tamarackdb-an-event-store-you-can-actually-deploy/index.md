---
date: 2026-09-21
title: "TamarackDB: An Event Store You Can Actually Deploy"
description: 'A standalone, DCB-compliant event store. Written in Go, backed by SQLite, reachable over HTTP.'
tags: ['Event Sourcing', 'DCB']
---

Back in February I wrote about [implementing a DCB-compliant event store in SQLite](https://maximegosselin.com/posts/implementing-a-dcb-compliant-event-store-in-sqlite/): a four-column table, a few indexes, and SQLite's own file lock doing the heavy lifting on consistency. It worked, but it had one obvious limit: SQLite is embedded, so the technique only helped the one process holding the file open.

[TamarackDB](https://github.com/tamarackdb/tamarackdb) is that idea turned into something you can actually run. It's a standalone event store: written in Go, reachable over HTTP, backed by SQLite. It follows the [DCB specification](https://dcb.events/specification/), but works just as well for plain old aggregates. Not a library you import into your app. A small service that sits next to your apps and holds their events.

## Why SQLite

The instinct, once you decide to build a server instead of a library, is to reach for Postgres or MySQL. Or, if you're feeling like a god that day, to invent your own proprietary storage format. I went the other way.

TamarackDB targets applications with modest throughput and few concurrent writers, not internet-scale traffic. At that scope, several million rows with a `name + value` btree index is well inside what SQLite handles comfortably. Beyond volume, three things made SQLite the better fit, not just an acceptable one:

- **No external dependency.** No separate database process to run, patch, and monitor next to the event store itself. State stays where the Go process already is.
- **Single-writer by design.** TamarackDB only ever has one process writing, by design. SQLite's own single-writer model matches that instead of fighting it.
- **A plain file.** The database is a `.sqlite` file you can open with the ordinary `sqlite3` CLI, inspect, and back up with SQLite's own tools. No proprietary format standing between you and your events.

WAL mode is what makes this pleasant to operate: reads keep going while a write is in flight, so the single-writer constraint never means single-user.

## HTTP: taking SQLite out of the process

An embedded database is only as available as the process embedding it. The moment you want two applications, maybe in two different languages, sharing the same stream of events, "just open the file" stops being an option.

So TamarackDB wraps that SQLite file in a small HTTP API. `QUERY /events` (yes, the [HTTP `QUERY` method](https://www.rfc-editor.org/info/rfc10008/), not `GET`, because a Decision Model's query can be too large or too nested for a query string) to read. `POST /write` to append. Responses stream back as [NDJSON](https://github.com/ndjson/ndjson-spec), one event per line, so a page of a few thousand events never has to be buffered whole on either end, and a dropped connection loses nothing already received.

That's the whole trick. SQLite stays exactly as simple as it always was. Go just stands between it and the network, serializing every write through one connection. Any application that can speak HTTP and JSON now has a remote, shared event store. No SQLite driver required on the client side at all.

## Persisting events and projections atomically

Most event-sourced systems end up maintaining two things: the event log, and one or more read-model projections built from it. Usually in two different stores, updated at different times, which means there's always a window where they disagree.

Eventual consistency between the two is usually presented as inherent to event sourcing, something you accept rather than choose. I don't see it that way. Plenty of people want to try event sourcing without taking on eventual consistency as part of the deal, and I think that's a legitimate choice, not something to be talked out of.

TamarackDB gives you the option to close that window. Alongside events, `/write` can carry documents: versioned, current-state-only projections, identified by type and id. A document's identity and version live in the same SQLite file as the events, in the same transaction. The version bump and the event that caused it either both happen or neither does.

If you don't need this, ignore it. An application that already maintains its own projections elsewhere never has to touch the documents endpoints.

## Go, because building it should be boring

None of the above needed Go specifically. What it gave me was a build I don't have to think about: one static binary, no C toolchain to install, a Docker image that's just the binary dropped into Alpine.

`make build` gives you the server plus three small support tools, and that's the entire build surface. For a project meant to be deployed next to an application and forgotten about, that's exactly the amount of ceremony I wanted.

## Where it sits among event stores

Most event stores fall into one of two categories. A library that runs inside your application process: my own [Backslash](https://github.com/backslashphp/backslash) for PHP, [MartenDB](https://martendb.io/) for .NET, [Fact](https://github.com/evntd/fact) for Elixir. Or a full platform built for clustering and internet-scale throughput, like EventStoreDB with its own storage engine and multi-node replication, or AxonServer, the JVM-based backbone of the Axon ecosystem.

TamarackDB is neither. It belongs to a newer wave of standalone, DCB-native stores: [UmaDB](https://github.com/umadb-io/umadb) (gRPC, its own Rust storage engine), [Tephra](https://docs.rs/tephra) (TCP, a segment-based log), [EventsourcingDB](https://www.eventsourcingdb.io/) (HTTP, its own storage engine). All real processes you deploy and point applications at, not a library, and all single-instance rather than a cluster. TamarackDB's own angle in that group is the storage engine: plain SQLite instead of something purpose-built, because for the scope it targets, a well-understood, inspectable file beats a faster but opaque one. It's sized for the kind of team that would otherwise reach for a library and accept the in-process limits that come with it, but wants events reachable from more than one application, in more than one language, without adopting a platform built for a scale they don't have.

## Why this needs to exist

I think event sourcing has an accessibility problem more than a technology problem. The pattern itself is simple. Most of the tooling around it isn't, and that gap is what keeps people from trying it on a project where it would genuinely help.

TamarackDB is still young but promising. It's proof that an event store doesn't need to be complicated to be real. SQLite, Go, HTTP, one file you can open with a command-line tool. If more of our tools stayed that plain, I think a lot more teams would actually try event sourcing instead of just reading about it.

The code is at [github.com/tamarackdb/tamarackdb](https://github.com/tamarackdb/tamarackdb).