---
title: "Preventive software maintenance"
date: 2026-08-13 02:00:00 +0200
authors:
 - "gilles"
language: en
categories:
 - technology
tags:
 - maintenance
 - technical debt
 - psychology
shoutout: false
needhelp: false
---

{{< tldr >}}
	Maintenance is not the same thing as fixing bugs.
	Only a fifth of it is about defects at all.
	Software often works because of work you can't see.
	So you can't wait for incidents to tell you what to fix.
{{< /tldr >}}


# Where this comes from
This article is adapted from a slide deck I prepared for a talk back in 2023.

I still think it's worth writing down,
and quite frankly I think it's a shame that it has been sitting in my Google Drive collecting dust for years.

The reason I still think it's worth it is that nothing has changed.
I have seen the same issues at countless companies,
the same errors being made,
again and again.

And I have seen both sides of it.
Teams that did follow something like what I describe here had few incidents,
and the few they had got resolved fast.
Teams that only repaid their debt when an accident forced them to had multiple incidents,
and those incidents lasted for extended periods of time.

That's a pattern I've seen too many times for it to be a coincidence.


# Maintenance is not bug fixing
Ask a room full of developers what software maintenance is,
and most of them will describe fixing bugs.

I did that for years myself,
so this is not me looking down on anyone :-)

But **the definition of maintenance says nothing about bugs**.
Maintenance is the **modification** of software,
**after it has been delivered or put in production**,
to **improve its general behavior**.
That's it.

There are actually four standardized categories of it,
and it's worth knowing their names because the vocabulary makes the problem obvious.
**Corrective** maintenance reacts to defects in the software.
**Adaptive** maintenance adapts to changes in the environment or in use-cases.
**Perfective** maintenance improves performance or maintainability.
And **preventive** maintenance detects and improves code before it fails.

Only the first one is about bugs.

The other three are the reason maintenance matters in the first place,
because doing it is what **reduces incidents**,
**preserves the value of the software over time**,
and **keeps the technical debt down**.

And the literature has been saying for decades that **corrective maintenance accounts for only about a fifth of the total**.
The number comes from Lientz and Swanson,
who surveyed 487 organizations back in 1980 and put corrective maintenance at roughly 21% of the effort.[^1]

[^1]: Lientz, B. P., & Swanson, E. B. (1980). *Software Maintenance Management: A Study of the Maintenance of Computer Application Software in 487 Data Processing Organizations*. Addison-Wesley. Worth noting that later work measuring actual change histories rather than asking managers, [Schach et al. (2003)](https://doi.org/10.1023/A:1025368318006), found corrective maintenance two to three times higher, which suggests self-reporting understates bug fixing. The exact ratio matters less than the direction: a large part of maintenance is not about defects.

A fifth.

Which means that if the only maintenance you do is the maintenance that incidents force upon you,
you are ignoring the other four fifths.
That work doesn't disappear because nobody filed a ticket for it.
It just sits there and becomes technical debt.


# Waiting for incidents doesn't work
In most teams,
the time allocated to maintenance is a consequence of incidents or reports.
Something breaks,
somebody complains,
and time appears.

That means the maintenance you end up doing is corrective,
and corrective maintenance is **reactive by nature**.
You are not deciding what to work on,
the failures are deciding for you.

The tempting answer is that it evens out in the end.
Incidents happen,
incidents force you to go fix things,
so the debt gets repaid eventually.

I don't believe that,
and I don't think anyone who has been on call believes it either.

The maintenance you do during an incident is the worst maintenance you will ever do.
Not because the people doing it are bad,
but because of what an incident is:
production is broken,
someone is waiting,
and you are choosing between the correct fix and the fix that works right now.

We all know which one wins at 3am.

So the debt does get touched,
but it often gets touched by a quick fix that replaces one technical debt with another.
You didn't really repay anything,
you just moved it around.

And this matters more than it sounds,
because **maintenance is really evolutionary development**.
Changing a system that already exists means understanding how it got that way,
understanding what your change will break,
and deciding what the next correct step is.
Those are all things that need calm and context.
They are exactly the things you don't have at 3am.

That's why preventive maintenance,
going to look for what's rotting before it fails,
can't be the thing you do when there's time left over.
**There is never time left over.**

It needs time allocated to it **regardless of incidents or defects**.
It is **not something done on the side**,
and it can't be done every once in a while when someone remembers.
It **MUST** be part of the regular development workflow,
with hours that belong to it whether or not anything is currently on fire.

Yes,
that's time not spent bringing new features.
It's also time spent **improving the value of the software**,
which is the thing features are supposed to be for.


# Prescribed work vs actual work
Everything above is the standard argument,
and you have probably heard some version of it before.

The reason I'm actually convinced comes from somewhere else.
I spent a few years studying occupational psychology,
and there's an idea in it that changed how I look at code bases:
the gap between **prescribed work** and **actual work**.

Prescribed work is the job as described.
Actual work is what people really do all day.
The two are never the same,
in any job,
anywhere.

And now the part that stings:
**ALL** workers hide part of their work,
and they do it for **good** reasons.
Not out of laziness,
not out of malice.
Work is, by definition, a constraint placed upon the workers,
and hiding some of what you do is how you absorb that constraint.

**Nothing can be done about that.**

That's not a management failure you can fix with a better process,
and it's not a defect in your team.
It's a property of work itself,
in every job,
including yours and mine.

Now apply that to software.


# Why does it work ?
The question I like to ask is simple:

**are you confident about your system if every member of the team is away at the same time ?**

If that makes you slightly uneasy,
the reason is usually that some of the system isn't in the system.

It's in someone doing a manual operation every Monday morning.
It's in a script that was running "temporarily" on somebody's machine in 2023.
It's in a person who knows to nudge a config value when traffic spikes.

None of that is in the repository.
All of it is holding the thing up.

The same goes for the code itself.
There's the hack written to get through an urgent situation.
The quick fix from an incident that was never revisited.
The features written in a hurry because someone was juggling three contexts at once.

And then the four facts that,
put together,
are the whole reason I care about this:

nobody remembers every hack they wrote.
Nobody knows the hacks written by everyone else.
Not everyone will admit to hacking something in the first place.
And **nobody is happy about any of it**.

That last one is the important one.
This isn't people being sloppy.
A hack written at 3am during an incident is not something you're proud of,
it's something you'd rather not bring up,
and the honest reason it doesn't get mentioned is that the constraint everyone was under made it the reasonable thing to do at the time.

Which leads somewhere uncomfortable:
the real state of your software is partly invisible to the people who wrote it.

And you cannot react to something you can't see.

Now,
it's not possible to avoid these situations,
and I want to be clear that I'm not proposing we try.
But it is in **everyone's** interest to get rid of them once they exist,
because it means less risk of incidents and stress,
and better software value.

It's in the group's interest that hidden work gets a chance to be fixed.
And it is very much in the developers' own interest to fix these shortcomings,
because a manual intervention you keep doing,
or a technical debt you keep stepping over,
is work that will come back to you later as an incident,
under stress,
at the worst possible moment.

Nobody wins by leaving it there.


# Software is creative
There's a second reason preventive maintenance needs real time,
and it has nothing to do with hidden work.

Software is dynamic by nature.
The environment evolves,
the use-cases evolve,
and development is therefore evolutionary:
the code in its current state is the result of **all its previous states**,
including states in previous or entirely unrelated projects.

And writing it is a creative process.
It is tainted by previous experiences,
education and personal preferences.
Two developers will not necessarily use the same algorithms,
and even when they do,
they will not use the same implementations.

Which means **there is a cost to understanding code you didn't write**.

That cost is where corrective maintenance quietly breaks down.
Corrective maintenance expects you to understand a code base.
Understanding a code base requires studying it.
Studying it requires spending time reading it and thinking about it.
So **not allowing time to study a code base is the same as choosing poor corrective maintenance**...
except you only find out during the incident.

It's worth being precise about why that understanding matters.
Understanding **why** things are done leads to better knowledge,
better knowledge leads to better development decisions,
and it leads to better analysis of incidents and better fixes.
The code is in its current state for a reason,
and knowing the reason is how you avoid changing it wrongly.

This is also,
by the way,
why guidelines only get you part of the way.
Guidelines put boundaries on subjectivity,
they don't erase it,
and they don't negate the evolutionary nature of development.
Following the same coding style helps everyone understand **what** a piece of code does.
It will not tell anyone **why** it does it.


# Gaining understanding of a code base
When a newcomer joins a project,
they read code and try to understand it.
They try to spot things to improve as a way of getting a grasp on it.
And eventually they understand enough to build on top of it.

That middle step is the interesting one,
because improving something is how people actually learn it.
It's not a detour from the real work,
it *is* how the understanding gets built.

And this is often forgotten,
but people already familiar with the code need to read it again too,
with the perspective of everything they've done since they last looked at it.
You are not the same developer you were two years ago,
so the code doesn't mean the same thing to you either.


# What you get out of it
Put together, maintenance buys you better code base understanding,
better testing,
and better maintainability in the long run.
It fixes issues before **and** after they're detected,
improves the software generally,
gets rid of technical debt,
and preserves the value of the software over time.

But the part I care most about is the human side.

It helps **retaining talents**,
because nobody wants to spend their days working with poor code.
It **ensures in-house skills**,
because knowledge that isn't maintained walks out of the door.
And it **reduces stress**,
because poor code quality is what produces incidents,
and incidents are what make this job miserable.


# How to actually do it
So here's the concrete proposal,
and it's simpler than it sounds:
**introduce dedicated maintenance time in each sprint**,
and make it about **a fifth of the project time**.

That time exists to absorb the maintenance cost.
During it,
developers are **not asked to deliver a feature**.
They work freely,
and they get to decide what they work on.
They can even decide,
on their own,
to work on features.

Projects carrying debt get a chance to repay it before an incident forces them to.
Projects without debt effectively get bonus time to work on features.
Either way it isn't wasted.

One thing needs to be said clearly,
because it's the most common misunderstanding:
this is **NOT** spare time,
and it is **not** Google's 20% personal project time.
This 20% is spent **on the project itself**.
It exists to fix the things that are off the radar and would otherwise never be fixed,
and it raises the quality of the project,
which benefits everyone.

For it to work,
everyone has to be a good player.
The PO must dedicate that 20% of project time **without questioning the work**,
and developers must use that time appropriately.
Both halves are required.
If the PO reclaims the time whenever a deadline gets close,
it dies.
If developers treat it as a day off,
it dies too,
and it kind of deserves to :-)

OKRs and reports are a decent way to hold that balance without turning it into surveillance.
Make it an objective each quarter to,
for example,
make at least **three improvements** to the existing code base,
and produce a **short report** describing them.

The number of improvements should be kept deliberately low.
Work has an inherent hidden part,
so the report is not a timesheet and it is not meant to account for every hour.
As long as the result is achieved,
you know the quality has improved.
The whole point is that you want people to have the time to fix things **you are not aware of**,
and by definition you cannot put those on a roadmap.

That's pretty much the whole idea.
Corrective maintenance can only address what surfaces,
and what surfaces is only the visible part of what's actually there,
which is why you need maintenance time that goes looking for the rest.


# UPDATE: what AI changes
I wrote the original deck in 2023,
before code generation became part of everyone's daily workflow,
and I think everything above is more relevant now than it was then.

The productivity gain is real,
I'm not going to pretend otherwise.
Code generation makes people faster.

But generated code is,
by definition,
code you didn't write,
and I explained above that there is a cost to understanding code you didn't write.
That cost doesn't disappear because the code appeared quickly.

It also lands in the project already finished.
The loop I described earlier,
where you read code,
spot things to improve,
and end up understanding it,
simply doesn't happen.
The code arrives without the understanding that used to come with writing it.

So people lose their grip on their own code base,
and they lose it faster than before.
Not because they're bad developers,
but because the thing that used to build familiarity has been skipped.

And I want to be clear that supervised sessions are not cutting it either.

People assume that driving an AI implies understanding what it produces,
but cognitive psychology and the way learning works say otherwise.
Learning happens by doing and repeating.
You do not internalize the same understanding from writing the code manually,
jumping from a function to another,
fixing compile errors,
building intuition while you're at it...
as you do from asking for two functions to be generated and then reviewing them.
It's simply not the same thing,
and it shows right away in peer review.

Someone who wrote the code can be asked about a specific line and be comfortable answering.
Someone who generated it has to stop and work out what that line is doing
inside the general idea they described to the model.
It's a different kind of knowledge,
and the difference is quite audible.

The hidden work part gets worse too.
Nobody recalls every hack they wrote,
and now nobody recalls every hack they **didn't** write either,
sitting somewhere in a diff that was skimmed for thirty seconds because it looked plausible and the tests were green.

And this is where I'll be a bit blunt.

Projects now pile up huge chunks of code at a speed we've never had before,
and a large portion of developers have become too lazy to review it.
I don't say that to be unpleasant.
Reviewing a large generated diff properly is tedious work,
and when it looks right and the tests pass,
there is close to no incentive to actually read it.

Multiplied by every sprint,
across a whole team,
for a year,
you end up with a code base nobody has fully read,
full of decisions nobody really made,
maintained by people who never built a mental model of it.
It's the same invisible state I described earlier,
except it now grows at machine speed.

To be clear,
none of this is me saying you shouldn't use AI.
It would be stupid to discard tools that powerful,
and I use them myself.

What I'm saying is that they come with this drawback,
and that you need to be aware of it.

Not taking preventive maintenance seriously was already a bad idea years ago.
Now that code lands in projects faster than anyone can read it,
it has become plain stupid,
at least if you intend to retain any value in your software.

So the 20% is not a nice-to-have anymore.
If code enters your project faster than humans can understand it,
then time to read it,
question it and improve it is the only thing left between you and a system that runs on nobody's understanding.

The tooling got faster,
we didn't,
and that gap has to be paid one way or another.
You just get to choose if you pay it deliberately or at 3am :-)
