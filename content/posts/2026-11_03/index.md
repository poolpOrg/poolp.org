---
title: "I'll never buy Framework hardware again"
date: 2026-11-20
authors:
 - "gilles"
categories:
 - personal
 - framework
---

{{< tldr >}}
We bought a 3000 EUR Framework Laptop 16 for one of our developers.
It was broken out of the box.
Three months, around thirty support emails, two replacement parts and a trip to their repair center later,
Framework concluded "No Fault Found" and blamed the keyboard layout of an OS that wasn't even installed when the problem first showed up.
I'm done with them.
{{< /tldr >}}

Some context first
--
I have been a Framework fan for a while.
At my company, developers pick the machine of their choice and several work on Framework laptops or desktops,
we have about quite a few of them around,
and I have been recommending them to pretty much anyone who asked me what laptop to buy so far.
Repairable hardware, Linux-friendly, a company that seemed to care about the people using its products:
it was an easy recommendation to make.

So when Q joined the team and needed a workstation,
we ordered a Framework Laptop 16,
around 3000 EUR once configured.

Three months later,
I'm pissed off at Framework and have banned future purchases from them at my company.
Not because the machine arrived broken,
this happens and I've been around hardware long enough to know it,
but because of how unprofessional and useless the support has been from the first email to the last.

OH and feel free to share, friends don't let friends enter hell.

Day one: broken out of the box
--
The laptop arrives early August.
Q assembles it, powers it on and is greeted by a "missing input module" screen,
**before reaching the BIOS, before any operating system is even installed**.
Keep this detail in mind, we'll get back to it.

The keyboard isn't detected when centered,
and when moved to the left it shows up as... a numpad.
In that state, nothing works: not the keyboard, not the touchpad, not the numpad.
There's a BIOS option, "Force Power for Input Modules", that bypasses module detection.
With it enabled, everything works.
Without it, the machine has no keyboard and no touchpad.
It is recommended **NOT** to turn it on permanently as it might be harmful to the hardware,
as stated on their own website.

I open a ticket on August 13th.


August: the script loop hell
--
What follows is two weeks of support reading from a script, one step per email, one email per day or two:

- is it out of the box? any damage on the packaging? which OS, which kernel, which BIOS version?
- do a mainboard reset by pressing the chassis switch ten times
- reseat the midplate
- try Force Power On (we had already told them it was the only thing that worked)
- send photos of the keyboard, the midplate, the connector, the whole laptop from six angles, and a video
- reset the BIOS to defaults
- reinstall the BIOS and the *drivers bundle* (a Windows drivers bundle, for a problem that happens before any OS boots, on a machine that doesn't run Windows...)
- try different input module combinations

We did all of it.
We sent the photos and the video in a shared drive.
Each reply came from a different agent who clearly hadn't read the thread,
and asked for something we had already provided.

On August 24th I lost patience and wrote that this was a 3000 EUR laptop that didn't work,
that my co-worker had been without a usable machine of his own for over a week,
and asked them to either send replacement parts or take it back and refund us.
I even went all Karen and asked to involve a manager.

Their answer: please send a clear photo of the midplate and the underside of the keyboard.
The same photos, again.
No manager involved to break the script because no flying fucks were given.


We did their diagnosis for them
--
Q then replied from his own address,
which apparently is a problem,
because support spent the next ten days asking him to "verify the shipping address" of the order,
which he did, several times.

Meanwhile, tired of being asked to repeat steps,
he updated the keyboard firmware himself (only possible with Force Power On, since the keyboard is otherwise not detected),
installed Framework's own `framework_tool`,
and ran a proper isolation:

```
$ sudo framework_tool --inputdeck
```

The result was unambiguous:
the keyboard module reports the ID of a numpad, `KeyboardB`.
The firmware therefore sees two numpads and no keyboard,
considers the input deck incomplete,
and never enables the power rail for the input modules,
which is why *everything* is dead and not just the keyboard.

He moved the keyboard alone across slots: it misidentifies itself in every position.
He moved the numpad alone into the same slot: it identifies correctly.
He removed everything: the baseline is clean.
The wrong ID follows the keyboard.
The keyboard is faulty.

He sent all of this to support on August 28th with a full report and a one-line reproduction command.


September: they replace the wrong part
--
On September 7th, support concludes... that the **midplate** needs to be replaced.
Not the keyboard,
the only part that a reproducible diagnostic, run with their own tool, had just pointed at.

The midplate arrives a week later.
The problem is, of course, still there because... GOTO previous paragraph.
Q asks them whether they're sure it's not the keyboard, given that he proved it with the diagnosis.

Their answer ?

Back to the queue.
Back to photos:
the packaging box from different angles including the interior,
the midplate front and back,
the keyboard top and bottom,
close-ups of the sockets.
Photos they already had.

At this point,
Q is no longer a developer but has become a profesionnal photographer.

October: a new keyboard, a new bug
--
On October 8th, two months after the laptop arrived,
Framework finally agrees to replace the keyboard.
What they should have done on the first week...

The new keyboard is detected, the original bug is gone,
but there is a new one:
pressing **C** sends a `c` followed by a Delete escape sequence.

```
$ showkey -a
c       99 0143 0x63
^[[3~   27 0033 0x1b
        91 0133 0x5b
        51 0063 0x33
       126 0176 0x7e
```

An external keyboard on the same machine produces a single `c`, as it should.

FU-CKIN' HELL it never ends...

You can't use a keyboard where typing a word containing a `c` deletes the next character.
You certainly can't type a disk encryption passphrase with it.

Q is smart so he knows the drill and sends the `showkey` output, photos, and a video to avoid wasting time on a useless round-trip,
but he understimated them:
support asks him whether the issue happens with an external keyboard (he had already said it didn't),
then asks him to boot a live USB to "isolate software issues" (yes, the famous ones that would trigger on just their keyboard).
He does, tries both the US and French layouts, same result as anyone not brain-dead would expect.

The ticket is escalated, with the usual "escalations may take 5 to 7 working days".


November: "No Fault Found"
--
End of October, Framework decides to have the laptop sent to their repair center for a "full review and assessment",
with a reminder that we may be charged if they find customer-induced damage.

We ship it.

This morning, November 20th, the verdict arrives:

> Our repair center has completed their diagnostics, tested two keyboards and found that they were working perfectly.
> They have determined No Fault Found and that the issue was likely caused by installing the wrong keyboard layout in the operating system.

Let me be very clear about why this is nonsense:

- **Keyboard 1** failed before any operating system was installed on the machine.
  The "missing input module" screen shows up before the BIOS.
  There was no keyboard layout to get wrong.
  However, the BIOS workaround *they asked us to enable* was still on when the laptop was shipped,
  and with it on, the problem is hidden.
  If you don't turn it off, you won't see the fault.
- **Keyboard 2** fails in GRUB, before the kernel is even loaded,
  and on a live USB with both QWERTY and AZERTY layouts,
  which, by the way, put the C key in the same place.
  A keyboard layout maps a key to a character;
  it does not make a key emit *two* scancodes.
  And if you test by typing a sentence and looking at the result,
  a spurious Delete at the end of a line deletes nothing and you'll never notice it.
  You have to actually look at what the key sends, which is exactly what we showed them.

So either the repair center tested with the workaround on and never pressed C in the middle of a word,
or they didn't test at all.
In both cases, "the user picked the wrong keyboard layout" is not a diagnosis,
it's a way to close a ticket.


What it cost us
--
Let's count:

- three months and one week since the laptop arrived,
- around thirty emails from support, handled by more than a dozen different agents,
- two replacement parts, one of which was the wrong one,
- a round trip to a repair center,
- hours of an engineer's time spent producing photos, videos, reports and diagnostics that the support team then didn't read,
- a developer who had to work for weeks with external devices plugged into a laptop that couldn't be used as a laptop.

And at the end, the laptop comes back in the exact state it left,
with a note implying we don't know how to configure a keyboard layout.


The problem isn't the hardware
--
Hardware fails.
A bad keyboard module out of the box is annoying, not scandalous.
I've had DOA hardware from every vendor I've ever bought from.

What's scandalous is the process:

- nobody owns the ticket, every reply comes from someone new who restarts from a script;
- information we provide is not read, so it gets requested again, and again;
- a clear, reproducible diagnosis made with **their own tool** is ignored in favor of replacing an unrelated part;
- basic customer identity handling is broken enough that a colleague replying on the thread triggers ten days of address verification;
- and when it finally reaches people who could actually look at the hardware, the conclusion contradicts facts that were in the ticket from day one.

What's scandalous is the amount of not giving a damn,
not escalating,
not caring at all about having a customer in a shit situation for more than three months when they could simply replace the broken part and be done.
Like they can't afford a spare keyboard and make sure that it works ?


So, what now?
--
We're not going to spend another three months trying to get Framework to acknowledge a fault we've demonstrated a dozen times when they obviously don't care.
We'll recycle the laptop into a desktop when it comes back from the repair center and will get Q a new laptop from another vendor that's more serious and professional.

As for me:
buying new Framework hardware is now off the table.
The machines we have will live their lives,
but the next ones will come from somewhere else.

I liked the idea of Framework a lot.
Repairability is worthless if the company behind it can't tell a broken keyboard from a keyboard layout nor... repair,
and the support I've experienced is the opposite of what I'd expect from a company selling premium-priced hardware to professionals.

If you're considering a Framework for your company,
I hope it works out of the box for you.
Because if it doesn't,
you now know what's waiting for you.
