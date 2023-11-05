---
title: "Plakar: an avalanche of changes"
date: 2023-11-02 10:13:00 +0200
authors:
 - "gilles"
categories:
 - technology
tags:
 - plakar
 - backups
---

{{< tldr >}}
Significant refactoring to improve its performance,
implemented read and write semaphores to throttle chunkers and reduce memory usage,
introduced a repository index and packfiles to decrease the number of I/O operations,
also intrioduced a new HTTP storage backend.
{{< /tldr >}}


# A ton of refactors
I've done so much work on `plakar` in the past few weeks that I don't even know what to begin with...

I've started comparing the current state of `plakar` with other solutions,
for the first time ever since I started working on it:
things were not looking too good performances-wise so I spent a bit of time doing thinking,
analyzing,
refactoring and optimizing.
I'll discuss this in details below.

Things are looking much much brighter now,
it just took a bit of work to get there.


# It all began with a benchmark
Several people have shown interest in `plakar` recently and I had to benchmark it to even have an idea of how it performs,
given that I had never compared it to others.

It turned out... well, not so good.

The features were there and appealing enough that you'd want to use it,
but when it came to actually launching a backup,
the performances were not good and this would rule out the use of `plakar` for anything but small backups.

Actually,
this wasn't completely true.
On SSD disks the performances were very good and you would not see much differences with alternatives,
but once launched on cheap mechanical disks...
oh my.

I stopped testing on my machine's SSD and rented a very cheap dedicated server with cheap mechanical disks,
and began testing on it.


## The problems

### Memory consumption
The first problem that I noticed was that `plakar` was using a lot of memory,
it actually caused the machine to swap,
making the backups slow and the machine unresponsive.

I almost immediately understood why:

`plakar` takes into account the number of cores to adapt its parallelism and not overload the machine CPU-wise,
but once it begins chunking a file and checking if it's already in the repository,
it does so as fast as it can.

On a fast disk,
the operations are fast enough that this doesn't cause any problem,
but on slow disks the chunks are generated faster than they can be written to disk,
they end up buffered in memory and the memory usage grows until the machine swaps...
while at the same time overloading the disk with I/O.



### Disk I/O
Then,
the other problem is that `plakar` was very I/O intensive,

Basically,
the structure of the repository was such that it would require a lot of I/O to read and write files,
given that each chunk had its own file and there was no repository index.

During a backup,
this meant that `plakar` was doing a lot of `stat()` calls to check the existence of a chunk,
then creating a lot of small files containing the actual chunk data.

Let's put that into perspective:
I had a backup with 400k chunks,
which meant that I had to perform roughly 400k `stat()` calls,
then 400k `open()`,
then 400k `write()`,
then 400k `close()`.

I say roughly because it's actually worse then that:
to ensure atomicity,
`plakar` creates a temporary file and renames it once the chunk is written,
so there's extra `rename()` calls too.

This is a lot of I/O,
which didn't show on performances on SSD disks,
but which crushed the mechanical disk.


## The solutions

### Chunkers throttling
There are multiple strategies for that and I took the most naive one to unlock the problem as fast as possible,
leaving me room for improvements later on:
read and write semaphores.

Basically,
once a chunk is generated,
it is passed to the storage layer to determine if it already exists in the repository or if it needs to be stored.
I added pressure control to the storage layer,
which is basically a semaphore that causes the storage operation to block,
causing the chunks generation to block too.

How is this different from letting the disk throughput limit mechanically ?

Well,
it happens ealier in the process,
when only a handful of chunks are lagging behind writes...
versus when the disk is already saturated and there's a ton of chunks already buffered and waiting their turn.

With this mechanism,
I was able to reduce the memory usage by a great lot,
and the machine managed to complete the backup without swapping.
I have ideas to improve this mechanism,
but it's already a great improvement and enough to unlock the problem,
so I'll leave it as is for now.


### Repository index and packfiles
The second solution was to reduce the I/O and I lacked ideas.

I installed `restic` on the cheap VPS to see if it suffered from the same issue,
and I figured they were doing something that I wasn't,
so I created a snapshot on both software and looked at the repository structure.

Unlike `plakar`,
they had a repository index and packfiles,
which allowed them to reduce drastically the number of files and I/O required to read and write chunks.
Basically,
instead of having a file per chunk,
chunks are grouped into packfiles and a repository index allows figuring which one contains the chunk.

I had thought about that in the past,
inspired by `git`,
but didn't go that way as I loved the idea that the repository didn't have any index and was just structured storage...
however,
`restic` proved me that packfiles and repository indexes provide such a huge performance boost that the benefits are too big to ignore.
Also,
considering WHY they bring that huge perforamnce boost,
I know that I will not be able to compensate that with smartness and hacks,
I either go packfiles or I'll be stuck with performances lagging behind `restic` and similar solutions forever...

so I implemented my own version of packfiles and repository index into `plakar`.

All in all,
I had to admit that it was a good idea despite my initial reluctance.

First of all,
the client fetches the index before it creates a snapshot so it can figure out what chunks and files are already in the repository,
without having to query for each of them,
this reduces the number of `stat()` calls A LOT.

Then,
as it chunks,
instead of writing the chunks to disk one by one,
it buffers them into a packfile and writes the packfile once it's over a specific threshold.
This reduces considerably the number of I/O operations,
as you can turn a ton of `stat()`, `open()`, `write()`, `close()` and `rename()` for small chunks,
into the sequential writing of a single `20MB` file.

With a bit of smartness (that I didn't implement yet),
I can even increase the chance that related chunks are written in the same packfile,
which would reduce the number of packfiles that need to be accessed when restoring a file.
This is a code-related issue,
it doesn't change the structure of packfiles or the repository index,
so I thought I'd tackle it later on and it can come as a transparent performance improvement in a future release.

This was a lot of work,
which took me a couple days to complete,
but it was worth it:
the performances are now on par with `restic` and I have a lot of room for improvements.

It required not only implementing the packfiles and repository indexes,
but also required implementing locking similarly to what `restic` does.
I had to implement a locking mechanism pre-packfiles anyways,
this just pushed it higher in my priority list as it was immediately required.


# Reworked all storage backends
The move to packfiles and repository index was a big change,
it required an interface modification,
which required the rewriting of all storge backends for repositories.

It's a bit heavy to describe in this article,
so I won't go into details,
but basically the way you store chunks in a repository changed,
which means that all functions around that changed,
and since there are now packfiles and repository indexes,
new functions are required to manipulate these too.

I took the opportunity to rewrite them all,
which made me spot a bug that caused the compiler to be less strict than it should have been.
Won't expand much as it's not that interesting,
but I now have `sqlite`, `fs`, `plakar`, `ssh` and `s3` working again...
but in packfile mode.

While I was at it,
I implemented a `null` storage backend,
helping me test and benchmark specific areas of `plakar`.


# Implemented HTTP storage backend
Since I was already in the `plakar/storage/backends` directory and had just implemented a `null` backend,
I decided to implement a storage backend that would expose a plakar repository over HTTP.

It literally took half an hour,
which is not due to magical skills,
but just shows that the storage interface is simple and can be used to implement custom storages easily:
anyone could write a custom `plakar` backend with minimum knowledge of the internals,
just by copying the `null` backend and filling in the gaps.

There's not much to say about it,
it does what you expect it to do,
which is expose a repository over HTTP for read and write.
The interesting part if is that it uses the same structures as the TCP protocol for requests and responses,
which means that the code uses the same structures to handle both protocols.

The HTTP backend is clearly my favourite one as it allows me to expose a repository over the network,
in a stateless way.


# Switch to blobs
Until recently,
a snapshot was a directory which contained a set of files:
INDEX, FILESYSTEM, METADATA.

This annoyed me because it made it harder to make changes to the model,
so I decided to switch to a generic blob mechanism,
where a repository can store blobs of data which the snapshot header can reference.

If I decide to associate new kinds of data to a snapshot,
I can just add a new blob type and reference it in the snapshot header,
from the repository point of view there's no change.

Same if I want to shard the INDEX, the FILESYSTEM or METADATA,
I can just store them as blobs and reference them in the snapshot header,
or reference a blob that references multiple blobs.

This allowed me to simplify a lot of code as I could remove the snapshot directory and the code that handled it,
once everything has been put in the repository,
a commit is essentially writing the header file ... and that's it.

But also it allowed me to simplify the storage interface since it can now just store blobs,
instead of having an API to store VFS, another one to store INDEX, etc...

While I was there,
I implemented more parallelization in commit by pushing the three blobs concurrently,
this can save some time on a commit while obfuscating which is which to the server.



# Simplified VFS, indexes and metadata
I spent a fair bit of time working on VFS, indexes and metadata to simplify them further,
with the goal of making them shardable in the future,
but also to shrink them as much as possible to push forward the limit when they need to be offloaded from memory.

This work involved making sure that none of these structures used any information that wasn't strictly required,
but also trying to ensure the information could be normalized in such a way that sharding was possible.
For instance,
instead of storing pathnames in the index,
or the VFS id of a pathname in the index,
I store a hash of the pathname so that if I sharded the index I could do it on hash prefixes and immediately now which index contains which pathnames, objects or chunks.

This was very enjoyable because it basically meant I commented fields,
commented the functions that got broken out of it,
figured out if code could be fixed to avoid using these functions,
and if it did...
then I removed big portions of code which is always a good feeling.


# Reworked VFS, indexes and metadata seralization
Plakar is implemented in Golang and the VFS, index and metadata structures rely on the use of the `map` structure,
also known as hash table or dictionnary in other languages.

This leads to a problem when it comes to serialization:

The `map` structure is not ordered,
meaning that it does not guarantee that the order in which you insert elements is the order in which you'll get them back when serialized.
For instance,
if I insert `a`, `b` and `c` in this order,
I might get `b`, `a` and `c` when I serialize the map.
But more importantly,
assuming same values,
if I insert `a`, `b` and `c` and serialize it,
then create a different map with `c`, `b` and `a` and serialize it,
I won't get the same serialized data for both maps despite the fact that they contain the same keys and values.
This is a problem because it means that the serialized data is not deterministic:
if you serialize the same data twice,
you'll get two different results...

Ok, but why is this a problem ?

Well,
if I scan the same filesystem ten times and nothing has changed,
I expect the resulting snapshot to be the same...
but if the VFS, indexes and metadata are serialized differently each time I end up storing variations of the same data.
If they are serialized into the identical output,
then snapshots become essentially free storage-wise when no changes are detected.

The obvious solution is to switch maps to btree,
or to maintain a sorted key list,
but for reasosn that would take too long to explain in this article it's not that simple:
I tested different approaches and `maps` are VERY efficient even if I need to devise mechanisms around them to make them fit my use case.

Long story short,
I had to implement a custom serializer for VFS, indexes and metadata that would ensure I could use maps and still serialize deterministically,
resulting in an awesome outcome:
I can now scan the same filesystem dozens of times and the repository grows only by a few KBs (for the snapshot header) regardless of the size of the actual snapshot.


# Plakar UI V2
I wanted to switch to the new UI for a while now,
but my frontend skills are not good and I decided to actually pay someone to do it for me.

I asked on LinkedIn if someone would take a freelance gig to do it and got a few answers,
eventually settled on someone I knew to do it,
and he started working on it right away.

It's not ready yet but it does look much nicer than the current UI.

I'll probably write a dedicated article about it once it's ready,
so I don't spoil :-)


# What's next ?
There are a few things that need to be tackled in priority:

First, `plakar cleanup` command no longer works and needs to be fixed.
It was straightforward to implement with the old repository structure,
but packfiles introduce new challenges,
so I need to think about it a bit more.

Then,
I need to rework the VFS layer because it currently is hard to shard,
meaning that as soon as you want to browse a specific directory,
`plakar` needs to deserialize the whole tree structure which can take some time with backups containing a lot of files.
The idea is to split the VFS into a set of subparts that can be fetched independently,
and then merge them into a single tree if needed.


Finally,
I need to switch to the new UI and make it the default one.
