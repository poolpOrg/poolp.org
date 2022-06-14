---
title: "June 2022: go-harmony and earmuff"
date: 2022-06-14 12:27:00 +0200
authors:
 - Gilles Chehade
language: fr
---

<blockquote>
<b>TL;DR:</b>
did a lot of music, even while writing code.
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


# Code-unrelated work
As last month,
I'll start with code-unrelated work !

First,
here's my progress on learning Hyunsoo Lee's adaptation of Bach's Air on G string,
still not quite right and with some missing parts but... slowly getting there:
<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/u6G3bgIk4Wo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

While at it,
I started learning Hyunsoo Lee's adaptation of Beethovens' 5th's Symphony,
also not quite right and with missing parts but also making progress:
<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/PtlDJwlM-SM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

It was a long time since I made a LoFi track,
here's two that I thought were not too bad:

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/rjxiUCOkLGA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/Fk0grNZ4LtY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

They're mostly an exercise to get familiar with Logic Pro X.




# go-harmony
`go-harmony` is a package written in Golang that intends to implement a **music theory engine** to build other tools with.
It's only a month old so there's not much to it yet,
however there's enough that I could use it to build the project I'll talk about in the next section.

My intent was not only to build the engine but also to refresh my memories and fill some gaps...
so I decided not to be lazy and copy-paste a bunch of values in static tables.
Instead,
I first implemented notes,
then intervals as semitones distances to a root note,
then chords as a pattern of intervals from a root note (taking into account chord inversions),
then also scales also as a pattern of intervals,
then derived triads and sevenths chords using the chords implementation,
and so on...

I made it so it would be very easy to extend for unsupported cases,
like chords which are defined as follows making it simple to add new families by simply describing the general structure:
```go
  MajorTriad Structure = Structure{
    intervals.PerfectUnison,
    intervals.MajorThird,
    intervals.PerfectFifth,
  }

  MinorTriad Structure = Structure{
    intervals.PerfectUnison,
    intervals.MinorThird,
    intervals.PerfectFifth,
  }

  AugmentedTriad Structure = Structure{
    intervals.PerfectUnison,
    intervals.MajorThird,
    intervals.AugmentedFifth,
  }

  DiminishedTriad Structure = Structure{
    intervals.PerfectUnison,
    intervals.MinorThird,
    intervals.DiminishedFifth,
  }

[...]

  AddNinth      Structure = append(MajorTriad, intervals.MajorNinth)
  AddEleventh   Structure = append(MajorTriad, intervals.PerfectEleventh)
  AddThirteenth Structure = append(MajorTriad, intervals.MajorThirteenth)

  SusSecond Structure = Structure{
    intervals.PerfectUnison,
    intervals.MajorSecond,
    intervals.PerfectFifth,
  }

  SusFourth Structure = Structure{
    intervals.PerfectUnison,
    intervals.PerfectFourth,
    intervals.PerfectFifth,
  }
```

The same applies for scales:
```go
  var Ionian = []intervals.Interval{
    intervals.PerfectUnison,
    intervals.MajorSecond,
    intervals.MajorThird,
    intervals.PerfectFourth,
    intervals.PerfectFifth,
    intervals.MajorSixth,
    intervals.MajorSeventh,
    intervals.Octave,
  }

  var Dorian = []intervals.Interval{
    intervals.PerfectUnison,
    intervals.MajorSecond,
    intervals.MinorThird,
    intervals.PerfectFourth,
    intervals.PerfectFifth,
    intervals.MajorSixth,
    intervals.MinorSeventh,
    intervals.Octave,
  }

[...]

  var BluesMajor = []intervals.Interval{
    intervals.PerfectUnison,
    intervals.MajorSecond,
    intervals.PerfectFourth,
    intervals.PerfectFifth,
    intervals.MajorSixth,
    intervals.Octave,
  }

  var MinorPentatonic = []intervals.Interval{
    intervals.PerfectUnison,
    intervals.MinorThird,
    intervals.PerfectFourth,
    intervals.PerfectFifth,
    intervals.MinorSeventh,
    intervals.Octave,
  }
```

Then `go-harmony` uses **artihmetics to compute everything** from a root note and its intervals.
For example the `scale.Notes()` method computes the notes for a scale by applying the intervals that match the scale pattern for a given root note:
```go
func (scale *Scale) Notes() []notes.Note {
	ret := make([]notes.Note, 0)
	for _, interval := range scale.structure {
		ret = append(ret, *scale.root.Interval(interval))
	}
	return ret
}
```
No hard-coded scales for each notes and once a scale structure is implemented, it works for all notes.


The package by itself doesn't do much more,
it's meant to provide APIs and structures for other projects to rely upon,
however I built a small utility that's shipped with it to showcase some of it's features.

For example,
it can be used to validate note names and obtain their frequencies relative to a particular tuning (only A440 for now):

```sh
% harmony -note 'C'
C 261.63

% harmony -note 'C5'
C 523.25

% harmony -note 'A'
A 440

% harmony -note 'Cb'
Cb 493.88

% harmony -note 'C#'
C# 277.18

% harmony -note 'Cbb'
Cbb 466.16

% harmony -note Z
2022/06/14 13:25:36 bad note (Z): should be 'C', 'D', 'E', 'F', 'G', 'A' or 'B'
```

Just as with notes,
it can be used to validate chord names (it supports multiple notations) and decompose them into their building intervals:
```sh
% harmony -chord 'C'
Cmaj
       1:   C 261.63
    3maj:   E 329.63
       5:   G 392.00

% harmony -chord 'C5'
C5
       1:   C 261.63
       5:   G 392.00

% harmony -chord 'C7'
C7
       1:   C 261.63
    3maj:   E 329.63
       5:   G 392.00
    7min:  Bb 466.16

% harmony -chord 'C7b5'
C7dim5
       1:   C 261.63
    3maj:   E 329.63
    5dim:  Gb 369.99
    7min:  Bb 466.16

% harmony -chord 'Csus2'
Csus2
       1:   C 261.63
    2maj:   D 293.66
       5:   G 392.00

% harmony -chord 'Cadd9'
Cadd9
       1:   C 261.63
    3maj:   E 329.63
       5:   G 392.00
    9maj:   D 587.33

% harmony -chord 'Cadd9/E'
Cadd9/E
    3maj:   E 329.63
       1:   C 261.63
       5:   G 392.00
    9maj:   D 587.33

% harmony -chord 'Cadd2' 
2022/06/14 13:24:43 unknown chord name: add2
```

It knows of a few scales and modes and can decompose them into notes, triads and sevenths chords for each degree of the scale:
```sh
% harmony -scale Caeolian
C
   C 261.63
   D 293.66
   Eb 311.13
   F 349.23
   G 392
   Ab 415.3
   Bb 466.16
   C 523.25
Triads:
   Cmin
   Ddim
   Ebmaj
   Fmin
   Gmin
   Abmaj
   Bbmaj
Sevenths:
   Cmin7
   Dm7b5
   Ebmaj7
   Fmin7
   Gmin7
   Abmaj7
   Bb7

% harmony -scale Cphrygian
C
   C 261.63
   Db 277.18
   Eb 311.13
   F 349.23
   G 392
   Ab 415.3
   Bb 466.16
   C 523.25
Triads:
   Cmin
   Dbmaj
   Ebmaj
   Fmin
   Gdim
   Abmaj
   Bbmin
Sevenths:
   Cmin7
   Dbmaj7
   Eb7
   Fmin7
   Gm7b5
   Abmaj7
   Bbmin7
```

And finally,
someone asked if it could build chords from a given set of notes,
so I implemented it which took only a couple minutes:
```sh
% harmony -notes 'C,E,G'
Cmaj
   C 261.63
   E 329.63
   G 392

% harmony -notes 'C,Eb,G,B'
C-M7
   C 261.63
   Eb 311.13
   G 392
   B 493.88

% harmony -notes 'C,Eb,G,Bb'
Cmin7
   C 261.63
   Eb 311.13
   G 392
   Bb 466.16

% harmony -notes 'C,F,G'    
Csus4
   C 261.63
   F 349.23
   G 392
```

That's about all it can do for now,
but I have many ideas I want to implement as I have big plans for it :-)


# earmuff

`earmuff` is a proof-of-concept **compiler and interpreter for a programming language to write music**.

After I showed the first iteration of the interpreter,
two people pointed me to [Sonic Pi](https://sonic-pi.net) and it kinda is a similar concept but executed a bit differently.
Whereas Sonic Pi's is an **advanced scriptable synthesizer with its own IDE**,
`earmuff` is simply a **language to express music sheet as code** and doesn't come with anything but the interpreter and compiler for it.

My intent was to be able to write music as code **in my usual programming environment**,
using `diff`, `patch` or even `git` for versionning my changes and applying new changes,
and basically **work with music in the same way I work with code** when I don't have instruments at hands.


So `earmuff` reads `.muff` source files,
and either compiles them to SMF (Standard Midi File) files that can be played with a standard MIDI player,
or interprets them into MIDI messages that are sent to a synthesizer to play the source code in real-time.

It relies on `go-harmony` to make sense of notes and chord names,
**spotting errors** as it converts the source code to its internal format,
and **validating** that the tracks are **structurally consistent** (ie: time signature violations) just as if it was checking for syntax or grammar errors.

A `.muff` file may look like this for a single-track 12-bars blues:
```txt
project {
    bpm 120;
    time 4 4;

    instrument guitar {
        bar { whole chord C7 on 1/1; }
        bar { whole chord F7 on 1/1; }
        bar { whole chord C7 on 1/1; }
        bar { whole chord C7 on 1/1; }
        bar { whole chord F7 on 1/1; }
        bar { whole chord F7 on 1/1; }
        bar { whole chord C7 on 1/1; }
        bar { whole chord C7 on 1/1; }
        bar { whole chord G7 on 1/1; }
        bar { whole chord F7 on 1/1; }
        bar { whole chord C7 on 1/1; }
        bar { whole chord G7 on 1/1; }
    }
}
```

... or may be more complex for multi-tracks projects,
like these few bars from [Django's Nuages](https://www.youtube.com/watch?v=DY0FF4iR9Cw) (I know, it's not quite right but hey, it was late and I'm still learning the language &#128517;):
```txt
project {
    bpm 80;
    time 4 4;

    instrument guitar {
        bar {
            8th note C# on 3/1;
            8th note D on 3/2;
            8th note A on 3/3;
            8th note G# on 3/4;
            8th note G on 4/1;
            8th note F# on 4/2;
        }
        bar {
            half note F on 1/1;
            8th note E on 4/2;
        }
        bar {
            half note Fbb on 1/1;
            quarter note Eb on 2/2;
            8th note D on 4/2;
        }
        bar { whole note D on 1/1; }
    }

    instrument piano {
        bar {}
        bar { whole chord Eb9 on 1/1; }
        bar {
            half chord Am7b5 on 1/1;
            half chord D7b9 on 3/1;
        }
        bar {
            half chord Gmaj7 on 1/1;
            quarter chord Am7 on 3/1;
            quarter chord Bm7b5 on 4/1;
        }
    }

    instrument percussive {
        bar {}
        bar { half cymbal on 1/1; }
        bar { half cymbal on 1/1; }
        bar { half cymbal on 1/1; }
    }
}
```

By default,
`earmuff` operates as an interpreter so running the following command will playback the source if a synthesizer is detected (currently only FluidSynth is):

```sh
% earmuff nuages.muff
```

<center>
<audio controls>
  <source src="nuages.mp3" type="audio/mpeg">
Your browser does not support the audio element.
</audio>
</center>
<br />

The `-verbose` option may be passed,
in which case `earmuff` outputs MIDI messages being sent to the synthesizer in real-time for debugging:
```sh
synth <- 0 ProgramChange channel: 0 program: 25
synth <- 2 ProgramChange channel: 10 program: 113
synth <- 1 ProgramChange channel: 1 program: 1
synth <- 0 NoteOn channel: 0 key: 49 velocity: 120
synth <- 0 NoteOn channel: 0 key: 50 velocity: 120
synth <- 0 NoteOn channel: 0 key: 57 velocity: 120
synth <- 0 NoteOff channel: 0 key: 49
synth <- 0 NoteOn channel: 0 key: 56 velocity: 120
synth <- 0 NoteOff channel: 0 key: 50
synth <- 0 NoteOff channel: 0 key: 57
synth <- 0 NoteOn channel: 0 key: 55 velocity: 120
synth <- 0 NoteOff channel: 0 key: 56
synth <- 0 NoteOn channel: 0 key: 54 velocity: 120
synth <- 0 NoteOff channel: 0 key: 55
synth <- 0 NoteOff channel: 0 key: 54
synth <- 2 NoteOn channel: 10 key: 51 velocity: 120
synth <- 1 NoteOn channel: 1 key: 51 velocity: 120
synth <- 1 NoteOn channel: 1 key: 49 velocity: 120
synth <- 0 NoteOn channel: 0 key: 53 velocity: 120
synth <- 1 NoteOn channel: 1 key: 55 velocity: 120
synth <- 1 NoteOn channel: 1 key: 65 velocity: 120
synth <- 1 NoteOn channel: 1 key: 58 velocity: 120
```
<br />

Passing the option `-out` will cause it to compile a `.mid` file:

```sh
% earmuff -out nuages.mid nuages.muff
```

... which can either be played by a standart MIDI player,
imported in tools that can ingest MIDI files such as a DAWs (here Logic Pro X):

<center>
    <a href="earmuff-daw.png"><img src="earmuff-daw.png"></a>
</center>
<br />

... or tools like Guitar Pro which can render sheet music from a MIDI file,
play it back after altering speed, instruments, etc...

<center>
    <a href="earmuff-gp.png"><img src="earmuff-gp.png"></a>
</center>
<br />

This is a toy project at a **very early stage**,
I don't have very serious plans for it but I'll keep working on it as it improves my overall knowledge,
and I kinda like the idea of being able to write music as code from my bed while the kid sleeps by my side
(also I'm far faster as writing code than editing sheets) :-)

There are still bugs and glitches in the MIDI generation but **the MP3 I linked above was converted from the .mid built by the source code just above it**,
so it may not be perfect yet but it works pretty decently in my opinion.

My current plans are to cleanup the proof-of-concept,
make it work on at least macOS, Windows, Linux and OpenBSD,
and then extend it with new features such as **loops and functions**.
A very nice feature would be to allow it to read from MIDI and generate the corresponding `.muff` code,
allowing me to auto-generate code from a MIDI input (like a keyboard for example),
or simply allowing me to export a SMF from another tool to work on it with `earmuff` and export it back to the other tool:
being able to **switch back and forth** between VScode and Guitar Pro or Logic Pro X would be awesome.


# What's next ?

Next is **A DAMN BREAK** cause I said last month I'd take a couple months and couldn't even hold my word for a single one.
I'll be marrying in a couple weeks,
going on vacations a few weeks later,
and got plenty of code-unrelated things to finish.

Feel free to [join my discord](https://discord.gg/YC6j4rbvSk),
engage on [twitter](https://twitter.com/poolpOrg) or 
support me on [github](https://github.com/sponsors/poolpOrg) or/and [patreon](https://www.patreon.com/gilles).

Take care,
stay tuned,
I'll post as soon as I resume my work !

---- 
Comments: [https://github.com/poolpOrg/poolp.org/discussions/153](https://github.com/poolpOrg/poolp.org/discussions/153)
