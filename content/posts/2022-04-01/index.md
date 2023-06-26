---
title: "April 2022: plakar.io, plakar refactor and ssh support"
date: 2022-04-24 17:24:00 +0200
authors:
 - Gilles Chehade
language: fr
categories:
 - technology
---

<blockquote>
<b>TL;DR:</b>
I refactored internal structures to split metadata from the index, implemented an stdio server and finally added SSH support.
</blockquote>

<center>
    <img src="cover.png">
</center>


# Shout out to my sponsors &#x2764;&#xfe0f;

A **HUGE thanks** goes to my sponsors on [github](https://github.com/sponsors/poolpOrg)
and [patreon](https://www.patreon.com/gilles):
your continuous support is very much appreciated !

I created a Discord where I hang out and discuss my projects or screencast as I work on them.
Feel free to hop in if you want,
and feel free to do just like me and share thoughts as you work on your own projects there:
**this is a virtual hack room for anyone to join**: [https://discord.gg/YC6j4rbvSk](https://discord.gg/YC6j4rbvSk)


# The [plakar.io](https://plakar.io) website

A project needs a website so...
I published the [plakar.io](https://plakar.io) website.

This will centralize instructions and documentation for the plakar project,
but **it is still a work in progress** as development still happens at a fast pace that's hard to keep up with.

The website is published from commits to the [plakar.io Github repository](https://github.com/poolpOrg/plakar.io),
so feel free to submit pull requests to improve it.


# Reworked internal structures

A snapshot generates an index that contains **everything needed to map chunks and objects into a filesystem hierarchy**.
The index is very small comparatively to the snapshot size,
but it may still be relatively large as **a multi-gigabyte snapshot may produce a multi-megabyte index**,
which still requires time to fetch, decompress and deserialize.
In many cases,
this is fast enough that you don't really feel it,
but the bigger the snapshot the more laggy it feels when browsing it through the UI or when listing content in a snapshot.

For many commands there's **no need to access the index** because it's not so much the content of files that is needed,
but really **some metadata or statistics**.
Good examples of these are `plakar ls` in the repository or `plakar info`:

```sh
% plakar ls 
2022-04-19T22:07:03Z  a9d01365    2.4 GB /Users/gilles/Wip
2022-04-19T22:08:11Z  f87bd54e    2.4 GB /Users/gilles/Wip
2022-04-19T22:11:46Z  70cc55f6    2.4 GB /Users/gilles/Wip
2022-04-20T22:48:55Z  1c537551    2.4 GB /Users/gilles/Wip

% plakar info a9
Uuid: a9d01365-4f97-42e1-8b19-59e4fe2f630a
CreationTime: 2022-04-20 00:07:03.728327 +0200 CEST
Version: 0.1.0
Hostname: desktop.local
Username: gilles
CommandLine: ./plakar -no-cache push /Users/gilles/Wip
MachineID: 3657f7dd-c012-53ba-b8e6-73e08d311a6a
PublicKey: 
Directories: 7589
Files: 77206
NonRegular: 84
Pathnames: 77206
Objects: 63306
Chunks: 65702
Size: 2.4 GB (2445114798 bytes)
Index Size (uncompressed): 113 MB (2445114798 bytes)
```

Here,
the index for snapshot `a9d01365` is 113MB uncompressed (really 21MB on disk),
which means that the `plakar ls` above required decompressing and deserializing roughly 4 times that size to list the 4 snapshots,
then decompressing and deserializing that size once for the `plakar info` above,
just to display some informations that didn't rely on the filesystem hierarchy and content.

I have split snapshots into two main structures:
`Metadata` and `Index`.

The first structure contains **general informations and statistics regarding the index**,
just enough that the structure fits in a few KiloBytes,
but can avoid relying on the much larger index for commands that don't involve diving into the content of a snapshot.
The second structure contains **the mappings themselves**,
and is **only accessed when data needs to be reconstructured somehow**.

This allowed `plakar ls` to get a serious boost when listing a large number of large snapshots,
but it is really a first step as I'm evaluating solutions to shard the index and allow fetching subsets of it.


# Implemented a Reader interface

When a snapshot is created,
plakar will consider each file **as an object consisting of one or many chunks**.
The repository stores the object structure and chunks as **separate resources**,
and so when a file is being recovered this is done by **first fetching the structure**,
**then fetching each chunk**.

For example,
to implement `plakar cat`, 
the code would look something like this (error handling omitted for simplicity):
```go
[...]
    object := snapshot.LookupObjectForPathname(pathname)
    for _, checksum := range object.Chunks {
        data, _ := snapshot.GetChunk(checksum)
        os.Stdout.Write(data)
    }
[...]
```

As every call may fail and needs to be checked,
the code is actually far more verbose as can be seen below:

```go
[...]
    object := snapshot.LookupObjectForPathname(pathname)
    if object == nil {
        logger.Error("%s: could not open file '%s'", flags.Name(), pathname)
        continue
    }

    for _, checksum := range object.Chunks {
        data, err := snapshot.GetChunk(checksum)
        if err != nil {
            logger.Error("%s: %s: could not obtain chunk '%s': %s", flags.Name(), pathname, checksum, err)
            continue
        }
        _, err = os.Stdout.Write(data)
        if err != nil {
            logger.Error("%s: %s: could not write chunk '%s' to stdout: %s", flags.Name(), pathname, checksum, err)
            break
        }
    }
[...]
```

and this construct needs to be replicated **everywhere a file is accessed** which is...
pretty much everywhere you do something with a snapshot.

I thought it would be nice to have a Reader interface abstracting the details of reconstructing a file.
This way,
I could simply obtain a reader to a file and call `Read()` to obtain the next bytes **without worrying about fetching chunks as the cursor advances**:

```go
    rd, _ := snapshot.NewReader(pathname)
    buf := make([]byte, 16*1024)
    for {
        nbytes, err := rd.Read(buf)
        if err == io.EOF {
            break
        }
        os.Stdout.Write(data[:nbytes])
    }
```

In the example above,
`snapshot.NewReader(pathname)` fetches the object structure for file located at `pathname`.
As `rd.Read()` is called,
it takes care of fetching chunks as needed to fill `buf` with the next 16k bytes.

How is that simpler than calling `GetObject()` and `GetChunk()` ?

Well,
first and foremost,
there's a large range of functions already available in Go to do what I want but which expect a Reader interface.
So **rather than rolling my own version of functions**,
by implementing the Reader interface,
I can **rely on existing stuff** and reduce the amount of code I have to maintain in plakar.

Rather than re-implementing a loop to read chunks and write them to stdout,
`plakar cat` could take advantage of the new Reader interface and the fact that `os.Stdout` is a Writer,
allowing it to use the `io.Copy()` function part of the standard library to copy from a Reader to a Writer:

```go
    rd, err := snapshot.NewReader(pathname)
    if err != nil {
        logger.Error("%s: %s: %s", flags.Name(), pathname, err)
        continue
    }

    _, err = io.Copy(os.Stdout, rd)
    if err != nil {
        logger.Error("%s: %s: %s", flags.Name(), pathname, err)
        continue
    }
```

This is also the case for `plakar tarball`,
which creates a tarball from a snapshort,
and relies on the standard libraries' `archive/tar`.
The code instantiates a `tar.NewWriter()` and I previously had something like:

```go
    obj := snapshot.LookupObjectForChecksum(checksum)
    for _, chunkChecksum := range obj.Chunks {
        data, err := snapshot.GetChunk(chunkChecksum)
        if err != nil {
            logger.Error("corrupted file %s", file)
            continue
        }

        _, err = io.WriteString(tarWriter, string(data))
        if err != nil {
            logger.Error("could not write file %s", file)
            continue
        }
    }
```

which could be rewritten as follows,
leaving it up to the standard library to loop and handle do the read and writes:

```go
    rd, err := snapshot.NewReader(file)
    if err != nil {
        logger.Error("could not find file %s", file)
        continue
    }

    _, err = io.Copy(tarWriter, rd)
    if err != nil {
        logger.Error("could not write file %s: %s", file, err)
        continue
    }
```

In commands such as `plakar diff` which require copying the full data to a buffer,
it also makes it possible to rely on existing functions like `ioutil.ReadAll()`:

```go
    buf := make([]byte, 0)
    rd, err := snapshot.NewReader(filename)
    if err == nil {
        buf, err = ioutil.ReadAll(rd)
        if err != nil {
            return
        }
    }
```

In the UI,
this is even nicer as it uses the standard libraries' http server which **uses a Writer for the response to the client**.
Implementing a download endpoint boils down to using the new Reader interface,
then copying it to the Writer **rather than doing the whole fetch object then chunks and copy them in a loop logic**.

Basically,
this simplifies a TON of areas in plakar and allows me to remove a lot of custom code for things that exist in the standard library.


# Fixed handling of max concurrent files

During the last few months,
a lot of work was poured into parallelizing operations in `plakar` to utilize resources as efficiently as possible and speed things up.

This led to some issues when processing large directories as **you could end up opening a very large amount of files**,
sometimes exceeding the number of descriptors available to the process and resulting in EMFILE errors.

My initial thought was that I could simply **implement a backoff mechanism**,
but this is complex and in the case of plakar it doesn't really make sense:
a backoff mechanism is useful as a mean to recover from bursts that cause resources exhaustion.
If plakar is going to parallelize and **constantly hit the backoff mechanism**,
it hints that the solution should be located elsewhere.

Contrarily to a network daemon which could accept a very large number of clients and take advantage of idle times from some to process others,
plakar only opens files when it will actively read or write them,
so **it makes no sense opening files when it can't process them**.
If it opens 1024 files but can only chunk concurrently 32 files before it uses all of the CPU time available,
then having 992 opened files waiting to be processed is pointless.
I decided to **tie the number of descriptors to a factor of the number of cores available**,
currently `2 * n + 1`,
and this ensures that the number of descriptors is low enough not to provoke an EMFILE while high enough to keep cores busy.
The factor may be adjusted in the future but this feels like the proper way to handle this.

This doesn't solve the ENFILE errors,
**where the system descriptor table is full while the process itself has not hit a limit**,
but I'm wondering if it's even worth having a backoff mechanism to retry as I don't see how a backup tool can really recover from a system not being able to reliably allocate descriptors.
I'd personally rather have the backup fail and restart it when the system is in better shape,
than have a backup take a lot more time to complete and then not trust that backup and do another one just in case some failures where not handled properly by plakar.
I'm still not decided on this,
**maybe I'm wrong about it**,
but I need more time to think about it.

Anyways,
with this change I was able to produce large snapshots that previously hit a `too many open files` error,
without observing any significant performances degradation and while still using cores efficiently.


# plakar stdio server

I had mentionned my work on remote repositories [in this post](/posts/2021-04-30/april-2021-opensmtpd-plakar-ipcmsg-privsep-and-a-small-hypnosis-talk/),
but didn't communicate much about it because **it was mainly a proof of concept to validate that no repository primitive expected locality of the repository**,
not a serious attempt at writing the final network mode.

Remote repositories work thanks to a TCP server **accepting the repository primitives and mapping them to a local repository**,
the client is basically **a proxy which doesn't run the primitives locally but passes them to the server and waits for the result**.

Because the server logic is isolated in a client handler function,
I thought it would be nice if I could abstract the network layer and consider that the client handler could work using `stdio` rather than a network connection.
Instead of passing the client handler a descriptor to **a network connection**:

```go
func handleConnection(conn *net.Conn) {
    [...]
}

func Server(repository *storage.Repository, addr string) {
    [...]
    
    go handleConnection(c)
    
    [...]
}
```

I converted it to using `io.Reader` and `io.Writer` interfaces and **passed the same network connection for both**:

```go
func handleConnection(rd io.Reader, wr io.Writer) {
    [...]
}

func Server(repository *storage.Repository, addr string) {
    [...]
    
    go handleConnection(c, c)
    
    [...]
}
```

This worked nicely,
so I took it a step further and created a new entry point `plakar stdio` which would call the client handler but passing `os.Stdin` as the Reader and `os.Stdout` as the Writer.

```go
func handleConnection(rd io.Reader, wr io.Writer) {
    [...]
}

func Stdio() error {
	ProtocolRegister()
	handleConnection(os.Stdin, os.Stdout)
	return nil
}
```

It meant that I could now fork `plakar stdio` from a parent process and **communicate with it through pipes or socketpairs**.
I did a few tests and it worked fine which made me very happy.

How is this any useful, you ask ?


# plakar over SSH

I've been willing to add plakar over SSH support for a while but **couldn't set my mind on the simplest way to do it**.
I did various experiments which were all either too complex or that led to unsatisfying experiences:
if I use SSH,
I want the full SSH experience with known hosts,
authorized keys,
public key authentication,
etc...
so all attempts that involved implementing an SSH client myself either led to missing features or to a TON of code unrelated to plakar.

However...

With `plakar stdio` **things are much simpler** because I can simply use the `ssh` client to spawn a remote `plakar stdio`,
then **take advantage of the fact that the client will do the `stdio` mapping and expose the ssh transport through `stdin` and `stdout` for me**:

```sh
% ssh gilles@backups.poolp.org 'plakar stdio'
```

Using this method,
plakar **does not need to have a daemon running on the remote end**,
it just needs the command installed.

So I implemented the `ssh://` protocol for plakar which **simply forks a process for `ssh`** and emits packets to its stdin while reading responses from its stdout.
Because it uses the `ssh` command,
I **no longer have to worry about known hosts & such**, they are already handled, and I can even benefit from having used `ssh-agent` so I don't have to keep typing my passphrase:

```sh
% plakar on ssh://backups.poolp.org ls                 
2022-04-24T08:18:04Z  9bca77d9    3.1 MB /private/etc
% plakar on ssh://backups.poolp.org push /bin
% plakar on ssh://backups.poolp.org ls       
2022-04-24T08:18:04Z  9bca77d9    3.1 MB /private/etc
2022-04-24T14:54:12Z  02895414     13 MB /bin
%
```

It took me hours to implement various SSH proof of concepts,
people [on the discord channel](https://discord.gg/YC6j4rbvSk) have endured me quite a lot,
but this last version... **took only two minutes to implement** using a few lines of code.

It was both **very satisfying** and **very frustrating** :-)


# Remote creation of repositories

Until yesterday,
using a remote plakar expected you to **create the repository on the remote end before starting to use it**.

I implemented the creation primitives both on the server and client and so it is now possible to do as follows:

```sh
% plakar on ssh://backups.poolp.org create -no-encryption
% plakar on ssh://backups.poolp.org push /bin
% plakar on ssh://backups.poolp.org ls
2022-04-24T15:02:48Z  181702cd     13 MB /bin
%

% plakar on ssh://backups.poolp.org/tmp/plakar create -no-encryption
% plakar on ssh://backups.poolp.org/tmp/plakar push /bin
% plakar on ssh://backups.poolp.org/tmp/plakar ls
2022-04-24T15:03:37Z  8a2ab608     13 MB /bin
%
```

Note that `-no-encryption` is only used here to simplify the examples,
dropping the flag will allow creating encrypted repositories remotely.


# What's next ?

Not really decided on my next tasks as I have multiple areas that need to be improve,
but **a good candidate** is the `plakar sync` command which currently only synchronizes clone repositories (same configuration, either unencrypted or both encrypted with same key).
I'd like to improve it **so it can synchronize snapshots with arbitrary repositories**,
allowing to **synchronize an unencrypted snapshot with an encrypted repository and encrypting chunks on the fly**.

I'm also working on two new projects,
which I won't disclose for now,
so I may be going back and forth depending on the mood.


---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/151](https://github.com/poolpOrg/poolp.org/discussions/151)
