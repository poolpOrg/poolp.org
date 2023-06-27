---
title: "October 2022: blog comments, a bit of plakar and the streamchain project"
date: 2022-10-18 11:11:00 +0200
authors:
 - Gilles Chehade
language: fr
categories:
 - technology
---

<blockquote>
<b>TL;DR:</b>
added comments support to the blog, did some music and hypnosis projects, fixed a few bugs in plakar and began a new toy project.
</blockquote>

# Shout out to my sponsors &#x2764;&#xfe0f;

A **HUGE thanks** goes to my sponsors on [github](https://github.com/sponsors/poolpOrg)
and [patreon](https://www.patreon.com/gilles):
your continuous support is very much appreciated !

I created a Discord where I hang out and discuss my projects or screencast as I work on them.
Feel free to hop in if you want,
and feel free to do just like me and share thoughts as you work on your own projects there:
**this is a virtual hack room for anyone to join**: [https://discord.gg/YC6j4rbvSk](https://discord.gg/YC6j4rbvSk)

This is a fairly open server where you can ask random questions about a lot of topics,
not necessarily related to my own work.



# Table of content

- [Learnt myself some Swift](#learnt-myself-some-swift)
- [Added table of content and comments support to the blog](#added-table-of-content-and-comments-support-to-the-blog)
- [Music-related work](#music-related-work)
- [Hypnosis-related work](#hypnosis-related-work)
- [A bit of plakar work](#a-bit-of-plakar-work)
- [A new project: Streamchain](#a-new-project-streamchain)


# Learnt myself some Swift
Got myself a book on the Swift programming language,
spent a couple hours reading it and I really enjoyed it.

I was mostly curious,
I don't have anything I want to do with it at the moment,
but it seemed really nice as **it kind of looks like a mix between C, Golang and Python**,
taking bits I like from the three of them.

We'll see if something comes out of it for me in the future :-)


# Added table of content and comments support to the blog

For the last few years,
this blog has used a static generator **so that I could write the posts in Markdown and publish by committing to a repository**.
This is nice because the website is standalone and I can easily move it around from machine to machine,
but **it has prevented me from providing comments** and I ended up relying on [Github discussions](https://github.com/poolpOrg/poolp.org/discussions) and adding a link to the proper discussion below each article I write.

I do get feedback from people,
but **at the exception of some articles that were referenced on popular websites**,
these feedbacks tend to happen through other means like private messages on Twitter or discord.
Maybe these people wouldn't leave a comment on the blog if it was possible,
but maybe it's just because having to follow a link and comment elsewhere is not practical and does not keep the article available with a scroll.

So I found a nice javascript library that **allows interfacing with the Github discussions API**,
and now each article has... <a class="far fa-comment" href="#comments"> its own comment section</a>.
It is now possible to comment directly at the bottom of each article,
the comments reflect what was discussed directly on Github and are published there.

While at it,
I figured it was possible to add a table of content to my articles so you can jump to specific sections.
**I have not added it to previous articles**,
though I might for some,
but **from now on I'll make sure to always include one** as I had lengthy articles in the past and it'll ease reading.

Feel free to comment this article and test :-)


# Music-related work

**I enrolled in a course on music production** that will start in March 2023 at a local studio in Nantes,
so I spent some time playing with Logic Pro X to be more familiar with it and not begin the course as a complete newbie with regard to tools.

These are four LoFi remixes I worked on by importing midi tracks,
altering instruments, scores and tempo,
and adding some effects.
Definitely not fantastic but I do like the result for some so I'm sharing:

|||
|:----------:|:-------------:|
| <center><iframe width="560" height="315" src="https://www.youtube.com/embed/SmUXQqVCXz4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> | <center><iframe width="560" height="315" src="https://www.youtube.com/embed/sQCbIUBPpj8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> |
| <center><iframe width="560" height="315" src="https://www.youtube.com/embed/6BshV_sLBCo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> | <center><iframe width="560" height="315" src="https://www.youtube.com/embed/r5FZRrXknLw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> |

Let me know if you like them,
I have created [a specific playlist](https://youtube.com/playlist?list=PLPkj9YHXkB0G9UbuT9SuEYvBQwJUL_3CU) on [my youtube channel](https://www.youtube.com/c/GillesChehade) for these experiments,
you can subscribe, comment and give thumbs up :-)


# Hypnosis-related work

I haven't spent much time at the hypnosis office since I came back from holidays,
only performing a few sessions here and there,
mostly due to dedicating time to my son as he entered kindergarten and I need to adjust to his schedule.

So I decided to spend some time **working on a podcast** in French to explain how hypnosis works at low-level.
I have recorded the first three episodes of 8 and I intend to release roughly 1 per month,
with next episode due in late November or early December as I'll present a conference in November and won't have time to record.

|||
|:----------:|:-------------:|
| <center><iframe width="560" height="315" src="https://www.youtube.com/embed/r5zG-0J4h_c" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> | <center><iframe width="560" height="315" src="https://www.youtube.com/embed/-q-PsMYuNQo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> |
| <center><iframe width="560" height="315" src="https://www.youtube.com/embed/GgybeD9mfF4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center> | |

As a side-note,
I found it enjoyable to record so **I might create a podcast related to computer stuff in the future**,
when I'm done with this one.



# A bit of plakar work

First,
**I fixed a crash that happened when trying to list content of a path that consisted in a single directory and not starting with /**.
Basically,
there is no such thing as a "relative" path in plakar as they are always relative to the snapshot and therefore to the root of a snapshot,
so `def:etc` is really `def:/etc` as far as plakar is concerned.
Because of how paths are handled,
by splitting pathnames into atoms and recreating a filesystem view,
this confused plakar about what was the root directory and caused it to crash.
**I fixed the function that splits a snapshot path** (ie: `<snapshot>:<pathname>`) into prepending a slash to the pathname portion whenever there isn't,
which **should not only fix the crash I observed but also all commands that accept snapshot paths**.


Then,
I spotted a more annoying bug where **restoring a snapshot results in a partial restore with missing files**.
I was concerned at first because **the last thing I want in a backup tool is for a snapshot to miss files**,
however after a bit of testing I could verify that **they weren't really missing as they could all be restored individually**,
this pinpointed to a bug in the `pull` primitive which seems to miss some restorable files.
I spent several hours trying to understand the issue but couldn't figure out why it happens,
except that **it happens mostly with snapshots containing a lot of files and when `pull` is parallelized**.
It doesn't happen,
or at least I could not reproduce,
when working with a small snapshot or when `pull` is sequential.
**I removed concurrency for the time being** but this is not practical,
sequential restore of a large snapshot is dead slow,
I'll continue tracking the issue.

**I also removed the `gzip` support that I added in `plakar cat` around May**,
and moved it to a dedicated `plakar gzcat` command.
Back then it seemed like a good idea to me that `plakar cat` transparently inflated compressed output at it allowed to `plakar cat` a compressed log,
but after some thinking **I'd rather provide that through a dedicated command** as it allows using `plakar cat` to redirect a compressed stream to a file,
something that was no longer doable.

Finally,
**I changed the way `plakar checksum` works and implemented a `-fast` option**.
The command allows printing the sha256 checksum of a file contained in a snapshot,
and since I already had the checksum in the snapshot INDEX,
the command **didn't recompute it but used the recorded checksum** instead.
**I changed it so that it recomputes the checksum by default**,
reading the file chunk by chunk,
and only resorting to the recorded checksum when the `-fast` option is used.
In practice,
this should always produce the same result unless the plakar store has corrupted chunks.

And that's all for plakar !



# A new project: streamchain

So this is a TOY project,
nothing serious,
I'm just **experimenting with an idea**.

Basically,
**I wrote a tool which lets me create a "twitter-like" stream** where I can post short messages,
however **instead of being centralized on a server** they are **stored in a standalone structured file**.

The file is structured in a way that allows verifying both **authenticity and integrity**,
but also allows for **fast random access** to any message **without having to read the entire file**.
A streamchain reader can validate that the streamchain file **originates from its owner**,
it can validate that it **has not been tampered with**,
and can **seek to any message in constant time** regardless of the streamchain size.

As a result,
it can be hosted anywhere from an s3 object storage to a static web server,
and if the storage supports range queries,
like pretty much all HTTP servers... then all the better.
A streamchain client reader that can talk HTTP can **either mirror a streamchain locally and perform range queries to synchronize updates**,
or it can **consume the streamchain remotely without downloading it entirely**.

I have a working PoC but I'm not ready to publish the code yet,
so I'll likely write an article about it when I open the repository publically.


# What's next ?

A conference on hypnosis for psychology students in November,
so probably not much happening in the next two weeks as I prepare for it.

Then,
I'll try to get streamchain in shape to open the repository.

Feel free to [join my discord](https://discord.gg/YC6j4rbvSk),
engage on [twitter](https://twitter.com/poolpOrg) or 
support me on [github](https://github.com/sponsors/poolpOrg) or/and [patreon](https://www.patreon.com/gilles).

Take care,
stay tuned,
I'll post as soon as I resume my work !
