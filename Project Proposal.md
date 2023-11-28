# LZ77 Optimization Project Proposal

> Pranavi Boyalakuntla
Lycia Tran
Autumn 2023
EE 274
> 

---

## Problem

We are planning on optimizing the LZ77 implementation on SCL. LZ77 is the basis for some of the most widely used compressors (gzip, zstd), but the current implementation on SCL does not implement features such as, a sliding window or a chain-hash that would allow for a more optimized performance. 

## Previous Work

As linked in the project description, we will start with this blog post by Glinn Scott that goes over the explains the components of the LZ coding algorithm ([https://glinscott.github.io/lz/index.html](https://glinscott.github.io/lz/index.html)). 

We will explore repcodes, different match finding algorithms and different entropy coders. We will explore several different methods of improving the algorithms and test their individual performance improvement on different types of data sets.

## Design

We will work the TAs and our mentor to determine what optimization strategies will implement. Some that we have discussed in our proposal meeting are using repcodes, different match finding algorithms such as a dynamic programming based optimal algorithm, and different entropy coders. 

## Evaluation

We will test all of our implementations with various datasets to see which LZ77 optimization has the best perfomance. We will work with the TAs to determine what would be the best ways to test the peformance of our various implementations and which data sources we should test our implementation on. We would also compare all of our optimized implementation against the LZ77 implementation that currently in SCL.