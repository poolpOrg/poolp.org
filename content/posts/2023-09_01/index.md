---
title: "Plakar: a TON of changes"
date: 2023-10-07 09:29:00 +0200
authors:
 - "gilles"
categories:
 - technology
tags:
 - plakar
 - backups
draft: yes
---

{{< tldr >}}
TL;DR:
{{< /tldr >}}


# Optimized `go-fastcdc`
I ran into a benchmark which included my implementation of the FastCDC algorithm,
and realised it was not on par with alternative implementations:

```
% cd _bench_test
% go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: bench_test
cpu: Intel(R) Core(TM) i5-5287U CPU @ 2.90GHz
BenchmarkAskeladdk-4               14      81648896 ns/op    1643.84 MB/s         13014 chunks        9364 B/op           0 allocs/op
BenchmarkTigerwill90-4             12      99914431 ns/op    1343.33 MB/s         16027 chunks       10985 B/op           1 allocs/op
BenchmarkJotFS-4                   10     107616777 ns/op    1247.18 MB/s         14651 chunks      131184 B/op           2 allocs/op
BenchmarkPoolpOrg-4                 4     251425674 ns/op     533.83 MB/s         14328 chunks    144083696 B/op       42990 allocs/op
PASS
ok      bench_test    8.210s
```

The benchmark showed that my implementation was considerably slower than three other popular implementations,
it also required far more memory and allocations per op.
I didn't expect it to outperform other implementations because I didn't go for performances and didn't optimize,
but I was surprised by how bad it was... so an hour later I fixed the injustice:

```
BenchmarkPoolpOrg-4                14      78039244 ns/op    1719.87 MB/s         15485 chunks       131200 B/op           4 allocs/op
```

To cut the gap,
I did a lot of different things:

First,
I replaced the chunker input from an `io.Reader` to a `bufio.Reader`,
letting the stdlib take care of buffering input and reducing the number of syscalls required.

Then,
I replaced the logic in the `Next()` method of the chunker which is in charge of returning the next chunk.
When it relied on an `io.Reader`,
it had to read the data,
so my algorithm had to read twice as much as the maximum chunk size so the FastCDC function could find a cut point within that buffer,
which would then allow me to copy and shift data.
With the change to a `bufio.Reader`,
I could peak into the input buffer,
pass the result to FastCDC,
return a slice of the input buffer up to the cutpoint and discard.
This considerably reduces the overhead and is as optimal as I can get with I/O.

Finally,
I micro-optimized the `FastCDC` function,
using `const` where necessary,
assigning variables to containt the value of a struct field to avoid some dereferences,
and so and so until I eventually switched from iterating over a []byte array to using the `unsafe` package and doing pointer arithmetics.

It may seem like a lot but these were quite obvious which is why they could all be completed within an hour.


# Implemented `go-ultracdc`
I implemented `go-fastcdc` sometimes in 2021 and it was the state-of-the-art methode for CDC,
as far as I knew,
with a relatively recent paper describing an algorithm that was based on a combination of techniques that outperformed previous contenders.

I started using it in `plakar` and go focused on other tasks,
so I didn't realize that some of the reasearchers behind the FastCDC algorithm had published a paper describing the UltraCDC algorithm... in 2022.

Unlike FastCDC,
which is the culmination of previous attempts to stack optimizations on a common method,
the UltraCDC method relies on a completely different approach.

I couldn't find the paper in the wild,
so I eventually caved in,
paid the ~$30 rack... free from IEEE,
and got my hands on the paper to have a look at it.

I eventually implemented it as `go-ultracdc`:
```
goos: darwin
goarch: arm64
pkg: github.com/PlakarLabs/go-ultracdc
Benchmark-8           48          24678812 ns/op        5438.58 MB/s          7929 chunks         131200 B/op          4 allocs/op
PASS
ok      github.com/PlakarLabs/go-ultracdc       1.662s
```

I did the same work as with `go-fastcdc` regarding optimizations,
and voila,
I haz UltraCDC at hands and published an ISC-licensed implementation.


# Implemented `go-ringbuffer`
In an effort to improve further,
I had the intuition that if I could replace the `bufio.Reader` with a ringbuffer
it would boost performances by saving data shifting and buffer resizes.
So I implemented `go-ringbuffer`

A ring buffer is a buffer of fixed length which maintains cursors for the beginning and the end of a buffer,
wrapping around the actual buffer end so that data may reach the end of buffer and continue at the beginning of the data as long as it doesn't overlap with the beginning of the data.

<img src="https://upload.wikimedia.org/wikipedia/commons/f/fd/Circular_Buffer_Animation.gif" />

I won't explain much more,
[look it up](https://en.wikipedia.org/wiki/Circular_buffer).

Basically,
my assumption was that a chunker will not return a more than the maximum size of a chunk,
so with a rinbuffer the size of two chunks I'd have enough read-ahead to find a cutpoint,
and could always refill the buffer as soon as cutpoint is found...
without ever needing to shift the bytes from the cutpoint to the beginning of the buffer.

I implement `go-ringbuffer` which provided the semantic I needed...
and it turns out that my intuition was bullshit.
The performances dropped faster than the price of a Bored Ape,
performances are far better with `bufio` and whatever it does than with the refilling required by my ringbuffer.

To be honest,
I had hypothesis that would explain why,
but I didn't bother confirming because they seemed obvious to me at this point and I didn't want to waste time

The ringbuffer implementation can still prove useful,
just not for this specific use-case.


# Merged go-fastcdc and go-ultracdc into go-cdc-chunkers
Given that I already had two CDC implementations,
both following the same interface,
I decided to merge then into a common projet `go-cdc-chunkers` which is ISC-licensed.

This will allow me to maintain a single project,
make it easier to experiment with new CDC chunkers,
and make sure that any optimization in a generic layer like `Next()` is beneficial to all.

{{< github repo="PlakarLabs/go-cdc-chunkers" >}}
<br />


# Switched `plakar` to `go-cdc-chunkers`

Well,
now that I had `go-cdc-chunkers` available which would allow me to experiment with different CDC algorithms and settings,
I switched `plakar` to using it as a dependency rather than `go-fastcdc`.


# Made compression algorithm configurable, defaults to `lz4`
`plakar` compresses data and indexes before they are sent to the repository.

The compression algorithm was chosen as `gzip`,
solely because during development it was easy to `gzcat` a chunk in the repository and check it was as expected.

As I started working on various optimizations,
I profiled `plakar` and realised that `gzip` was much heavier than I assumed.
I gave a test to `lz4` which seemed as a good trade-off,
and which improved the profile...
so I made the compression algorithm selectable.

Right now,
only `gzip` and `lz4` are available,
but the plumbing to allow future improvements is there now.


# Made hashing algorithm configurable, defaults to `sha256`
Well,
if I made compression configurable,
I might as well do the same with hashing since `plakar` relies heavily on hashing.

I tested `blake3` which improved performances significantly,
however this was just so I had a proof of concept of swappable algorithms,
the default remains `sha256`.


# Initial support for a `plakar config`
XXX


# Swapped the chunk offset to a uint64 for large files support
While doing assorted work,
I realised that the index used an uint for the start offset of chunks,
limiting the start offset of a chunk to 2^32... which excluded large files.

Problem solved.


# Lots of work on the index format
I spent a fair amount of time working in improving the index format with two goals:

- reducing the memory footprint as the index needs to fit in memory
- making it possible for future work to shard the index

Summing up the work is hard because there's been a TON of experimenting and maths to estimate scaling,
but work is not finished and I have a clear path where I'm going with this.

For what it's worth, I almost halved the memory required to fit the index,
and it is mostly shardable at this point,
but there are many ideas floating to improve further.



# Refactor snapshots and storage layers
A snapshot consists of METADATA, an INDEX and a FILESYSTEM.

Each one covers a specific kind of data access:
METADATA is for summaries,
INDEX is for finding the chunks necessary to reconstruct a specific object behind a filename regardless of filesystem structure,
FILESYSTEM is for browsing through a filesystem structure and obtaining stat info regardless of actual data.

Technically,
it could be summarised as:
if you want to access data, you're using the INDEX,
if you want to browse files, you're using the FILESYSTEM.

This wasn't entirely true because the INDEX contained everything before I introduced the FILESYSTEM,
and there were leftovers from the move.
So a first refactor was to manage to make the INDEX completely unaware of the filesystem structure,
removing anything unnecessary to shrink it,
while also making sure that I could reconstruct a file from a pathname using the INDEX only without ever loading the FILESYSTEM...
and without replicating all pathnames from the VFS in the index.

The second issue si that the snapshot object in the snapshot layer assumed the loading of METADATA, INDEX and FILESYSTEM,
so whenever loading a snapshot to perform a command like `plakar cat` or `plakar ls`,
there would be an unnecessary overhead in memory consumption but also in terms of time necessary to load this info and deserialize in memory.

Most notably,
`plakar cat` relied on `snapshot.NewReader(pathname string)` which exposes a reader that fetches chunks as they're accessed,
but given that it's exposed from snapshot,
`plakar cat` requires the loading of METADATA, INDEX and FILESYSTEM when all it needs is in INDEX.
There was no easy fix,
a refactor was needed so that the Reader is a storage reader taking an index in parameter,
allowing to obtain a Reader from the storage as long as you have an index regardless if you have a snapshot.

To achieve this required doing a LOT of other simplications in the snapshot and storage layers,
it was... tricky... but it works, the code is simpler and it paves the road for better performances.


# Update `plakar ls` and `plakar cat` to take advantage of the refactor
I updated `plakar ls` and `plakar cat` so the first one can work with a FILESYSTEM without loading the INDEX,
and the second can work with an INDEX without loading a FILESYSTEM,
this reduces memory requirements and is essentially a proof of concept that `plakar` now allows granularity on what it needs to peek into a backup depending on the operation it does.

I need to adapt all commands,
which will take a little while,
but it's a transparent change:
commands work the same, they just become faster and more memory efficient.


# Fix a few bugs
I fixed a few bugs,
most notably a crash when using a relative path,
which was introduced when I added support for VFS importers,
but also a bug with `plakar ui` which might generate a random port below 1024 preventing an unprivileged user from binding.

I added a check to ensure that `plakar` detects its own repository during a snapshot push,
preventing it from trying to push its own reposiotry into its own repository.




# What's next ?
XXX
