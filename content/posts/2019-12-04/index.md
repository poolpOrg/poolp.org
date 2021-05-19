---
title: "Decentralised SMTP is for the greater good"
date: 2019-12-15 07:29:00 +0200
category: opensource
authors:
 - Gilles Chehade
---

    TL;DR:
    - SMTP is the way computers exchange e-mails
    - it is a decentralised protocol meaning that ANYONE can run a node and be independant
    - it is being centralised at companies that have a history of abuse
    - it is being centralised in a country that has a history of abuse


Where did you read this already ?
--
In August,
I published a small article titled "[You should not run your mail server because mail is hard](https://poolp.org/posts/2019-08-30/you-should-not-run-your-mail-server-because-mail-is-hard/)" which was basically my opinion on why people keep saying it is hard to run a mail server.
Unexpectedly,
the article became very popular,
reached 100K reads and still gets hits and comments several months after publishing.

As a follow up to that article,
I published in September a much lenghtier article titled "[Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/)" which described how you could setup a complete mail server.
I went from scratch and up to inboxing at various Big Mailer Corps using an unused domain of mine with a neutral reputation and describing precisely for each step what was done and why it was done.
The article became fairly popular,
nowhere near the first one which wasn't so technical,
but reached 40K reads and also still gets hits and comments several months after publishing.

The content you're about to read was part of the second article but it didn't belong there,
it was too (geo-)political to be part of a technical article,
so I decided to remove it from there and make it a dedicated one.
I don't want the tech stack to go in the way of the message, **this is not about OpenSMTPD**.


Self-hosting and encouraging smaller providers is for the greater good
---
There are **political consequences** to centralizing mail services at Big Mailer Corps.

It doesn't make sense for Random Joe,
sharing kitten pictures with his family and friends,
to build a personal mail infrastructure when multiple Big Mailer Corps offer "for free" an amazing quality of service.
They provide him with an e-mail address that is immediately available and which will generally work reliably.
It really doesn't make sense for Random Joe not to go there,
and particularly if **even techies** go there without hesitation,
proving it is a sound choice.

There is nothing wrong with Random Joes using a service that works.

What is **terribly wrong** though is the centralization of a **communication** protocol in the hands of a few commercial companies,
**EVERY SINGLE ONE OF THEM** coming from the same country (currently led by a lunatic who abuses power and [probably suffers from NPD](https://psychcentral.com/blog/the-psychology-of-donald-trump-how-he-speaks/)),
**EVERY SINGLE ONE OF THEM** having been in the news and/or in a court for random/assorted "unpleasant" behaviors
(privacy abuses, eavesdropping, monopoly abuse, sexual or professional harassment, you just name it...),
and **EVERY SINGLE ONE OF THEM** growing user bases that far exceeds the total population of multiple countries combined.

Let's put a bit of perspective.
The biggest one of them reports a user base that exceeds **1.4 BILLION** users (April 2018),
roughly the entire population of either one of China or India,
exceeding by four the population of United States by itself,
or by over twenty the population of my own country.
If you counted its users at the rate of one each second,
**it would take you over 44 years to go through**.

Then,
**very far below**,
it is followed by the next one which reported 400 million active users (2018 too),
also exceeds the population of the United States,
being four times the population of Egypt,
and five times the population of my own country.

In many of the mailings I have monitored,
the very first one covered approximately half the recipients e-mail address space...
before the other half was even split in large parts between the _other_ Big Mailer Corps.

This is **NOT** sane, not here, not anywhere, not in any alternate dimension _(insert "sliiiiiiders" whisper here)_.

<center>
<img src="/images/2019-09-01-portal.jpg">
</center>

Take a moment to let this sink in:

If these companies were somehow required to cut communications with a country for sanctions,
well over a **BILLION** people could be out of reach for the targeted country.
If you think that this is far-fetched,
the users from Iran, Syria, Cuba or Crimea could surely provide you with an alternate point of view after
[discovering they were kicked out of Github](https://techcrunch.com/2019/07/29/github-ban-sanctioned-countries/) to enforce sanctions on their countries.
But **as annoying as it is**,
git is still a techie thing which mostly impacts techie people,
a minority of human beings...
and it is also decentralized.
Even if Github services are stopped,
the users can move to another platform or self-host easily,
there's no need for interoperability with the punishing country if it wants to cut ties.

E-mails are much different:
they affect everyone,
from all age and all backgrounds,
the consequences of a similar block if it was done by Big Mailer Corps are orders of magnitude harsher.
The people of [shithole countries](https://www.washingtonpost.com/politics/trump-attacks-protections-for-immigrants-from-shithole-countries-in-oval-office-meeting/2018/01/11/bfc0725c-f711-11e7-91af-31ac729add94_story.html) are not only techies that can work around,
they are people who **NEED** interoperability to communicate with the billion and more of people hosted at Big Mailer Corps.
Mail is good for connecting people worldwide in a reliable way,
but this only works if you keep it decentralized,
**NOT** when you centralize it in the hands of a few companies all geolocalized in the same country.

Now that might never happen,
and I would bet heavy money it won't because
[a country known for intercepting communications and spying on the whole world](https://en.wikipedia.org/wiki/PRISM_(surveillance_program))
has far more interest in keeping communications flowing,
but the sole fact that it is **_TECHNICALLY_** possible should be enough to make us all uneasy.
I'm even voluntarily looking away from the reason why I think they won't do it as if it didn't matter.
If Big Mailer Corps were hosted in China we'd be dead worried and already considering a plan out,
but the only reason we're not is because we _think_ we're currently on the "right side" of the fence.

But that's until we're screwed.

So **YES**, I'd rather see a lot more people self-host or **rely on smaller geolocalized providers**,
spreading the SMTP network as much as possible across multiple operators in multiple countries.
I don't think everyone should self-host,
I just think **MORE** people should self-host or move away from Big Mailer Corps into **ANY** other provider...
so there is at least a bit of constraint for them to interoperate with operators outside their gang,
as well as a financial risk for them **IF** they suddenly went rogue.

I'll say it again:

I don't think that either one of the Big Mailer Corps are evil or bad,
**I use some of their services on a daily basis**,
and most of the people operating them are genuinely seeking the greater good...
however they have grown too big and there needs to be a balance in power because who knows how they'll evolve in the next ten years,
who knows how the politics of their home country will evolve in the next ten years,
and recent news doesn't paint them as heading in the right direction.

I'll conclude by recommending that you see this **excellent** presentation by Bert Hubert ([@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert)) from PowerDNS,
about how a similar problem is starting to happen with DNS and the privacy and tracking concerns that arise from this.
Many, many, many key points are also valid for mail services.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/pjin3nv8jAo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>



---- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/68](https://github.com/poolpOrg/poolp.org/issues/68)
