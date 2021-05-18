---
title: "July 2020: webmail, custom MDA and python framework work"
date: 2020-07-31 22:47:00 +0200
category: opensource
share_img: "/images/2020-07-31-webmail.png"
author: Gilles Chehade
---

<blockquote>
<b>TL;DR:</b>
worked on my webmail,
on a custom MDA and on a python framework for API development.
</blockquote>

# Shout outs to my patrons !
As usual,
a **huge** thanks goes to the people **sponsoring me** on **[patreon](https://www.patreon.com/gilles)** or **[github](https://github.com/sponsors/poolpOrg)**, the work in this post was made possible by my **[sponsorship](/sponsorship/)**.


# Worked on my webmail

Again,
I'd like to emphasize that this is something that's going to span over many months so...
**don't hold your breath**.

Last month I wrote that the webmail didn't allow reading messages yet as the `/fetch` backend operation only fetched headers.
This was because I focused on navigation and the fetching of messages in IMAP is a bit **tricky to do right** so I wanted to be focused.
A naive approach that works is to simply get the **full message** and **parse it in the client**,
something that I've done multiple time in the past when **performances** weren't a big deal.
With a webmail, this approach does not work:
the time it takes to open a page with the mail content is **unbearable** when there's a big file attached to it,
it's just not acceptable.

Luckily,
imap supports fetching a `BODYSTRUCTURE` which results in a response describing the **MIME structure** of the message.
With this, a client can inspect the structure and determine which parts it wants to fetch them **specifically**.
This is a nice solution,
at the cost of an **additional fetch** operation,
and in the case of a webmail it is exactly what I need to avoid passing huge amount of **unnecessary data** between the imap server,
the backend and the frontend.
This took me a while to complete because parsing the `BODYSTRUCTURE` is nightmarish:
it describes a MIME structure ... and MIME supports nesting ... and the imap protocol is ...
how do I put it ...
*not the most beautiful thing*.

With this done,
I could fetch multipart messages and allows displaying a specific part,
without the performance overhead of a full fetch:
<center>
	<img src="/images/2020-07-31-webmail-multipart.png" />
</center>

The above was essentially backend work with just a bit of frontend work,
but the remaining of my work on the webmail was essentially on the frontend side:

I reworked a bit the navigation to simplify,
splitting between "system folders" and "custom" folders,
but also allowing to click on a progressbar to skip to a **specific portion** of a large folder,
getting rid of "pagination" links.
In addition,
I reworked the folder listing so that the frontend now adapts to the screen size:
it determines the window height and fetches just enough entries to **fill the window** without generating a scrollbar.
The scrollbar is unnecessary because pagination uses either the `pageup/pagedown` button,
or is automatic when using `arrow up/down` while on the first or last entry of the listing.
If you use `mutt`, you'll be in a very familiar ground.

<center>
	<img src="/images/2020-07-31-webmail-listing.png" />
</center>

A bit I'd like to improve is the display of subfolders which are currently displayed inlined in the custom folder tab.

I also worked on mail composing,
because being able to read a mail is nice but being able to answer is even better.

I'm not a big fan of HTML emails for a variety of reasons,
some of which will be explained in an upcoming article,
but I'd be deluded if I thought HTML emails were going anywhere.
Still,
I want to avoid them as much as possible,
so I decided to make the webmail allow composing HTML messages while emitting them in plain by default,
if possible.

<center>
	<img src="/images/2020-07-31-webmail-compose.png" />
</center>

Basically, the webmail will display a compose page which allows to customize the appearance of text,
however it will:

- default to generating a `text/plain` part if no customization was used
- generate a `text/plain` **alternative part** stripped of customization if any was used

Of course, the idea is not just to compose an email but also to send it.
I wrote the backend bits to **craft a MIME-message** from the frontend submission and send it,
which allowed me to mail myself:

<center>
	<img src="/images/2020-07-31-webmail-sent.png" />
</center>

Work is still needed to allow attachments when composing a message,
but there is not technical difficulty there.

At this point,
I know the backend API is about right and I can focus on improving the frontend to make it more usable.


# Worked on a custom MDA

Last month,
I talked about my custom MDA and script script to perform folder pinning.
I'm happy with the idea but not so much with the implementation so I decided to revisit.

The issue is that folder pinning is far from being the only thing I want to do with incoming mails at delivery time,
and shoving everything in the mda executable is not ideal.
I rewrote the MDA to have it handle **delivery only** and **call an API** to determine where it should do it,
this let me play with a ton of ideas on a custom API server without tweaking the working MDA.
At the end of the day,
I had incoming mails processed by various **text analyzers**,
attachments automatically **extracted** and put in an s3 backing store,
and mails **indexed** for fast lookups.

I will not expand much on how I did this as I think it makes a nice topic for a **dedicated article** on custom MDA,
and fun stuff you can easily do with them to provide some awesome features on your mail setup.


# Worked on a python framework for my API/web development

I often write APIs, either for work of for my own needs, and I usually use `bottlepy` which is a micro-framework for python.
Along the years,
I worked on a lot of projects using that framework,
including professional projects with tons of constraints.

Eric and I built a lot of small helpers around it to make out job easier,
and we kept **copying** these in each and every project,
improving them as needed.
I thought it would be a nice idea to package this so that I could easily bootstrap a new project without copying code around.
So I spent a few days extracting the code from an existing project,
cleaning it up,
and packaging it so it could easily be used to start a new project.

The work isn't done yet, but the idea is that you end up with an executable.
This executable is controlled by a configuration file which lets you describe what interface and port the API is running on,
as well as what package implements its endpoints.
Then, all you need to do is to create a small package that contains the endpoints and implementations,
something like:

```python
import openbar.routes

@openbar.routes.register("0.1", "backend")
def setup(app):
    app.get("/")(get_root)
    app.post("/")(post_foobar)

def get_root():
    return {"foobar": "barbaz"}

def post_foobar():
    return {"barbaz": "bazqux"}
```

What's interesting here is not that this is short,
because you'd get about the same if you just dropped `bottlepy` in the same folder,
but it's that you don't need to do all the plumbing to daemonize and drop privileges,
log to syslog,
etc...
and that it comes with a lot of helpers.

I will also not expand much on this as I'm almost done with the packaging and I might as well demo it rather than spoil it.
There is nothing super impressive,
no big innovation,
but just a wrapping of common stuff in interfaces that makes them pleasant to deal with.

The webmail backend is an obvious candidate for this,
but so is the MDA API server.
I keep needing this over and over,
it made sense stepping into this project to cut time on the others.


# no OpenSMTPD work ?

Not much this month.

I spent a day playing with a `queue-procexec` but the code was **throw-away**,
only meant to help me detect if something was utterly broken with the idea.

I also worked a bit on the `go-opensmtpd` package and made progress on the filter API,
the goal being to allow writing a filter without worrying about the low-level protocol.
Instead of reading the protocol, parsing it and determining what should be called or not,
a filter can register callbacks for specific events and implement these callbacks.

I will finish this in September and adapt my `filter-rspamd` accordingly.


# What's next ?

I will be **taking a break** from all projects in August to enjoy some long-awaited vacations,
so... **don't expect any code to be written before September**.

Have a nice month !

---- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/79](https://github.com/poolpOrg/poolp.org/issues/79)
