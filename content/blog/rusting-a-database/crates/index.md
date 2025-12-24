+++
title = "Crates"
date = 2025-12-07T13:00:00+01:00
description = "Organizing a project of this scale is already difficult - doing it in a new language, where you are not familiar with the idiomatic ways of structuring the project makes it even trickier"
+++

One of the things I have found relatively weird in Rust was the way you structure a (larger) project - you have crates, packages, workspaces? Coming from the .NET world with only solutions and projects, it was not super clear from the get-go how to organize this project. 

As with many software projects, the main goals were:
- clear separation between components
- clarity to which component depends on which
- some way of testing each component individually
- (and many more I'm sure)

The way I see the database project is split into these large chunks:
- a storage engine
- a query execution engine 
- a query planner and optimization engine (optionally part of the query engine)
- a front end (parser, binder, etc)
- cross cutting elements (transactions, locks, etc)
- a server component
- a binary to bind them all together 
- extra tools: client apps, drivers, etc

To try to achieve this, I've defined one workspace with the following components:
```
/trdb // main binary crate for the server. 
/crates
	/storage
		/page // lib 
		/buffer // lib 
		...
		/storage-api // lib - access point into the storage engine
	/execution
		...
		/execution-api // lib - access point into the query exec engine
	/plan
		...
		/plan-api // lib - access point into the plan generator engine
	/front-end
		/parser // lib crate
		/binder // lib crate
		...
		/front-end-api // lib - access point into the front end
	...
	/tools
		/cli-client // binary for the CLI client of the database
		...
```

Except for the main `trdb` binary crate which is stored at the root level, everything is stored under the `crates` folder, with subfolders based on the specific component. The `trdb` crate might also act as the server implementation since it has a library crate inside as well, I have not decided yet. 
The only crates that have been actually created so far are the ones under the `storage engine`, so the others might get renamed, reorganized, split or merged. 

But the main idea was to separate each major component of the engine into its own folder, where it can be broken down into separate crates if needed, and each component should expose an `-api` crate that will be the only point of access to that module.

The reason for breaking them apart into separate crates was mostly from an organizational point of view - I don't particularly like folders with a lot of files/subfolders underneath. I'll have to see if this split was too granular (or maybe too coarse, though I doubt that). As of right now, only part of the `page` crate is implemented, so only time will tell.