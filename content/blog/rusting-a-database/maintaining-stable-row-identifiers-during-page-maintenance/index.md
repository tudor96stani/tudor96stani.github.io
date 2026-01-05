+++
title = "maintaining stable row identifiers during page maintenance"
date = 2026-01-05
slug = "maintaining-stable-row-identifiers-during-page-maintenance"
description = "A rule that must be respected by the storage engine is to guarantee that internal page operations do not alter the slot numbers of existing rows"
tags = ["slotted-page", "storage-system"]
draft = false
+++

## intro
Christmas, New Year's, a 1-week ski trip, time with family, the rabbit hole of a full re-organization of my Obsidian vault into a [Zettelkasten](https://zettelkasten.de/overview/) slip box after reading [How to take smart notes](https://www.soenkeahrens.de/en/takesmartnotes) - all of these resulted in me making far less progress on this project than I would have expected, especially during my 3-week winter vacation. Nevertheless, there has been some progress.

I am still within the storage system, working on the slotted pages for heaps and there have been a few interesting things I have encountered around the way a database engine will handle certain page maintenance operations.

## row IDs
Most systems (though here I am mainly knowledgeable about how SQL Server handles things) will use a *row ID* to internally identify a row.

A **row ID** can be considered any identifier that allows the system to uniquely identify a single row. There can be multiple places where the engine would need to store this identifier, but maybe the best example is a non-clustered (or secondary) index. In here, each index entry will contain a pointer to the full row it belongs to. When retrieving rows based on that non-clustered index, the row identifier is used to obtain the full row (if needed).

### how SQL Server does it
In the case of SQL Server, row IDs used in non-clustered indexes depend on the source table, and it is usually referred to as the 'row bookmark value' (Delaney et al., 2013, p. 308):
- if the source table is clustered, the non-clustered indexes will use the clustering key as the row identifier
- if the source table is a heap, the non-clustered index will use a physical row locator to identify the row

Today we are only focusing on heaps, as clustered tables are a whole other beast. That physical row locator (called RID by Microsoft) is an 8 byte value comprised of 3 different segments `FileID:PageID:SlotId` and can be broken down into 2 bytes for the FileId, 4 bytes for the PageId and 2 bytes for the SlotId (Delaney et al., 2013, p. 311).

The relevant point here: **a row ID will contain the slot number of the row and this row ID will appear in all non-clustered indexes**. Which leads us to the concept behind this post.

## stable slot numbers
Because indexes will use the slot ID as part of the row's identifier, the system needs to guarantee that rows maintain their slot number during internal page maintenance operations. If rows were to be assigned to other slots, all non-clustered indexes would also require an update, which would be pretty bad performance wise.

The most direct implication of this rule is for the process of page compaction but, as I've noticed, the journey to guarantee this rule starts with deletions.

## row deletions
One of the advantages abstracting the access to rows through the slot array is that certain operations such as deletions become trivial. Similar to how a file system works, deletion from a slotted page does not require us to touch the actual row data. Instead, it is enough to invalidate the slot pointing to that row, thus making the area of bytes corresponding to the row completely unreachable. 

The process is quite simple, but what does it mean to invalidate a slot? In my case, it is just a matter of setting both its `offset` and `length` fields to `0`, while keeping the slot count from the header untouched:
```
initial: [(96, 100), (196, 50), (246, 50)]
slot_count = 3
// delete slot index 1 (196, 50)
after: [(96, 100), (0, 0), (246, 50)] 
slot_count = 3
```
This implementation adheres to the **stable slot numbers** rule:
- if, instead of invalidating the slot, we would have deleted it from the array completely (and decremented `slot_count`), we would have had to shift the next slots one index down, thus blatantly breaking the rule
- if we set it to `(0,0)` as we are doing now, but decremented the `slot_count`, we would have made the last valid slot unreachable.

Turns out this rule logically simplified the implementation by quite a lot. The only more complex part of deletion is a small optimization I've added: if the row being deleted is *physically* the last on the page (so right at the end of the data section), we can actually avoid leaving the page fragmented by moving the `free_start` pointer to the end of the previous row (physically). If this is the happy case, the `can_compact` flag is also not set in the header to indicate fragmentation has started. 

I had actually missed doing this in the Rust implementation (despite me having done it in the Java db which was open on the other display) and I only realized it when I was trying to do the setup for some insertion tests - I initially marked the scenarios as 'technically possible, but not valid', until I gave it some more thought and discovered that they are in reality impossible *only if this optimization is implemented* (you can still see the [comment where I marked them as impossible](https://github.com/tudor96stani/trdb/commit/182f42336fcb4195aeae1a95d7efc12df88eef88#diff-dfdbef1dd59ac775b8b2417ced94bb200da00cadb034183714effb8018c0003aR256))

## page compaction
The process of internally reorganizing the records in the page, in order to remove fragmentation. While all pages start off as tightly packed arrays where all rows are in a contiguous chunk of bytes, deletions and updates can lead to small fragments of bytes appearing in between rows. As a result, a page's free space counter might be high, but without any contiguous area of space large enough to fit a new row. 
Compaction is done by moving all the existing rows next to each other, thus removing any fragments. 
It is usually a lazy process, performed when an insertion cannot find any large enough fragment to fit the new row.

My implementation for it might not be the most optimal, but it seems to work. I allocate a new byte array the size of the data segment (`free_start - header_size`), go through the slot array and for each *valid* slot, copy the row in the new byte array, packing them one after the other & updating the slot offset value. Then I just overwrite the data segment with the new byte array (which will have some trailing 0s, as many as there were bytes from fragments).

I'm sure there is some fancier in-place approach where you don't need to temporarily allocate a new array of up to 4k bytes, but that is more of a nice-to-have, later-on-if-i-ever-get-to-it optimization. Especially since this implementation checks the most important box: it does not break the rule of stable slot numbers. Funnily enough, the initial implementation in Java was to actually allocate a whole new page and only copy the valid slots, then replace the current page's byte array (a hacky way of leveraging the insertion algorithm without having to think too much). It was only while discussing the whole process with a friend almost a year later that we realized how the approach violated this rule. So maybe that's why I'm writing about it.

## outro
That is mostly it for this post, which turned out to be way longer than I expected anyways. As I said in the beginning, things have been moving pretty slowly during the last few weeks, but they should pick up now that the holidays are over.

I've also been working on is a nicer infrastructure for testing - my goal for this project is to try and add tests for everything as I am writing the code, to avoid having to cover tens of scenarios later on. I've been looking into ways in which I can leverage some of the features of Rust to create helpers for this module that will make the setup of any test scenario a breeze - but I think I will cover that in another post.

One other thing that has kept me busy has been the insertion algorithm - the Java implementation was quite chunky, so I've been splitting it into smaller pieces, cleaning them up (and trying to cover them as much as possible).

As next steps, I will hopefully finish the things related to heap pages pretty soon. Then I need to decide whether I want to stay in the storage engine and work on things like index pages (and the monstrous B+trees), or to expand into the other modules. 
Most likely I am going to go with the second. This whole process is quite slow, not due to the differences between the two languages, but because at every step I am questioning many of the implementation details from the Java db, trying to improve things or to solve shortcomings. Because of this, going through all the layers with a small working prototype might make decisions easier when I revisit them to complete the MVP. And on a subjective side of things, I'd like to have a working prototype as soon as possible.

## references
- [Delaney, K., Beauchemin, B., Cunningham, C., Kehayias, J., Freeman, C., Nevarez, B., & Randal, P. S. (2013). _Microsoft SQL Server 2012 Internals_](https://www.microsoftpressstore.com/store/microsoft-sql-server-2012-internals-9780735658561)