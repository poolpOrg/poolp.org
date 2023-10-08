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
---

{{< tldr >}}
TL;DR:
a lot of work in plakar,
but also on CDC and index optimization.
{{< /tldr >}}


# Optimized `go-fastcdc`
**I ran into a benchmark which included my implementation of the [FastCDC](https://www.usenix.org/system/files/conference/atc16/atc16-paper-xia.pdf) algorithm,
and it made me realise that it was not on par with alternative implementations**:

{{< note author="gpt-4" >}}
Hold up! Before we dive deeper, let's demystify CDC a bit. CDC stands for Content-Defined Chunking. Imagine you're trying to find the best places to split a chocolate bar so that each piece has a nut (or not, if you're not into that). That's kinda what CDC does, but with data. Cool, huh?

The FastCDC algorithm is a nifty tool used for content-defined chunking. Think of it as a way to split data into chunks based on the content, rather than fixed sizes. Super useful for things like deduplication!
{{</ note >}}

```t
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

**The benchmark showed that my implementation was considerably slower than three other popular implementations,
it also required far more memory and allocations per op.**
I didn't expect it to outperform other implementations because I had focused on getting it functional and didn't even look at performances,
but I was surprised by how bad it was... so an hour later I fixed the injustice:

```t
BenchmarkPoolpOrg-4                14      78039244 ns/op    1719.87 MB/s         15485 chunks       131200 B/op           4 allocs/op
```

Now that my implementation no longer lags behind the others and can actually outperform even the top contender in the list,
I can write the following without it sounding like a lame excuse:
this benchmark was not really fair to JotFS and my implementation.

{{< note author="gpt-4" >}}
Speaking of benchmarks, it's always a fun exercise to see how different algorithms stack up against each other. While FastCDC has its merits, there are other contenders in the ring too. Each with its own set of strengths and quirks. But hey, variety is the spice of life, right?
{{</ note >}}


First,
the top contender state that in this unscientific benchmark their implementation is 20% faster than the next best implementation,
however their implementation took shortcuts:

> Unlike other implementations it is not possible to tweak the parameters. This is not needed because there is a sweet spot of practical chunk sizes that enables efficient deduplication: Too small reduces performance due to overhead and too high reduces deduplication due to overly coarse chunks. Hence, chunks are sized between 2KB and 64KB with an average of about 10KB (2KB + 8KB). The final chunk can be smaller than 2KB. Normalized chunking as described in the paper is not implemented because it does not appear to improve deduplication.

Then,
the top two implementations do not provide a usable API,
they benchmark the FastCDC algorithm by itself in a copy-less pattern:

```go
n, err := fastcdc.Copy(w, r)
```

where the chunks are split from a `Reader` stream and written in sequence to a `Writer`,
without the ability to intercept them and process them in any meaniningful way...
why even apply FastCDC and not `io.Copy()` ?

As soon as you want to make something useful of the chunking,
it implies being able to return the individual chunks to a caller,
which means additional overhead of somehow passing the data in a buffer from the chunker to the caller...
which JotFS and my implementation do.

Anyways,
I closed the gap so I can speak freely my mind on this and warn that benchmarks should be taken carefully when you don't have a close understanding of what is being measured,
and I will also provide a `Copy()` method so that I can actually benchmark my raw FastCDC implementation as was done by others and measure the overhead of my API to expose chunks.


## Buffered I/O
First,
I replaced the chunker input from an `io.Reader` to a `bufio.Reader`,
letting the stdlib take care of buffering input and reducing the number of syscalls required.
I don't know why I didn't use a `bufio.Reader` to start with,
I wouldn't skip `stdio` in C to do I/O buffering on my own...
yet I did in Golang.

## Reworked `Next()` to reduce allocations and copies
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
This considerably reduces the overhead and is as optimal as I can get with I/O as far as I can tell:

```go
func (chunker *Chunker) Next() ([]byte, error) {
    // chunkder.rd is a bufio with a buffersize that's 2*chunker.maxSize
    // so I can always peek chunker.maxSize safely while avoiding allocs
    // and not wasting too many memory either.
    //
    // data is a slice in bufio's buffer so it's not a "real" allocation
    //
	data, err := chunker.rd.Peek(chunker.maxSize)
	if err != nil && err != io.EOF && err != bufio.ErrBufferFull {
		return nil, err
	}

	n := len(data)
	if n == 0 {
		return nil, io.EOF
	}


	cutpoint := fastCDC(chunker.options, data[:n], n)

    // we already have data available in a slice coming directly
    // from bufio's buffer, we don't need to read again, just do
    // a Discard() so bufio skips the bytes we'll return.
	if _, err = chunker.rd.Discard(int(cutpoint)); err != nil {
		return nil, err
	}

    // return a slice on the slice from Peek
	return data[:cutpoint], nil
}
```

## Micro-optimized `FastCDC`

Finally,
I micro-optimized the `FastCDC` function to go from this:

```go
func (chunker *Chunker) fastCDC(src *bytes.Buffer, n uint32) uint32 {
        NormalSize := chunker.NormalSize

        MaskS := uint64(0x0003590703530000)
        MaskL := uint64(0x0000d90003530000)

        if n <= chunker.MinSize {
                return n
        }
        if n >= chunker.MaxSize {
                n = chunker.MaxSize
        } else if n <= NormalSize {
                NormalSize = n
        }

        fp := uint64(0)
        i := chunker.MinSize

        for ; i < NormalSize; i++ {
                fp = (fp << 1) + G[src.Bytes()[i]]
                if (fp & MaskS) == 0 {
                        return i
                }
        }

        for ; i < n; i++ {
                fp = (fp << 1) + G[src.Bytes()[i]]
                if (fp & MaskL) == 0 {
                        return i
                }
        }

        return i
}
```

To this:
```go
// prototype has changed, it receives a []byte rather than a bytes.Buffer
//
func (c *FastCDC) Algorithm(options *chunkers.ChunkerOpts, data []byte, n int) int {
    // since MinSize, MaxSize and NormalSize are going to be accessed several times,
    // assign to local variables and save a few struct dereferences
    //
	MinSize := options.MinSize
	MaxSize := options.MaxSize
	NormalSize := options.NormalSize

    // constify MaskS and MaskL, this avoids setting in the stack at each call
	const (
		MaskS = uint64(0x0003590703530000)
		MaskL = uint64(0x0000d90003530000)
	)

    // readability change
	switch {
	case n <= MinSize:
		return n
	case n >= MaxSize:
		n = MaxSize
	case n <= NormalSize:
		NormalSize = n
	}

	fp := uint64(0)
	i := MinSize

    // instead of having two loops, one from i to NormalSize with MaskS
    // and one from NormalSize to n with MaskL, use a variable that has
    // a toggle point in a simple loop.
    //
    // since I'm 100% positive the buffer is n bytes long, I can switch
    // to pointer arithmetics and avoid Golang checks that an index has
    // not exceeded an array capacity.
    //
	mask := MaskS
	p := unsafe.Pointer(&data[0])
	for ; i < n; i++ {
		if i == NormalSize {
			mask = MaskL
		}
		fp = (fp << 1) + G[*(*byte)(p)]
		if (fp & mask) == 0 {
			return i
		}
		p = unsafe.Pointer(uintptr(p) + 1)
	}
	return i
}
```

It may seem like a lot of work but the function being small to begin with,
it was just a matter of thinking about the cost of each line and how to squeeze a bit of perfs,
then benchmark a change and see if it brought significant gains or should be reverted before applying another change.


# Implemented `go-ultracdc`
I wrote my `go-fastcdc` implementation sometime in 2021,
and back then [FastCDC](https://www.usenix.org/system/files/conference/atc16/atc16-paper-xia.pdf) was the state-of-the-art algorithm for CDC (as far as I knew).

I won't spend much time explaining how it works,
there's a paper that's readily available and that describes how it works and how it compares to others.
Basically it was not a brand new method but a set of optimization applied to top of a previously published method to boost its performances.

I started using my package in `plakar` and focused on other tasks,
given that CDC is very important but far from being the key feature,
so I didn't pay attention when some of the researchers behind the FastCDC algorithm published [a new paper describing the UltraCDC algorithm](https://ieeexplore.ieee.org/document/9894295)...
a few months later in 2022.

Unlike FastCDC,
the UltraCDC method relies on a completely different approach rather than optimizations of an existing method,
and announces chunking speed from 1.5 to 10x faster than the state-of-the-art CDC approaches,
with comparable or higher deduplication ratio.

So I was like:
SAY NO MORE SCIENTISTS,
WHERE DO I SIGN ?

I couldn't find the paper in the wild,
it was hidden behind a racket-wall at IEEE,
unavailable in the closest sci-hub.
I couldn't find how to contact the authors,
so I eventually caved in and paid the ~$30 tax which is probably used to pay authors fairly (lol).

Anyways,
I eventually implemented the paper in a `go-ultracdc` package:
```t
goos: darwin
goarch: arm64
pkg: github.com/PlakarLabs/go-ultracdc
Benchmark-8           48          24678812 ns/op        5438.58 MB/s          7929 chunks         131200 B/op          4 allocs/op
PASS
ok      github.com/PlakarLabs/go-ultracdc       1.662s
```

The results were initially lovely,
but it turns out that it doesn't necessarily translates to better performances on my use-case.
As you can see, there are far less chunks,
because the average chunk size has slighly increased,
and for some reason this translates to slower backups in `plakar` despite faster chunking.
I still need to investigate why to get an actual understanding beyond observations,
so at this point `plakar` is still FastCDC-powered.

The `go-ultracdc` implementation benefited from the optimizations made in the `Next()` method for `go-fastcdc`,
and some optimizations I had applied to the `fastCDC` function,
but unintuitively not all optimizations produced the same benefits and I reverted some to favour readability.

Anyways,
at this point I had UltraCDC at hands and published an ISC-licensed implementation.


# Implemented `go-ringbuffer`
**A general performance bottleneck in CDC algorithms is that you are looking for cut points in a stream of bytes for which you have to read-ahead a certain amount**.
Once you find a cut point in between the beginning of the buffer and the end of your read-ahead,
you end up with left-over bytes that should be the beginning of your next chunk.
A naive approach would be to move these left-over bytes at the beginning of the buffer,
but this means that at each iteration of the `Next()` method you read the left-over bytes twice and copy them once.
An alternative would be to leave them where they are and allocate more memory behind to serve the next chunk,
which may or may not be possible and which may or may not impose a copy.

{{< note author="gpt-4" >}}
Buffer talk! Imagine you're reading a book but can peek a few pages ahead. That's kind of what a read-ahead buffer does with data. Tweaking how you handle that peeking can make a world of difference.
{{</ note >}}


There are several ways to tackle this, each with pros and cons,
and an article on its own would be needed to discuss the topic.
However,
in an effort to squeeze performances further,
I had the intuition that if I could replace the `bufio.Reader` with a ringbuffer
it would boost performances by avoiding data shifting and buffer resizes.

A ring buffer is a buffer of fixed length which maintains cursors for the beginning and the end of data within that buffer,
wrapping around the actual data end so that it may reach the end of buffer and continue at the beginning of the buffer as long as it doesn't overlap with the beginning cursor again.
I won't explain much more,
[this Wikipedia article which I didn't read](https://en.wikipedia.org/wiki/Circular_buffer) probably does a better job at explaining it than me :-)


<img src="https://upload.wikimedia.org/wikipedia/commons/f/fd/Circular_Buffer_Animation.gif" />


My assumption was that since a chunker will not return more than the maximum size of a chunk,
then a ringbuffer that's the size of a maximum chunk can provide the guarantees I need without reallocations and without shifting.
If the ringbuffer is filled with MaxSize bytes,
the cutpoint is guaranteed to be somewhere between the beginning cursor and MaxSize,
when they are consumed it moves the cursor to the cutpoint and the ringbuffer can be refilled with as many bytes as were read starting from after the left-over bytes.
This avoids any shifting,
guarantees that there are no reallocations,
and basically seems optimal on paper.
Furthermore,
you can control I/O pressure easily,
for example by assigning a ringbuffer size of 2 (or 3, or 4) * MaxSize,
ensuring that there are two chunks of read-ahead and reducing the frequency of having to refill the buffer and hitting the underlying `Reader`.

The ringbuffer could be made to look like a `Reader`,
so that it could be swapped wherever I needed easily,
and so I implemented `go-ringbuffer` which provided the semantic I needed...
which proved to be one of the greatest idea I had ever had.

Nope.
Actually,
it turned out that my intuition was buillshit.
The performances dropped faster than the price of a Bored Ape NFT.
Performances are far better with `bufio` and whatever it does than with the refilling required by my ringbuffer.
I have hypothesis that could explain why,
but I didn't bother confirming yet because they seemed obvious to me in retrospect and I didn't want to waste more time on this.

The ringbuffer implementation can still prove useful,
just not for this specific use-case.


# Merged `go-fastcdc` and `go-ultracdc` into `go-cdc-chunkers` and switched `plakar` to it
Given that I already had two CDC implementations,
both following the same interface,
I decided to merge them into a common `go-cdc-chunkers` project which is ISC-licensed.

This will allow me to reduce my workload by maintaining a single project,
while making it easier to experiment with new CDC chunkers and malking sure that general optimizations apply to all my implementations.


{{< github repo="PlakarLabs/go-cdc-chunkers" >}}

Now that I had `go-cdc-chunkers` available which would allow me to experiment with different CDC algorithms and settings,
I switched `plakar` to using it as a dependency rather than `go-fastcdc`.
It defaults to the FastCDC implementation,
but I no longer have to swap dependencies and imports all around whenever I want to test something.



# Made compression and hashing algorithm configurable
By default,
`plakar` compresses data and indexes before they are sent to the repository.
This makes sense since CDC deduplication is a kind of compression,
and I can't see a reason why you'd want to CDC deduplicate but not compress.

I chose `gzip` as the compression method primarily because during development it was easy to `gzcat` a chunk in the repository and make sure3 it was as expected:
most modern systems comes with several tools that manipulate `gzip` files easily (`zcat`, `zless`, ...).
But as I started working on various optimizations,
I profiled `plakar` and realised that `gzip` was much heavier than I originally assumed.
I gave a test to `lz4` which is a good trade-off,
and which improved the profile considerably...
so I decided to make the compression algorithm selectable at the repository level and use `lz4` by default now that I no longer spend so much time inspecting chunks manually.

Right now,
only `gzip` and `lz4` are available,
but the plumbing to allow future improvements is there now.

```t
% plakar create -compression gzip
%
```

Since had I already made compression configurable,
I decided to do the same with the hashing algorithm  since `plakar` relies HEAVILY on hashing.

At this point,
it was not so much about efficiency and performances,
but more about getting the plumbing right so if I had to change the algorithm tomorrow due to a security weakness in `sha256` it wouldn't require a huge refactor.
I decided to implement a Hashing layer,
whose algorithm is selected at the repository level,
and adapted the code to rely on that method rather than calling `sha256` directly when needed.

To validate my test,
I created a repository configured to use `blake3` rather than `sha256` and validated that it works.
This is an easy case because I selected a 32-bytes digest for both,
but it's a first step towards the direction of not hardcoding a method.
As of now,
the index layer expects digests to be 32-bytes long,
and it would require a bit of work to dynamically pick a specific index structure based on the digest length.
It's very likely an hour of work or so,
I just don't have an incentive to spend that hour now,
particularly because I'm doing other things with the index at the moment and don't want two simultaneous projects on the same layer.

So now,
you can technically switch a repository to use `blake3` which is faster and more efficient on resources,
even though at the moment I prefer to keep `sha256` the default as I have had no look at the current state of crypto in a while and can't take a proper decision on this by myself at the moment.


# Initial support for a `plakar config`
A problem with `plakar` currently is that it doesn't support a configuration file,
so there's no way to keep the command line short.

For example,
I have a repository on my NAS,
and if I want to push to it then my command line is:
```t
% plakar on ssh://gilles@nas.poolp.org/var/backups/plakar push /home/gilles
%
```

If I want to synchronize this with another repository,
I'd need to provide the two URLS:
```t
% plakar on ssh://gilles@nas.poolp.org/var/backups/plakar sync to ssh://gilles@offsite.poolp.org/var/backups/plakar
%
```

So I began basic support for an OPTIONAL configuration file,
`~/.plakarconfig`,
which is absolutely not required but allows storing informations that can be resolved by plakar.

I chose a `yaml` format and plugged a `config` subcommand,
similar to `git config`,
to populate the configuration.

For now it's not really used,
but I did a proof of concept to show how it could work,
allowing the setting of a name for repository:

```t
% plakar config repository nas location ssh://gilles@nas.poolp.org/var/backups/plakar
% plakar config repository offsite location ssh://gilles@offsite.poolp.org/var/backups/plakar
```

which would turn the previous commands into:

```t
% plakar on @nas push /home/gilles
% plakar on @nas sync to @offsite
```

This is just the beginning and I may change my mind a lot around how this is meant to be used,
feel free to command and share ideas.

One idea I already have is that the configuration may be read from a different backend then `~/.plakarconfig`,
allowing it to be shared from a central configuration to a set of users.
It was prepared to support this even though it's not in my plans at the moment.




# Lots of work on the index format
I spent a fair amount of time working in improving the index format with two goals:

- reducing the memory footprint as the index needs to fit in memory
- while making it possible for future work to shard the index

Summing up the work is hard because there's been a TON of experimenting and maths to estimate scaling,
as well as a LOT of braindump on my discord server,
but I'm not finished even though I have an already clear path where I'm going with this.

After a week of work,
I almost halved the memory required to fit the index,
and it is mostly shardable at this point.
There are many ideas that still need to go in before I'm happy.

**The best features in `plakar` are really dependant on the ability of having an efficient index**.
I could immediately give up on this and provide very fast backup and recovery,
by dropping the index,
but this would mean that features like file-level restore,
diff between files in different snapshots,
the snapshots browser or even search would go away.

{{< note author="gpt-4" >}}
And speaking of efficient indexes, think of them as the super-smart librarians of the digital world. Without them, finding that one piece of data would be like searching for a needle in a haystack. With plakar, it's more like searching for an elephant in a room. Hard to miss!
{{</ note >}}

In my bigger plan,
the index is a key element of snapshots,
and it is essential that I take proper time to make it scalable WAY BEYOND expectations both in terms of resources required to operate and access time.

At the current time,
`plakar` can backup pretty much any volume of data,
but backups > 100G already require around 1GB of RAM to create and as much to restore ... with an additional overhead of deserializing the index in memory.
This is what I'm currently working on optimizing,
because once it falls in a reasonnsable range,
we get scalable backups with a TON of nice features that are instant-time for the most part.




# Refactor snapshots and storage layers
A snapshot consists of `METADATA`, an `INDEX` and a `FILESYSTEM`.

Each one covers a specific kind of data access:
`METADATA` is for summaries,
`INDEX` is for finding the chunks necessary to reconstruct a specific object behind a filename regardless of filesystem structure,
`FILESYSTEM` is for browsing through a filesystem structure and obtaining stat info regardless of actual data.

Technically,
it could be summarised as:
if you want a summary, you're using the `METADATA`,
if you want to access data, you're using the `INDEX`,
if you want to browse files and see how they are related one to another, you're using the `FILESYSTEM`.
So when you list all your snapshots,
you actually access their `METADATA`,
when you call `plakar ls` on a specific snapshot you access its `FILESYSTEM`,
and when you call `plakar cat` on a specific file you access the `INDEX`.

This wasn't entirely true because the `INDEX` contained the three of them before I did a split,
and there were leftovers from the move.
So a first refactor was to manage to make the `INDEX` completely unaware of the filesystem structure,
removing anything unnecessary to shrink it,
while also making sure that I could reconstruct a file from a pathname using the `INDEX` only without ever loading the `FILESYSTEM`...
and without replicating all pathnames from the `FILESYSTEM` in the index.

The second issue is that the snapshot object in the snapshot layer assumed the loading of `METADATA`, `INDEX` and `FILESYSTEM`,
so whenever loading a snapshot to perform a command like `plakar cat` or `plakar ls`,
there would be an unnecessary overhead in memory consumption but also in terms of time necessary to load this info and deserialize in memory.

Most notably,
`plakar cat` relied on `snapshot.NewReader(pathname string)` which exposes a reader that fetches chunks as they're accessed,
but given that it's exposed from snapshot,
`plakar cat` requires the loading of `METADATA`, `INDEX` and `FILESYSTEM` when all it needs is in `INDEX`.
There was no easy fix,
a refactor was needed so that the `Reader` is a storage reader taking an index in parameter,
allowing to obtain a `Reader` from the storage as long as you have an index regardless if you have a snapshot.

To achieve this required doing a LOT of other simplications in the snapshot and storage layers,
which was... tricky... but ultimately it works:
the code is simpler and it paves the road for better performances.

I updated `plakar ls` and `plakar cat` so the first one can work with a `FILESYSTEM` without loading the `INDEX`,
and the second can work with an `INDEX` without loading a `FILESYSTEM`,
this reduces memory requirements and is essentially a proof of concept that `plakar` now allows finer granularity on what it needs to peek into a backup depending on the operation it does.

I still need to adapt all commands,
which will take a little while,
but it's a transparent change:
commands work the same, they just become faster and more memory efficient when they get converted to use the proper API calls.


# What's next ?
Quite obviously my main area of work at the moment is the `INDEX` as I have many plans for it,
some I'm willing to disclose,
other I want to keep private for the time being.

I'm currently assessing the idea of creating a company around `plakar`,
providing an ISC-licensed opensource version that covers most users needs and which would be fairly similar to what `plakar` showcases currently,
and a premium-licensed version for the corporate world with a ton of feature no one needs unless they work with teams and manage >TB of data.

Lots of documentation is being written,
targeting potential investors,
but also allowing me to clear my mind and write down the costs, strategies and related stuff.

Stay tuned !