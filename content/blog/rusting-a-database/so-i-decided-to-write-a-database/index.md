+++
title = "So I decided to write a database system"
date = 2025-12-07T09:00:00+01:00
description = "First post, going over the reason behind this blog and giving some context."
+++

About a year ago I started a personal project to implement a relational database system from scratch, mostly as a result of (1) my interest in the field and (2) a desire to better understand how things work under the hood.

Professionally, I’ve been working for the past 7.5 years as a .NET backend engineer. A lot of my recent tasks involved attempting to improve the response time of a SQL Server database, which led me to spend quite some time looking at queries, execution plans, and schema design, trying to find the culprit behind the performance issue. I kept reading articles and blogs about random operators and how they work to try and grasp what was happening to my query.

The idea of writing some sort of database & query engine kept bouncing around in my head more like a conceptual thing - _I wonder how difficult it would be to implement a JOIN_ - until one day last fall when I was telling a friend about this over FaceTime, and we decided to make a go of it. We chose Java over something like C++ mostly because (a) it was a language we are both familiar with, C# being quite similar in many aspects, and (b) writing a database engine is hard enough - we wanted to focus on the design and functionality, not on also getting familiar with a low-level language at the same time.

After one year of active development, I started feeling a bit stuck. Some of the design choices I made turned out not to be the best, which led me to create several tasks tagged with `refactor` and `XXL`. Looking at the impact these refactors would bring, it felt easier to cut my losses, learn from my mistakes, and re-work everything from scratch rather than make the changes in place.

That decision also meant choosing a new direction for the implementation itself - not just a redesign, but a complete rewrite. I still didn't particularly want to go the C++ route, but I did want something closer to how actual database systems are implemented, so I chose to learn Rust for this new version.

Since I’m redoing the whole thing anyway, I figured I might as well make use of everything I learned the first time around. One of the lessons was that I didn’t do a great job documenting my thought process or why I made certain choices. This time I want to change that, and having a place to write things down from the very beginning might help.

This blog will follow my journey of learning Rust by re-implementing my initial database server. I’ll cover design choices and approaches I’ve taken for the engine, along with Rust-specific things I’ve used to implement them (or that I find cool). I know I’ll make plenty of mistakes - both Rust-specific and DB engine ones - but I guess that’s the whole point: I’ve been programming for long enough to know that you rarely get the right solution on the first attempt. I don’t regret any of the wrong choices from the Java project - by having made them, I’ve learned a lot more. Same goes for the _way_ I’ve implemented things in the few hundred lines of Rust code I’ve already written for this - I’m sure many of them will come back and bite me a few months from now, but that is part of the process I’ve learned to accept and actually enjoy.

I use various resources when designing the system: lectures I find on YouTube from various universities, several books, documentation, and (very little) open source code like PostgreSQL or SQLite. Many of the design choices are influenced by SQL Server (through observation & Microsoft’s public documentation) mostly because it’s the system I’m most familiar with and enjoy the most. However, I try to use these resources only to guide me in the right direction and have the actual implementation be as much my own as possible.

This database project will never see the light of day as an actual product to be used in real life - I don’t see the point in trying to create a competitor for existing solutions with decades of experience on the market (especially as a side project done by one person after work). It’s designed for my own growth and passion.

I do welcome any kind of feedback from anyone reading this - whether I’ve made a stupid choice regarding the design/behavior of the database engine, or if I’ve written some junk piece of Rust code that can be improved. Appreciation of anything that looks right is also more than welcome :D

The repo with the Rust code can be found [here](https://github.com/tudor96stani/trdb). `cargo` generated docs can be found [here](https://tudor96stani.github.io/trdb/trdb/index.html)