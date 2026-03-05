---
title: fastbloom - Insanely fast Bloom Filter for Node.js, written in Rust
pubDate: 2026-04-03
---
For those who are new to bloom filters, it is one of the hashing data structures that are used in large scale systems. This post assumes you already know what a bloom filter is and how it works, but if you don't you can check it out [here](https://systemdesign.one/bloom-filters-explained/).

A bloom filter is probabilistic data structure with 2 operations:
- Insert - Add an item to the bloom filter
- Contains - Check whether an item exists in the bloom filter

I am building an analytics platform (for fun) that requires to load up 50 million UUIDs into an in-memory bloom filter on application startup. The application fetches these UUIDs from a Valkey cache.

There are 2 potential bottlenecks to this:
1. The IO bandwidth fetching from Valkey (major)
2. The bloom filter insertion speed, which technically should be minor cause its in-memory right?

Wrong.

A slow bloom filter implementation can slow down the app boot up time by a lot.

![](_assets/bloomfilter-1.png)

Using the most popular bloom filter implementation on npm, the [bloom-filters](https://www.npmjs.com/package/bloom-filters) package takes **~8 minutes to load all 50 million UUIDs into memory.**

That is...slow. Really slow

The reason is due to the whole implementation being written in Javascript, especially the hashing function using the [xxhashjs](https://www.npmjs.com/package/xxhashjs) package which is written purely in Javascript.

Since a bloom filter's inserts are computation heavy due to hashing, we can; in theory; achieve faster results with something that sits closer to the hardware without the layers of JS.

And there starts [fastbloom](https://www.npmjs.com/package/fastbloom), the aim to write the fastest bloom filter for Node.js.

## It is Rust time!

I started with writting the bloom filter in Rust and using NAPI-RS to call Rust functions from Node.js.

Result? **~1 minute to load all 50 million UUIDs into memory.**

That is an **800% speedup compared to the `bloom-filters` package.**

If you are aware about how bloom filter inserts items into its memory, you might know that bloom filter inserts are commutative, i.e, the order doesn't matter.

Keeping that in mind, we could parallelize the process of inserting records into our  bloom filter instead of running a regular for-loop to insert.

I searched for an existing implementation of this and came across a crate that is also coincidently named [fastbloom](https://crates.io/crates/fastbloom).

The crate has an implementation of an `AtomicBloomFilter` that avoids lock contention, allowing us to concurrently insert into the bloom filter.

This means that using [rayon](https://crates.io/crates/rayon), we can parellely insert a gigantic number of items into the bloom filter, utilizing all the CPU cores on the system.



---

PS: I somehow managed to miss out on [fast-bloom-filter](https://www.npmjs.com/package/fast-bloom-filter) that implements most of the heavy lifting in C, and compiles to WASM SIMD 128 and ends up with similar performance to fastbloom's bulkAdd while sequentialling inserting 