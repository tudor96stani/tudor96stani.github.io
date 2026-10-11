+++
title = "Binary rows and AI"
date = 2026-10-11T13:00:00+01:00
slug = "binary-rows-and-ai"
description = "A quick update on the binary row format I am implementing and the AI-assisted workflow I added to the repo"
+++

Long time since an update here - I've been kept busy by various personal things, so involvement on this project has been lower than usual, but I should be getting more towards normal now. 

While I've done a few other things for the db engine, the binary row representation is the most relevant. 

# Binary rows
Basically as said in previous posts, a data page will store a row/record as a contiguous chunk of bytes. Since the rows are variable length, we need to have an encoding for them as well. 

At its core, the format is compact and pretty clearly organized. We start with a header of course, then we have all the integers (fixed size), then a null bitmap then the string section (which will be variable length). Well, the integer section will be variable as well, depending on the number of integers, but at least we can easily split that one up. 

The overall layout is:

```text
| header | integers | NULL bitmap | string end offsets | string bytes |
 6 bytes   4 * I      8 bytes       2 * S               B bytes
```

Here:

- `I` is the number of integer columns
- `S` is the number of string columns
- `B` is the total number of UTF-8 bytes used by all string values

## Header
The header itself is 6 bytes long and contains three `u16` values:
- total row size
- integer column count
- string column count

So the total row size works out to `6 + 4 * I + 8 + 2 * S + B`.

## Integer storage
Integer columns are stored in schema order, starting immediately after the header. The first integer starts at byte offset `6`, and integer `j` starts at `6 + 4 * j`.

A NULL integer still occupies four bytes and is represented with zero in the payload; the bitmap tells us whether that value is actually NULL or just the integer zero.

## NULL bitmap
The NULL bitmap is a fixed `u64` value stored right after the integers. Bit 0 represents the first column, bit 1 the second, and so on. A set bit means NULL.

The fixed-size bitmap is a design choice: it handles up to 64 columns, which is enough for now, and any unused bits stay zero.

This is the first time I am implementing this concept - the Java version did not support NULL - so idk, things might have implementation wise. 
Though it was fun to do some bit shifting. 

## Strings and offsets
Strings are stored after the bitmap as a sequence of *end offsets* and then the UTF-8 payload itself.

Each string column means one `u16` in the end offset array. These offsets are relative to the beginning of the string-data section, not to the row itself. To find string `j`, we look at its end offset and the previous string end offset. The first string starts at offset `0`.

This means if the end offsets are `[4, 17]`, the first string occupies bytes `0..4`, and the second occupies `4..17`.

This is a nice encoding because it lets us find a value without scanning everything before it, while still keeping the data packed together. Other options (e.g. from the Java impl) were to store a pair of shorts for each string - `(start, length)`. It allowed us not to check the previous entry to locate string `j`, but it used more space. Decisions decisions.

NULL and empty strings both contribute zero bytes to the string data section; the bitmap determines which one it is.

## Example
For a schema like this:

```text
id    int
name  string
age   int
email string
```

and a row like:

```text
{ id = 1, name = "Mary", age = 25, email = "mary@mail.com" }
```

we have two integer columns and two string columns. `Mary` uses 4 bytes and `mary@mail.com` uses 13 bytes, so the row total is:

```text
6 + 8 + 8 + 4 + 17 = 43 bytes
```

The row is laid out like this:

```text
row-relative range   contents
0..6                 header: 43, 2, 2
6..14                integers: 1, 25
14..22               bitmap: 0
22..26               string end offsets: 4, 17
26..43               blob: "Mary" followed by "mary@mail.com"
```

The actual bytes are:

```text
2B 00 02 00 02 00                         header
01 00 00 00 19 00 00 00                   integers
00 00 00 00 00 00 00 00                   bitmap
04 00 11 00                               string end offsets
4D 61 72 79                               "Mary"
6D 61 72 79 40 6D 61 69 6C 2E 63 6F 6D    "mary@mail.com"
```


## Why and how
The implementation is pretty simple - just some public free floating functions. I kinda like the fact that I can do these - in Java I had to have a static class for these methods.
Anyways, nothing special - just a function that receives a row & schema and produces a binary array if validation passes. Another one that does the opposite. Schemas are always passed for double checking. Rows by themselves, while they can be read and decoded, do not contain schema information (like column names or anything like that). 
It's pretty much the same design as I had in Java, with the NULL bitmap added - it worked pretty well there. 

# AI in the repo
When I started this project I decided the limit the amount of AI I use - not cause I have anything against it, but because the goal of this project is for me to learn stuff and think things through, so having an agent write the implementation for me would have deprived me of those lessons - topic for another day, but I don't think you can actually learn something by having an AI do it. 

Anyways, the other goal for the project was to do it more 'correctly', as part of the *lessons learned* from the Java attempt. Some of the top items on this list are: do proper testing and have proper documentation.

And I still stand by those decisions - they will definitely help down the line.
But this is also a pet project, done in my (more often than not) limited time. And it turns out that not only do they use a lot of time when there is only one person who has to handle them, but they also leave one quite tired.
Perhaps that's one of the reasons I've not worked as much on this lately - whenever I opened the project, I was presented with the docs I had been working on previously, still in a draft state, still a long way to go. Or I would look at the method I had just written and realize I would need to spend at least an hour testing it and validating everything. 
Useful, but boring. 

So this weekend I've spent some time designing some agent skills to help me with that. The goal is still not to have AI write the code - but the help with the repetitive and uninteresting maintenance tasks, so I **can** write the code myself.

## Skills
### Maintenance
Started with some skills that handle technical docs - look at what's already there, what's implemented and keep the docs up to date. On demand or scheduled, depending on the situation. 
Also added some reviewer-style skills: mostly looking for gaps in docs, tests, reviewing any drifts in architecture, dependencies, tech debt, etc. 
One can be executed pre-PR: it checks the docs, checks the tests, does a review of the code changes to identify any obvious issues. Nothing fancy.

### Requirements and testing
Coming from the regulated field of medical software, I also thought I could apply some of the things from that way of working (simplifying things by making them more complex, I know.)

Namely, one area that I knew from the beginning I would need, but did not feel like doing was requirements. And once you have those, testing becomes a bit easier (so does implementing).

So as a next step, I implemented a few other skills with the following goals:
- turning rough ideas into precise requirements
- proposing test cases from those requirements
- reviewing for gaps in the test design
- implementing approved tests without drifting into implementation guessing

The workflow is based on the following skills:
- `requirements-engineer` grills me about one topic/area (based on docs/what I give as input/domain knowledge) and proposes requirements for the desired implementation, stored in `docs/reqs.md`
- `blackbox-test-designer` produces tests from the requirements without reading the production implementation
- `whitebox-test-designer` looks at the code and proposes edge-case and unhappier flow scenarios
- `test-implementer` turns the approved cases into Rust tests

None of them run automatically and the transition from each step to the next requires my review & approval. 

I did a small trial run on the `Schema` component, and it was relatively useful. The grill-me part was probably the most useful and tbh I think the one that will be the most valuable in the future - partially cause it does write the reqs for me, but also cause it forces me to think of various edge cases, flows and other things I might have missed. The reqs it produced were ok - I had to do some tuning, but in the end they were ok. The test cases were also relatively ok - I had to tame it down since it ended up doing way too many for such a simple thing. 
The test cases implementation - I left this one sorta open in the skill, so that depending on the situation I can decide whether I want it to implement them as unit tests, public API tests, fully functional tests.

This also does not mean I won't write any more tests or at least provide test cases, but I'll the the agent handle the large chunk of it - while I still think that they are important, a big chunk of them are relatively mechanic, throwing multiple inputs and checking the outputs. Also writing tests that verify the correct bytes are placed in the correct places in an array is not a super fun experience for me, not matter what.


Haven't had the chance yet to determine if these will be useful or not, we'll have to see.

During the following weeks I'll continue both with the implementation of new features and with some regression improvements - updating docs for existing functionality, adding requirements & tests. For as long as I still have credits in Codex I guess. 