---
title: "March 2023: Melodya, MHL, MIDI-csv and more..."
date: 2023-04-02 17:29:00 +0200
authors:
 - Gilles Chehade
language: en
categories:
 - technology
---

<blockquote>
<b>TL;DR:</b>
I played with MIDI and ChatGPT.
</blockquote>


# Shout out to my sponsors &#x2764;&#xfe0f;

A **HUGE thanks** goes to my sponsors on [github](https://github.com/sponsors/poolpOrg)
and [patreon](https://www.patreon.com/gilles):
your continuous support is very much appreciated !

I created a Discord where I hang out and discuss my projects or sometimes screencast as I work on them.
Feel free to hop in if you want,
and feel free to do just like me and share thoughts as you work on your own projects there:
**this is a virtual hack room for anyone to join**: [https://discord.gg/YC6j4rbvSk](https://discord.gg/YC6j4rbvSk)

This is a fairly open server where you can ask random questions about a lot of topics,
not necessarily related to my own work.


# Table of content
- [Code-unrelated work](#code-unrelated-work)
- [I have not been very active these last three months](#i-have-not-been-very-active-these-last-three-months)
- [MHL: MIDI to human language](#mhl-midi-to-human-language)
- [go-midicsv: MIDICSV implementation in Golang](#go-midicsv-midicsv-implementation-in-golang)
- [Melodya: a melody and music assistant](#melodya-a-melody-and-music-assistant)
- [What’s next ?](#whats-next)


# Code-unrelated work
I began a few week ago a **certified training in music production** which I'll be attending for a couple weeks still,
and I'll be sharing here some of the things I work on.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/Ef8m4stneZ8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe></center>

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/RrtTNPesJzU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</center>

Feel free to [subscribe to my Youtube channel](https://www.youtube.com/channel/UCU-1Rn7gWhessHWwC3_EsEg)
where I will not only publish my works in progress but also document my journey in that area.


# I have not been very active these last three months
I have not been very active these last couple months.
Partly because of [life events and grieving](/posts/2023-03-01/i-love-you/),
partly because of general demotivation and partly because the training is draining what's left of my brain.
I'm slowly but surely stepping back in the game.


# MHL: MIDI to human language
For a couple of projects,
I needed a way to **express MIDI in a human-readable format**.

Those who have followed my work on `earmuff` may assume that it can serve this purpose,
but it doesn't because **it expresses things programatically and at a higher-level**:
where `earmuff` works with bars and beats and durations,
MIDI works with a set of events triggering at absolute ticks,
where `earmuff` has repeats blocks to provide "loops",
MIDI ... well, actually repeats events sequentially.

I wrote a tiny compiler called `mhl` that allows building an SMF,
Standard MIDI File,
from the following input and the other way around to generate MHL output from an SMF:
```
track=1 ticks=16080 message=NoteOff channel=0 key=40 velocity=64
track=1 ticks=16080 message=NoteOn channel=0 key=59 velocity=89
track=1 ticks=16080 message=NoteOn channel=0 key=55 velocity=89
track=1 ticks=16080 message=NoteOn channel=0 key=52 velocity=89
track=1 ticks=16320 message=NoteOff channel=0 key=59 velocity=64
track=1 ticks=16320 message=NoteOff channel=0 key=55 velocity=64
track=1 ticks=16320 message=NoteOff channel=0 key=52 velocity=64
```

Awesome idea, right ?
nope, wrong.


# go-midicsv: MIDICSV implementation in Golang
It turns out that after I was done writing `mhl`,
I ran into MIDICSV.

MIDICSV is a specification to represent MIDI as CSV,
a human-readable comma-separated format.
That's exactly what I wanted,
except that it already existed,
already covered what I wanted,
and there were already reference implementations in Perl and Python.
I did some experimenting with the Python version,
and since it proved to cover my functional needs,
**I decided to toss MHL and switch to using MIDICSV**.

I could not use the python version itself,
partly because I needed to call it from Golang which was painful,
but also because `py_midicsv` insists on using files whereas I need to work on bytes...
so I wrote a Golang implementation for my needs.
This is a work in progress,
do not use for anything serious,
but the initial code is [already commited and available in a Github repository](https://github.com/poolpOrg/go-midicsv).

Long story short,
the package comes with two commands `midi2csv` and `csv2midi`,
both built using the `go-midicsv/encoding` package which **provides an Encoder and Decoder**.
The Encoder reads bytes from an SMF and converts them into a MIDICSV output,
whereas the Decoder reads a MIDICSV input and converts it into an SMF output.

The code currently only works for valid SMF files and will likely panic on any invalidly crafted SMF,
I will make it error out nicely in the upcoming weeks.
Good enough for my pocs.


# Melodya: a melody and music assistant
Melodya is a project I'll be working on and off for the next few months:
it is **a melody and music assistant**,
helping you write, understand and improve music through the help of AI.
Unless you've been hiding under a rock,
you have probably already seen ChatGPT at work,
it's impressive af,
and it can prove really useful in this use-case of assisting music creation.

Unfortunately,
it **only understands text so it's hard to have it work with music**:
it can't take sound or sheet music as input.
I had already played with this in December,
[teaching ChatGPT about `earmuff`](/posts/2022-12-30/december-2022-some-more-earmuff-and-go-harmony/#chatgpt)
and having it generate new `earmuff` source based on instructions.
It worked,
it worked insanely well,
but had some shortcomings because `earmuff` was invented after the cut-off date for its corpus and doesn't come with enough examples.
Unless my prompts were very precise,
it would generate instruments names that aren't recognized,
sometimes even constructs that it thought existed but didn't,
and such...

Luckily,
between `earmuff`, `go-harmony` and `go-midicsv`,
I have written enough code that help deal with music programatically and in human-readable formats that I can use `ChatGPT` reliably enough for my purpose...
so I played tetris.
I assembled blocks,
and here's the initial result after a few hours of work.
A chatbot that understands what I want to do on a musical project,
that keeps track of the state of my project and that can provide explanations while generating MIDI output:

<center>
<img src="melodya.jpg" />
<br />
<br />
<img src="melodya2.jpg" />
<br />
<br />
</center>

This was the very first step.

I would like to implement speech-to-text so I can dictate my needs,
text-to-speech so it can hear back its answers,
but I would also like to **accept input coming from a MIDI device** so I can ask it to analyze, correct me or make suggestions,
just as I'd like to allow it to **output to a MIDI device** so I can plug it to a synthesizer or DAW so I can use it more directly.

In addition,
because **MIDI can be (indirectly) converted to sheet music**,
I'd like to be able to display the sheet music as it's being worked on and adapted.


# What's next ?
April will be essentially spent on my training,
so I don't expect much code to be written.

If I manage to,
it'll likely be melodya related.

Stay tuned !