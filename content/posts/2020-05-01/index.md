---
title: "April 2020: worked on a webmail and a bit of OpenSMTPD too"
date: 2020-05-01 11:37:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
Worked on my <mark>webmail</mark>;
Did a bit of <mark>OpenSMTPD</mark> work;
{{< /tldr >}}

# Shout outs to my patrons !
As usual,
a **huge** thanks goes to the people sponsoring me on [patreon](https://www.patreon.com/gilles) or [github](https://github.com/sponsors/poolpOrg), the work in this post was made possible by my [sponsorship](/sponsorship/).


# Webmail vs mutt

I started using console clients to read my mail back when I was a student in early 2000s,
and I've been using `mutt` for as long as I can recall now.
I installed various mail clients along the years,
some with graphical interfaces such as `thunderbird` or `sylpheed`,
others with a web interface such as `nullWebmail`, `SquirrelMail`, `Roundcube`, `Mailpile` or currently `Rainloop`.
No matter which,
I always end up falling back to `mutt` for most tasks.

<center>
	<img src="2020-05-01-mutt-1.png"/>
</center>

Then why don't I stick with `mutt` and stop trying to use other ones ?

Wether I like it or not,
a lot of mails that are important to me contain MIME-parts that are just not comfortable to deal with in console.
There are ways to work-around,
like having `lynx` render HTML mails, `mupdf` be launched for PDF attachements, and so on...
but this is not comfortable enough (to me).
I keep using `mutt` for a large part of my mails and keep some unread so that I can read them from another client.

Ok, so why don't I stick with other ones and stop using `mutt` ?

I want to be able to read my mail from various places and not rely on a heavy client.
This leaves me with console-based clients that I can `ssh` to,
and webmails that I can hit from any browser.
Unfortunately for me,
most webmails have interfaces that are inspired by heavy clients with framing and pagination,
something that's not practical with my use of mails.

<center>
<img src="2020-05-01-webmail-0.png"/>
</center>

If I want to cleanup the `List` folder which has accumulated too many mails,
I'd rather use `mutt` which lets me scroll through the entire folder and delete mails with a key press,
rather than rely on this user interface to paginate through 81 pages.
This particular webmail has some keyboard shortcuts which makes it less painful than some others,
but still,
I have to scroll down to see the list of mails for this page and skip to the next pages to see other mails in the folder.
Because of this,
I will simply never use that interface to do this task and fall back to `mutt` which lets me do in a couple minutes.


## My 2012 attempt at a webmail
In 2012,
I wrote a small webmail for my use in `python` with the `bottlepy` mini framework,
with the intent to release it opensource.

I had not yet understood what caused me not to use them on the long run,
but I knew I didn't work along with traditional webmail interfaces.
So it was mostly keyboard based and had a different look than others:

<center>
<img src="2020-05-01-webmail-1.png"/>
</center>

It also had some features I thought were interesting for my own use-case,
like for example the use of [robohash](https://robohash.org) to immediately spot mails originating from the same sender,
the detection of diffs and their highlighting to make them more readable,
or a `show analysis` button which allowed me to display a ton of technical details about a particular sender and mail:

<center>
<img src="2020-05-01-webmail-2.png"/>
</center>

Unfortunately,
I got carried by **${DAYTIMEJOB}** and gave up work on it before it became usable for real.
It is a mail viewer with limited writing abilities.


## My 2020 attempt at a webmail

We're now 8 years later and I still want to scratch that itch that annoys me since then.
I think my past effort had value but it didn't tackle the issue correctly,
and with 8 additional years of mail experience and nuisance,
I think I can come up with something that suits me better.

My knowledge and the technologies available are not the same than in 2012.
I'm sure I could have achieved something very usable back then but definitely not something comfortable.
Part of what makes `mutt` comfortable is its interface,
but the other part is the fluidity of being able to scroll a folder of thousands of mails.
My 2012 webmail didn't prefetch mails, didn't do any kind of caching and did everything synchronously.
Sure I could skip from a page to another with the press of an arrow key,
but I then had to wait for a command to be sent to the IMAP server and for the answer to come back before the page actually changed.
It was not really slow but it was definitely not really fluid.

I decided to rewrite it from scratch,
still with the intent of publishing it opensource.

Below is a screenshot from my initial work,
start this week:
<center>
<img src="2020-05-01-webmail-3.png"/>
</center>

The webmail is now split into a backend and a frontend.
The backend is written in `python` + `flask` and exposes a REST API to the IMAP server,
whereas the frontend is written in `react` (which I had never used before this Monday) and only implements the rendering.
The reason for this split is that I intend to write custom frontends too.
This is too birds with one stone.

The screenshot is for an early work in progress so this is a "developer" interface more than a user interface.
The idea here is that I have a mail listing with a cursor controled by my arrow keys,
scrolling down and up will paginate mail by mail whereas left and right will paginate by full pages.
`Delete` or `d` keys will move a mail to `Trash`, while `Enter` will display the mail viewer.
`Space` will select the mail that's below the cursor and add it to a selected list so that `Delete` will allow bulk deleting for example.

Of course, I will make it easier to use for non-techies but my main goal is that I'm able to find the same comfort as `mutt`,
THEN add buttons so that people who are more into using a mouse can find their ways.

I'll continue working on this every now and then until I have something that I can put up a demo for,
then when I'm happy enough I will release the code for backend and frontend.
Right now,
having no experience with `react` causes me to do a lot of write and throw-away code,
there's no point in sharing code at this point.


# Preparing OpenSMTPD 6.7.0

OpenBSD 6.7 is around the corner and OpenSMTPD 6.7.0 by the same occasion.

As I [explained last month](/posts/2020-03-30/anxiety-openbsd-break-covid-19-and-resuming-work/),
I no longer contribute directly to OpenBSD as a committer but I didn't ghost and helped review several diffs.


## Bug fixes

As with any release cycle,
people wake up right before a release with annoying bugs.

This time it was three bugs.
One was a bug in the aliases expansion code and the
[handling of virtual aliases catchalls](https://github.com/OpenSMTPD/OpenSMTPD/issues/1053)
where the `@` catch-all incorrectly caught recipients in some setups.
Another was a shortcoming in the [order of fields of two events](https://github.com/OpenSMTPD/OpenSMTPD/issues/1044)
in the filtering protocol that required a protocol change and version bump.
The last one was a [buggy root-restricted command](https://github.com/OpenSMTPD/OpenSMTPD/issues/1046)
causing the server to halt due to a `fatal()`.

Aliasing code is particularly tricky because it doesn't just map a recipient address to a user,
it can map a recipient address to multiple users spanning over multiple domains that may or may not use aliases.
It's not a `key => value` but a `key => tree` resolution where each node has a different expansion context.
To make things harder,
the resolution can't block as it would prevent concurrent sessions from working,
so the whole expansion code is an interruptible-resumable loop which makes code simple to read but complex to follow.
**The first reaction to a bug affecting aliases is always denial** because even simple fixes require a lot of thinking.
`eric@` and I confirmed and discussed the issue,
we found a fix very fast that we were both sure worked,
but we had to make sure that it didn't have side-effects by testing multiple scenarios.

The shortcoming in the filtering protocol was easily fixed by swapping two fields but required a protocol bump,
so this also needed discussions to make sure the fix was really the one we wanted and not come up with another one a month later.
This also required planning for how existing filters would need to be adapted to support both 6.6.0 and 6.7.0 transparently.

The last issue `eric@` fixed alone, I only reviewed the diff.

All the fixes were committed to OpenBSD and synchronized in portable,
so they will not be present in the 6.7.0 release.

## Portable

The portable version of OpenSMTPD also needed a specific fix after
[Denis Fateyev](https://github.com/dfateyev), package maintainer for Fedora,
and [Ryan Kavanagh](https://github.com/ryanakca), package maintainer for Debian,
reported that it [failed to build](https://github.com/OpenSMTPD/OpenSMTPD/issues/1051) with newer `gcc-10`.
I worked on a fix which was committed to the portable version allowing it to build again and,
while at it,
added a `gcc-10` target to our CI so we'll detect breakages there too.

Given that I will spend more time working in the portable branch from now on,
I also took time to set myself a proper docker-based development environment to test build and fixes on various systems.

## Github history

Finally,
I decided to tackle an issue that's been around since forever.

The history for the project in our [Github](https://github.com/OpenSMTPD/OpenSMTPD) mirror was shit.
OpenBSD is the main repository and uses `cvs`,
we synchronize code in `git` by diffing the checkout from `git` and the checkout from `git`...
then applying the diff to `git` with a commit log saying `sync`.

As a result,
our `git` repository contained detailed commit logs for portable-specific commits,
but whenever trying to track a specific commit originating from OpenBSD...
you could not find it by commit log and you had to find the `sync` commit which contained the relevant diff.

I managed to bring back the entire OpenSMTPD history from `cvs` to our mirror,
so you can now track individual OpenBSD commits there instead of `sync`.

Here's the [first import](https://github.com/OpenSMTPD/OpenSMTPD/commit/bcc02bda06c866c719adce97668b373e00d8ff11#diff-b67911656ef5d18c4ae36cb6741b7965) commit of OpenSMTPD into OpenBSD,
which we didn't have before.

More interestingly,
here's the [sync](https://github.com/OpenSMTPD/OpenSMTPD/commit/033cd161f809bf756024e36d5250bdf88b91892c) commit
for the fix to the aliasing bug discussed above... which also contains various other commits to unrelated stuff,
and here's the [actual commit](https://github.com/OpenSMTPD/OpenSMTPD/commit/d48edb0478f1e77ff7fd50177b582d0c8e0c2c18) to OpenBSD.


# What's next ?

I have switched my part-time scheduling so that instead of a week every month,
I have a day each week + 1 day each month to work on my projects.
This will be the same amount of time but spread over the month.

I'll spend most of my spare time in May preparing the actual release of portable OpenSMTPD 6.7.0,
but also to work on my webmail.

I also have various branches I want to submit to OpenBSD for review after the 6.7.0 release,
and work to add support to ECDSA in portable OpenSMTPD when OpenSSL is used instead of LibreSSL.

If you're interested in my work,
consider supporting me on [patreon](https://www.patreon.com/gilles) or [github](https://github.com/sponsors/poolpOrg).

---- 
