+++
title = "Server, crates and docs"
date = 2026-04-29T19:00:00+02:00
slug = "server-crates-docs"
description = "Some updates on the initial server implementation, crate reorganization and docs"
draft = false
+++

As I've had a pretty busy period, I haven't had the chance to post anything new here, and I only managed to get a few things done on the actual coding side - I will cover them here.

# Server - client
As the buffer implementation was thread safe (I hope!) from the beginning, I decided to avoid going the same route of implementing the whole thing as a single executable, only to have to split it into client and server later on.
Therefore, I created a second executable for the database client (a CLI app) and implemented a TCP server on the main binary.

## Server
### High level
The main idea behind the server implementation is the following: the app will use `tokio` threads to asynchronously communicate via the socket with the client - this allows us to yield threads back to the pool as we are waiting for things like network.
The more problematic area is actual query execution:
- we do not want the client thread that accepted the connection to be blocked for the entire duration of the query execution
- we also do not want to sprinkle async through the entire codebase, but unless a long-running operation IS async (e.g. reading from disk), it will block the thread
- we do not want to give full control of the scheduling to the `tokio` runtime
  
To accommodate this, the following implementation was chosen:
- only the handling of client requests is done on the async thread pool from `tokio`
- actual query execution is done on a worker, blocking thread
- we will control the number of concurrent active queries (active meaning being executed) through a fixed size worker thread pool
- we will, however, accept as many client connections/request as possible

To achieve this, actual query execution is moved to the blocking thread pool from `tokio`, operation which is guarded by a semaphore.

The overall flow is the following:
1. Server boots up, reads config file (`trdb.toml`) to determine data directory and buffer size. Also starts a TCP listener
2. In a loop, calls `accept` on the listener and hands off the interaction with the client on a separate `tokio` thread (async) (actually does a `tokio::select!` with an additional branch to listen for a shutdown request as well)
3. The client handler starts a loop that reads (async) from the socket and prepares the query request - the client can send multiple requests on the same socket
4. It attempts to enter the semaphore - if available, acquires a permit, otherwise it waits (done on a `tokio::select!` as well, listening for a possible shutdown request)
5. Once it passes the semaphore, it triggers a `spawn_blocking` thread to actually execute the query and `awaits` the result
6. After it receives the result, it streams it back to the client.
### What works right now
While the overall logic of the server is implemented, it only does some very basic things for now:
- creates a page on startup, if it does not exist already on disk
- client handlers expect a single integer to be received
- it generates a row with that value repeated 100 times, inserts it in that given page, then reads it and sends it back
- a background worker is listening for shutdown requests, allowing us to add shutdown logic in the future (flushing buffer, etc)
- added logging as well 

### What is next
Besides the obvious part of implementing a server that does more than accept an integer (and cleaning up the code which is all dumped in `main.rs`), some next steps:
- define a TCP protocol for the client-server communication, to allow both inbound requests of queries and outbound responses with the result sets
- extract server related logic elsewhere
- make things more configurable (e.g. port)
- (nice to have) find a way to stream back the results as they arrive (so if the engine produces a total of 100k rows, start sending them 1k by 1k as the engine is still appending new ones to the result set, instead of waiting for it to return all of them at once)
- (nice to have) define a more custom scheduling policy for the worker threads - something towards a cooperative approach where the blocking worker threads can be returned to the pool while they are waiting for I/O.

## Client
The client app (`trdbcmd`) is very simple as of right now. It only connects to the server, waits for user input of a number, sends it to the server and prints back the result. Obviously done only for testing purposes, will be expanded.

## Next steps
While many of the next steps are important, the current implementation allows me to test the things I am working on, so they have been pushed down in priority a bit.

  
# Crates
The other larger thing I have done was to reorganize the crates a bit.

Initial choice for project structure was to separate each major component (storage engine, query engine, etc.) into separate small crates and define an `*-api` crate (e.g. `storage-api`) that acted as the gateway into that component. This might be too complex and overengineered.

Because of this, I decided to simplify the architecture by doing a single crate/component. Therefore, I grouped all storage related crates into a single one (called `storage`), adjusted the exposed items, error types, etc. This seems to have made things a bit cleaner.

## Docs
These two last items are more maintenance: the first one is around docs:
I started working on some actual technical docs, which are already behind schedule, as it's often the case with documentation. Will keep updating it as I go though.

Additionally, I've decided to add my own notes to the repo - the ones I usually take during development (so they might be outdated or raw) - I think they be interesting from a timeline perspective. They can be found in `implementation_notes.md`. They all have an ID (timestamp) and are pretty much pulled from my notebook unaltered. Since I am also placing them in my main Obsidian vault, they have some links that are not relevant in here.

I've also added a `decision_log.md` note in which i extract (from the actual implementation notes) any relevant decision I might make around the design.

All of them can be found in the repo, under the [/docs](https://github.com/tudor96stani/trdb/tree/main/docs) folder.


## Backlog and board
I've also started defining the backlog of things that needs to be done - for the Java app I used GitHub's Project + Issues, but I did not particularly like it. For this one, I've created an Azure DevOps board (private) in which I transferred the backlog, organized it a bit.
I still need to migrate the features from the Java board, right now I only have work items to reach feature parity.