+++
title = "insertion in heap pages"
date = 2026-01-19
slug = "insertion-in-heap-pages"
description = "Implementation of the insertion algorithm for unsorted heap pages"
tags = ["slotted-page", "storage-system"]
draft = false
+++

## Intro
Insertion in heap pages is, in theory, straightforward: you find some free space in the page, place the row there and create a slot pointing to that location. There are, however, multiple small details that make things a bit more complicated (mostly edge cases caused by fragmentation). 

The implementation in TRDB is very similar to the one in Java - I mostly covered some scenarios I had missed there & cleaned it up a bit, making it more granular and easier to test. 

I will mostly cover the overall logic and reasoning behind it, limitations & possible improvements. 

## High level
From a high level, the insertion is done in two separate steps:
- plan the insertion
- perform the actual insertion

The planning phase looks for a location for the new row, along with metadata for the insertion algorithm. **No data is changed during this phase, it is a read-only operation.**

The insertion phase uses the metadata provided in the plan to actually place the row at the target position and to update the header & slot array

## Planning phase
This phase attempts to answer several questions about where the row is going to end up and what other changes are needed to the page from a maintenance point of view:
- what slot number will the row use?
- where is it going to be physically placed?
- is compaction needed for this insert?

### Slot number
The first step of the planning is to go through the slot array and look for any unused slots. As a reminder: when a record is deleted its slot is only being invalidated, but kept in the array. The insertion algorithm will reuse these empty slots, to avoid increasing the footprint of the slot array unnecessarily. 

The scan starts from the beginning of the array (rightmost element) and stops at the first unused slot it encounters. If no such slot is found, it flags that a new one will be needed.

### Physical location
In the second step of the planning, we look for a place to physically place the row itself in the page. 

This is done by checking the following conditions, in this order:
1. Can the row fit in the free space? (`free_end - free_start`)
2. Can the row fit between two existing rows? 
3. Can the row fit between the last row and `free_start`?

The options are probed one by one, halting at the first positive answer. If none are successful, the algorithm flags that a compaction is needed for this insertion.

#### Notes
- Prior to this phase, an additional check is done to ensure that the row can physically fit in the page - meaning that there are enough free bytes in the page, regardless of fragmentation; if there are not, the entire flow is aborted. This allows us to have the fallback of compaction (we can be sure that if we could not find a large enough spot right now, there will be one once we compact)
- All checks are done with the information from the first phase in mind - if a new slot is needed, 4 extra bytes are taken into account when determining the necessary overall space in the page.

### Implementation 
The actual result of the planning phase is represented as an enum comprised of two other enums [(code)](https://github.com/tudor96stani/trdb/blob/main/crates/storage/page/src/insertion_plan.rs):
```rust
/// Defines the offset at which a new record should be inserted in an unsorted heap page.  
#[derive(Debug)]  
pub enum InsertionOffset {  
    /// Record should be inserted at the start of free space after compacting the page.  
    AfterCompactionFreeStart,  
    /// Record should be inserted at an exact offset.  
    Exact(usize),  
}  
  
/// Defines whether a new slot should be created for the record or an existing slot can be reused when inserting into an unsorted heap page.  
#[derive(Debug)]  
pub enum InsertionSlot {  
    /// A new slot should be created for the record.  
    New,  
    /// An existing slot can be reused for the record.  
    Reuse(usize),  
}  
  
/// Represents a plan for inserting a new record into an unsorted heap page.  
#[derive(Debug)]  
pub struct InsertionPlan {  
    /// The slot information for the insertion.  
    pub slot: InsertionSlot,  
    /// The offset information for the insertion.  
    pub offset: InsertionOffset,  
}
```

## Insertion phase
This is pretty straight forward: based on the insertion plan received from the previous step, it will know exactly whether it needs to compact the page beforehand, at what offset to place the row and how to handle the slot. 

Besides inserting the actual row data and updating the slot array, the following header values are also updated:
- `SLOT_COUNT` - incremented by 1 only if we are creating a new slot
- `FREE_END` - decremented by `SLOT_SIZE` only if we are creating a new slot
- `FREE_START` - incremented by `row.size` only if we are inserting at this location (meaning we are not using a gap between two rows)
- `FREE_SPACE` - always updated. Decremented by `row.size` if we are reusing a slot, or by `row.size + SLOT_SIZE` if we are also creating a new slot.

## Reasoning
The initial implementation in Java was a single step insert - it worked ok, despite it becoming a bit too bloated. However, I had to split it into these two phase once I started working on write-ahead logging (WAL).

As the name suggests, in WAL you need to create a log entry with the operation that is **going to** happen, meaning you need to have all the information about the change you are about to make **before** you perform the change. This was problematic because one of the fields that had to be included in the log entry was the slot number of the row being inserted - value which was only known during/after the insertion, not before. 

Since it is not acceptable to write an incomplete WAL entry and update it afterward, the only choice was to extract the logic that determined the physical location of the row outside the actual insertion, allowing me to (1) generate the insertion plan, (2) log the operation and (3) perform the actual insert. 

I could have, of course, updated the insert method to perform the log internally, right before it placed the row in the page. However, I preferred to have the `Page` implementation unaware of the WAL and not couple the two in this manner. 

## Possible improvements
During the search for a physical location in the planning phase, because the checks being performed are short-circuiting, their order is relevant and up for debate:
- on one hand the current implementation favors fragmentation, because it will only attempt to fill in gaps if the `free_end - free_start` area is not large enough.  However, it avoids scanning the existing slots if it is not necessary, thus providing a quicker response.
- on the other hand, switching those two would reduce overall fragmentation, but would also incur useless scans of the slot array for scenarios where the fragmentation is low (which is also kept low by this algorithm)
- don't think there is a right or wrong, although implementing them both and doing some benchmarking would be the only way of telling for sure
	- it is also highly influenced by access patterns: tables with low deletions would benefit from the current implementation, while tables with lots of inserts & deletes could benefit from the second one

Another area of improvement is also during the planning phase: as of right now, the two steps in the planning phase scan over the slot array twice: once for the slot # and once for the physical location. It can be done in a single pass, but for now I chose clarity and testability over efficiency. Again, only some benchmarks could give us a performance penalty for this implementation.