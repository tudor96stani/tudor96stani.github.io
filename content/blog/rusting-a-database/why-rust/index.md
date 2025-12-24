+++
title = "Why rust"
date = 2025-12-07T10:00:00+01:00
description = "Why I chose Rust for the rewrite of this project"
+++

Since I decided to rewrite everything from scratch, I also decided to pick a different language for the new version.

While I don’t think Java was the wrong choice (nor do I regret it), I felt the need to re-write the project in something that is closer to a real database engine. Most of the open source code I was consulting (PostgreSQL, SQLite) was C++, but for my entire professional career I’ve been working with managed, garbage-collected, runtime-based languages, quite far away in my perspective from a systems language. I did some C/C++ in college, but going from that to being able to write an entire project of this scale with it would have meant putting the database on hold for quite a while.

Then I started looking into Rust, which felt like a good middle ground between what I knew and where I wanted to get. Manually allocating memory without the fear of a `SEGFAULT` sounded like a good segue into this type of programming. Knowing that the borrow checker sits beside me and does not allow me to make blatant mistakes sounded like a nice to have safety net.

After a crash course of going through the official [Rust book](https://rust-book.cs.brown.edu/), doing the [Rustlings exercises](https://rustlings.rust-lang.org/), and several small test apps, I decided the best way to actually improve my Rust skills was to get to work and hope I'll pick up stuff along the way.

I’m putting all of this out here partly to keep track of what I’m doing, and partly because I’m sure someone out there will know better and point things out - whether that’s spotting issues I missed or suggesting better ways of implementing things.