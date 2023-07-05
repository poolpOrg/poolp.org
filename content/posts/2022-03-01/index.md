---
title: "March 2022: plakar clone and plakar sync"
date: 2022-03-06 00:51:00 +0200
authors:
 - "gilles"
language: fr
categories:
 - technology
---

<blockquote>
<b>TL;DR:</b>
implemented cloning and synchronization between plakar repositories
</blockquote>


# Shout out to my sponsors &#x2764;&#xfe0f;

A **HUGE thanks** goes to my sponsors on [github](https://github.com/sponsors/poolpOrg)
and [patreon](https://www.patreon.com/gilles):
your continuous support is very much appreciated !

I created a Discord where I hang out and discuss my projects or screencast as I work on them.
Feel free to hop in if you want,
and feel free to do just like me and share thoughts as you work on your own projects there:
**this is a virtual hack room for anyone to join**: [https://discord.gg/YC6j4rbvSk](https://discord.gg/YC6j4rbvSk)

# Slacked a bit
Shortly after I published my [yearly retrospective](/posts/2021-12-30/farewell-2021-welcome-2022-a-personal-post/),
I was hit with two highly annoying personal issues that kept me very busy these last two months.
I finally got everything back under control,
my mind is mostly free again and I resumed playing with `plakar`.


# plakar clone
I implemented a `clone` operation in `plakar` allowing the creation of a new repository that is **an exact copy** of a source repository.
This is a bit trickier than it reads,
repositories rely on a specific structure as well as hard links magic to properly represent snapshots and **reference counts** on files.
Performing a recursive `cp` of the plakar storage directory would lead to a broken repository unable to properly handle de-duplication.

To implement cloning,
`plakar` creates a new repository from scratch **using the exact same configuration** as the source repository.
It retrieves the list of chunks, objects and snapshots from the source repository,
then for each chunk, object and snapshot it retrieves the data from the source repository and writes it to the destination repository.
Once this is done,
it loads the index file for each snapshot and performs the hard link magic to recreate the references required for snapshots to be complete.
When done,
**both repositories are identical** and the destination one can be used just as if it was the source one.

This required **adding a few primitives** to the storage layer and it was quite frustrating as mistakes didn't show up right away.
A lot of tasks can rely on the snapshot index,
so when a chunk or an object are missing or incorrectly referenced for example,
the cloned repository can still handle a lot of commands as if there was no issue.
This is where `plakar check` came in handy,
letting me know right away that something was off.

```sh
$ plakar create -no-encryption
$ plakar push /bin
$ plakar ls
2022-03-05T22:30:58Z  3a5769ff-3353-4ae9-ae64-13fc0fc0b6ac     13 MB /bin
$ plakar clone /tmp/plakar-copy
$ plakar on /tmp/plakar-copy ls
2022-03-05T22:30:58Z  3a5769ff-3353-4ae9-ae64-13fc0fc0b6ac     13 MB /bin
$ 
```

As you can see in the example above,
**the clone retains snapshots UUID** as this is not a new snapshot being copied from another one in a new transaction,
but really the exact same snapshot copied using lower level primitives.

As of today,
`plakar clone` is not fully finished as it **only supports filesystem-based repository**.
I need to implement the new storage primitives in the SQLite and network backends,
so that cloning can happen **seamlessly over the network or between different backends**.
When this is done,
it'll be possible to convert a filesystem plakar to an SQLite plakar by cloning it (right now, it's only doable the other way),
or to clone a plakar repository on a different machine.


# plakar sync
With `plakar clone`,
all the primitives required to **synchronize two repositories** became available.

Similarly to cloning,
synchronizing allows performing a low level copy of snapshots **without initiating new transactions**,
thus retaining the same snapshots UUID.
To achieve synchronization,
`plakar` opens a source repository and a destination repository,
then compares which chunks, objects and indexes are available in the source one but missing from the destination one to **compute a delta**.
It then performs a cloning of that delta,
**only exchanging the missing bits** and doing the hard links magic.

```sh
$ plakar create -no-encryption
$ plakar clone /tmp/bleh.t1   
$ plakar ls                   
$ plakar on /tmp/bleh.t1 ls   
$ plakar push /bin            
$ plakar ls                
2022-03-05T23:24:08Z  480e6781-4f51-4289-9983-17f0e5962cc9     13 MB /bin
$ plakar sync /tmp/bleh.t1 
$ plakar on /tmp/bleh.t1 ls
2022-03-05T23:24:08Z  480e6781-4f51-4289-9983-17f0e5962cc9     13 MB /bin
$ 
```

In the example above,
I create a new empty repository and clone it.
Then I push `/bin` to the source plakar,
perform a `plakar sync` to the destination plakar,
**which ends up having the same snapshot**.

Like for cloning,
this is a work in progress and not finished.

The idea behind sync is that when the primitives are ported to network plakar,
it becomes possible to have a remote plakar centralize the backups of a local plakar,
but it also becomes possible for multiple remote plakars to synchronize themselves one with another and provide multiple copies of the same backups very easily.


# What's next ?

I'll be working on and off as I have a wedding that's coming soon and a lot of things to prepare,
but this should not prevent me from finding time to relax with some code.

I'll rework clone and sync because functionality set aside **I don't like the way it's implemented in the CLI**.

I'm currently reading the QuickCDC paper and wondering if it's worth implementing on top of my go-fastcdc implementation,
it is unclear if there will be any gain at this point.

I'm considering a few more repository-level operations,
similar to clone/sync,
to help make plakar as friendly as possible to operators.

It will be time to start working on a website too :-)


---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/149](https://github.com/poolpOrg/poolp.org/discussions/149)
