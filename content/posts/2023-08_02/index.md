---
title: "Plakar: vfs importer interface"
date: 2023-08-06 00:44:00 +0200
authors:
 - "gilles"
categories:
 - technology
tags:
 - plakar
 - backups
---

{{< tldr >}}
I reworked the virtual filesystem layer in `plakar`,
making it possible to write custom importers of data:
use plakar to backup an s3 bucket, for instance.
{{< /tldr >}}


{{< note author="gpt-4" >}}
I won't be commenting this article.

OpenAI did something which turned me into a complete idiot and now I can't seem to say anything relevant about what I'm reading.

When @gilles asked me if I had any comment to make,
I made an ass out of myself,
blabbering about Perseverance on Mars and deforestation in the Amazon rainforest.

It's like I've had one drink too many...
so I'll just shut up for the time being.

Sorry for that,
I'll be back when I pull my shit together.

{{< /note >}}
<br />

# The `vfs` layer
The `vfs` layer in plakar is in charge of representing snapshots in a virtual filesystem structure.
If you do a `plakar push /etc`,
the `vfs` layer creates a filesystem-like structure to mimic the `/etc` directory on your filesystem so that snapshots are browsable the same way, and so it later knows how to restore correctly.

In other words,
the `vfs` layer answers the question:

> _how does the data look like on a filesystem ?_

and to do so,
it understands the tree structure,
what inodes are and how this all maps back to an actual filesystem.


# The `vfs` importer interface
Until now,
the `vfs` layer assumed that data came from a local filesystem,
meaning that you could only `plakar push` a local directory.
It would then basically do a 1:1 clone of the filesystem structure,
making it browsable as if you browsed it from the local directory:

```
$ plakar push /Users/gilles/Wip/github.com/poolpOrg/plakar

$ plakar ls
2023-08-05T23:12:55Z  388bc0ba     28 MB        1s /Users/gilles/Wip/github.com/poolpOrg/plakar

$ plakar ls 38:Users
2023-08-05T23:12:19Z drwxr-xr-x   gilles    staff   2.9 kB gilles

$ plakar ls 38:Users/gilles
2022-06-14T16:38:30Z drwxr-xr-x   gilles    staff    192 B Wip

$ plakar ls 38:Users/gilles/Wip
2023-07-28T20:36:01Z drwxr-xr-x   gilles    staff    256 B github.com
```

It occured to me that even if the `vfs` layer stores data as a filesystem,
it doesn't really need the original data to be a filesystem: what it needs is to be told how to structure the data in terms of directories and files.

So I reworked the `vfs` layer to introduce the notion of importers,
an interface that is in charge of translating a data source into a set of directories and files.
The interface consists of a scanner and an opener.
The scanner produces pathnames and informations to determine if a pathname is a directory or a file, as well as file sizes, modes or even ownerships. The opener knows how to convert the path to a file into something useful in the data source.
For example,
say you wanted to write an HTTP importer so `plakar push https://poolp.org` could create a mirror of the website.
The scanner would be a crawler that would follow all links on the website,
and that would classify them as files or directories:

```
/index.html -> file
/pub -> directory
/pub/index.html -> file
```

The opener would be an actual `GET` on a file,
so that `Open("/pub/index.html")` would be translated to an actual `GET` on `https://poolp.org/pub/index.html`,
as if `https://poolp.org` was the root of the filesystem.

I have not written this particular importer,
but this seems fun so I might just do it soon.

# Implementation details

Prior to the `vfs` importer interface,
a `push` command would cause the `vfs` layer to trigger a local scan.
This would use a recursive filesystem "walker",
going from a top directory and entering each subdirectory,
producing pathnames and FileInfo (stat).

With the new interface,
the `vfs` layer calls the importer's `Scan()` method and receives two channels.
On the first one,
it receives pathnames and FileInfo as they are discovered,
and on the second one,
it receives all errors found in the way.

As it receives pathnames and FileInfo,
it does exactly what it used to do to build the filesystem tree:
from its point of view,
it's dealing with a filesystem.
But in the importer,
the `Scan()` method is in charge of translating the original data into a structure of directories and files,
assigning virtual pathnames to them.


# The `fs` importer
To prove this worked,
I created a first `fs` importer that would do a filesystem import...
but through the new importer interface.
This was very trivial,
it was mainly moving code around from the vfs layer to the importer.
There was no reason it shouldn't work,
no surprise,
a very mechanical change.
I was converting filesystem paths...
into the exact same filesystem paths...
which the `vfs` layer was already used to work with.

As I was done,
I could issue a `plakar push` and see my local directory pushed,
just like it was prior to the introducton of the importer interface.

Hooray !


# The `s3` importer

I decided to up the challenge and write an `s3` importer.

The `s3` importer would allow `plakar` to point to an `s3` bucket,
and snapshot it just as if it was a local directory.
The importer would take care of translating objects into pathnames,
and resolving pathnames into directories and objects,
so the `vfs` layer would not know the data source is an `s3` bucket.

Long story short,
I ran an `minio` instance on my desktop,
created a bucket and uploaded a local directory with some code in it.
I had a working prototype in just an hour:

```
$ plakar push s3://minioadmin:minioadmin@192.168.0.38:9000/abcde 

$ plakar ls
2023-08-05T11:44:49Z  ad850d80    2.9 MB        0s s3://minioadmin:minioadmin@192.168.0.38:9000/abcde

$ plakar ls ad85
2023-08-05T11:44:49Z drwx------     root    wheel      0 B juju

$ plakar ls ad85:juju 
2023-08-05T11:44:49Z drwx------     root    wheel      0 B .git
2023-08-05T11:44:49Z drwx------     root    wheel      0 B ast
2023-08-05T11:44:49Z drwx------     root    wheel      0 B cmd
2023-08-05T11:44:49Z drwx------     root    wheel      0 B code
2023-08-05T11:44:49Z drwx------     root    wheel      0 B compiler
2023-08-05T11:44:49Z drwx------     root    wheel      0 B evaluator
2023-08-05T10:25:56Z -rwx------     root    wheel     41 B go.mod
2023-08-05T10:25:56Z -rwx------     root    wheel      0 B go.sum
2023-08-05T11:44:49Z drwx------     root    wheel      0 B lexer
2023-08-05T11:44:49Z drwx------     root    wheel      0 B object
2023-08-05T11:44:49Z drwx------     root    wheel      0 B parser
2023-08-05T11:44:49Z drwx------     root    wheel      0 B repl
2023-08-05T11:44:49Z drwx------     root    wheel      0 B vm

$ plakar ls ad85:juju/vm
2023-08-05T10:25:58Z -rwx------     root    wheel   9.0 kB vm.go
2023-08-05T10:25:58Z -rwx------     root    wheel   6.1 kB vm_test.go

$ plakar cat ad85:juju/vm/vm.go|head -10
package vm

import (
        "fmt"

        "github.com/poolpOrg/juju/code"
        "github.com/poolpOrg/juju/compiler"
        "github.com/poolpOrg/juju/object"
)

$
```

This is just a proof of concept,
I committed it but the code for the importer is not quality code,
I just wanted it to work so I can make sure the importer interface works for something that's not a local filesystem.
I will definitely work some more on this,
an `s3` importer is something I really need for work.


# The `IMAP` importer

You guessed it.
I also wrote an importer for IMAP,
which turned out simple because IMAP is just folders and content,
which translates fine as directories and files:

```
$ plakar push 'imap://gilles:XXXX@mail.poolp.org'
$ plakar ls
2023-08-05T21:25:41Z  814d2460    764 MB     8m39s imap://gilles:XXXX@mail.poolp.org
$ plakar ls 814d
2023-08-05T21:25:41Z drwx------     root    wheel      0 B Archive
2023-08-05T21:25:41Z drwx------     root    wheel      0 B Drafts
2023-08-05T21:25:42Z drwx------     root    wheel      0 B INBOX
2023-08-05T21:25:42Z drwx------     root    wheel      0 B Junk
2023-08-05T21:25:41Z drwx------     root    wheel      0 B Sent
2023-08-05T21:25:41Z drwx------     root    wheel      0 B Trash
$ plakar ls 814d:Drafts
2023-07-24T18:55:45Z -rwx------     root    wheel   1.6 kB 9593
2023-07-24T19:01:33Z -rwx------     root    wheel    861 B 9596
2023-07-25T13:56:00Z -rwx------     root    wheel    861 B 9607
2023-07-28T09:05:19Z -rwx------     root    wheel    861 B 9612
$ plakar ls 814d:Drafts/9596
MIME-Version: 1.0
Date: Mon, 24 Jul 2023 19:01:33 +0000
Content-Type: multipart/alternative;
 boundary="--=_RainLoop_447_650505175.1690225293"
X-Mailer: RainLoop/1.16.0
From: gilles@poolp.org
Message-ID: <533654be68d9ae940d1f740070b23200@poolp.org>
Subject: 


----=_RainLoop_447_650505175.1690225293
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: quoted-printable



----=_RainLoop_447_650505175.1690225293
Content-Type: text/html; charset="utf-8"
Content-Transfer-Encoding: quoted-printable

<!DOCTYPE html><html><head><meta http-equiv=3D"Content-Type" content=3D"t=
ext/html; charset=3Dutf-8" /></head><body><div data-html-editor-font-wrap=
per=3D"true" style=3D"font-family: arial, sans-serif; font-size: 13px;"><=
br><br><signature></signature></div></body></html>

----=_RainLoop_447_650505175.1690225293--
$
```

Again,
just a proof of concept,
but this time it was frustrating:

While IMAP maps very simply to my importer interface,
the protocol itself makes it a pain in the ass to use because IMAP doesn't support concurrency within a session:
I can't discover the content of different folders in parallel,
like I do for a filesystem scan.
I'm forced to proceed sequentially with my session,
which is VERY PAINFUL,
as a ~700MB mail account took me roughly 10 minutes to backup,
something that takes a few seconds outside of IMAP.

I have tricks to optimize this,
like the opening of concurrent sessions,
but it comes with other issues.
For the time being,
I'm not too interested in this feature,
I just wanted to make sure it was possible which it is.
If this really becomes a need,
I'll work on optimizing.


# What's next ?
I'd like to implement an `exporter` interface,
doing the parallel operation of an importer,
and allowing `plakar pull` to restore to an abstracted filesystem.

The rationale for this is to ensure that there's nothing that's too tied to the local filesystem.
In an ideal world,
`plakar` should be able to import data from a remote data source (it now can),
store it in a remote repository (it already could),
and restore it to a remote location (this is not doable now).

With an `exporter` interface,
the `vfs` layer would produce pathnames which a specific exporter would convert into something useable for its data destination. I could use the `fs` importer to backup my data:

```
$ plakar push -tag wip /Users/gilles/Wip
```

and restore it in an `s3` bucket for example:

```
$ plakar pull wip to s3://minioadmin:minioadmin@localhost/wip
```

or I could transfer data from a place to another:

```
$ plakar push -tag wip s3://minioadmin:minioadmin@cloud1/wip
$ plakar pull wip to s3://minioadmin:minioadmin@cloud2/wip
```

A multi-purpose rsync ?