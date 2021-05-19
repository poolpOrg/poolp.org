---
title: "Some OpenSMTPD overview, part 3"
date: 2014-12-10 12:38:26
category: OpenSMTPD
authors:
 - Gilles Chehade
---

EHLO,

This is the third post of a series about OpenSMTPD improvements that have taken place since this summer.

Content altering
----------------
For a long time, we have developed OpenSMTPD with a strict rule that the daemon should not alter DATA (as in the DATA SMTP phase) in any way.

The rationale was that by enforcing that rule, the message writing was simplified as the smtp process would simply read data from a client and write it, without any post-processing, to a file descriptor. We didn't want to prevent the writing of data modifiers, but we wanted to move this feature out of the daemon and into the filtering API which would allow any kind of rewriting outside the daemon memory space and with different privileges if we wanted to.

The other rationale was that by avoiding adding hooks to alter content we were receiving, we made it harder on us and contributors to go lazy and implement every feature out there in the daemon just because it was easier through the hooks. When we wanted to implement a feature, we would immediately ask ourselves if it could be done outside the daemon scope before eventually adding it.

Besides the occasional complaints that we did not support address masquerading (yet), this did not really cause any issue with our users running a wide range of MUA and we decided to maintain that rule, convince people that masquerading was really a filter thing and that it would become available as soon as our filter API had stabilized.

Then we became default MTA on OpenBSD
-------------------------------------
We started getting complaints from hackers that, although without a value, the BCC header was being emitted by OpenSMTPD.

It seemed rather strange because clients typically strip them from the DATA phase and we never actually receive them. It turned out that some older MUA did not do that stripping and I had to add a basic check to skip BCC line(s) if any appeared in the DATA phase before the message content. I was not too happy about that change, and a few other hackers/people either as it turns out, but it was not really altering content, simply discarding content that we were not meant to receive in the first place so...

Then came another complaint: some hackers were receiving mails from some other hackers ... but the mail had a From header with the local domain instead of the sender domain.

This also seemed strange because the "local domain" is not necessarily the "sender domain". For instance, if you mail me at @opensmtpd.org, the destination domain will be @opensmtpd.org but the local domain will be poolp.org as that's the local domain of the MX machine. This could only mean that the domain had been inserted by the receiving MX because it was missing.

Now, when a mail goes out, it can take two paths:

- it can go through a SMTP connection to localhost;
- or it can use the local enqueuer, `send-mail` or `sendmail`, which are actually both mapped to `smtpctl` working in enqueue mode;

Many MUA support both modes, but usually rely on the second mode by default when ran locally except for a few modern ones that need to be configured explicitly to do so.

What bothered me was that modern MUA usually append a domain so they could pretty much all be ruled out, and older MUA tend to rely on the local enqueuer which has code to insert the local domain if it is missing. In these two cases, the mail would have not been emitted without a domain.

So the logical explanation was that the sender used a MUA that both bypassed the local enqueuer AND did emit a From without a domain. After a bit of testing, this proved to be the case and it brought hell on me.

Addresses Parsing
-----------------
It became clear that if we wanted to continue supporting these older MUA, some basic rewriting was required in the daemon and were no longer just a feature but a mandatory part of the session processing. Without this, an OpenSMTPD server could send mails to another MTA such as Sendmail or Postfix and the recipient would get unexpected headers due to the local rewriting performed by them.

As a quick fix, I wrote a small addresses parser that would extract From, To and Cc fields. It would then attempt to parse the addresses in them, check if a domain was missing and append the local domain in that case. This proved to be a very very bad idea because unlike email addresses in the SMTP protocol, the email addresses inside a message can be written in many ways and are a nightmare to parse. Addresses can span on multiple lines, have comments inside them, they can have brackets or not, they can be surrounded by components, parts of the representation may use different encodings and we even ran into representations such as "Gilles <gilles> Chehade" which goes beyond mind-fuckery.

The issue was not just the parsing, but also that once appended the address had to be rendered back so not only I had to find a way to parse the way-too-many fucked up representations into a structure I could work on, but I had to be able to render the fucked up representation back with the domain inserted in them.

I took a first approach of not doing the replacing in place, but rather constructing a list of sanitized addresses that would be output as a multi-line header. The addresses would be parsed into a common structure and rendered so that (superfluous spaces intended):

    From: "Gilles Chehade"        gilles, "Eric Faurot" eric@poolp.org,     "Charles Longeau" < chl >

would turn into:

    From: "Gilles Chehade" <gilles@poolp.org>,
        "Eric Faurot" <eric@poolp.org>,
        "Charles Longeau" <chl@poolp.org>

But this lead to another round of complaints that the rewriting was altering the message structure, not only did it expand multi-line (which is RFC compliant) but '<' and '>' appeared surrounding the addresses even if they were lacking in the original header.

It became really clear at this point that any attempt at taking a "parse-then-render" approach was not going to work.

Message Parser
--------------
After a discussion with eric@, I decided to take a different approach and split the problem in two smaller problems.

The first one is locating the specific parts of a message that need to be altered, in this case some specific headers. The second one is to actually do the appending somehow.

Locating the specific parts of a message is a seemingly simple issue, however it raises some concerns of its own. The DATA arrives line by line and the headers may span on multiple lines, therefore the processing can't be done as lines arrive but requires a bit of context and knowing which line is the last one ... when this information is actually carried by the next one which has not yet been received.

The simplest way to deal with the problem is to take a full message parser, such as the one we have in the enqueuer, grab the part that deals with the headers and feed it with the full headers for the session in progress.

To achieve this, we could either buffer the headers in memory during the session then work on that buffer or, since we're writing the full message to a file anyway, have the message parser work on that file before it is committed to the queue. Unfortunately, neither of these works with us:

Keeping full headers in memory paves the road for resources starvation attacks. On a default install, a message can be up to 35MB with many people bumping that value further. Depending on your operating system, OpenSMTPD may accept several thousands of concurrent clients. Now, if we went that path, an evil human being could simply craft very large messages consisting solely of headers and have thousands of clients emitting them without sending the final "." but waiting for the server timeout to trigger instead. You get the idea why this is not a good idea.

Working with the file we saved the message to is slightly better, however two issues pop-up.

First, we can't do in-place editing so we will basically be rewriting the entire content to another file. For large messages, the performances penalty will be very visible. Then, we have an atomicity requirement, we can't acknowledge the client that we have accepted the message until it has been committed to queue, so between his final "." and our acknowledgement, he will have to wait for the entire message to be processed, copied to the other file and committed to queue... during that time, he will be consuming a session slot preventing another client from being handled. If he gets lucky, he may even have a chance to hit a timeout causing his disconnect while the server will abort processing of his message and trash it.

So, nope, working with full message or full headers is not the way... data has to be processed as a stream.

Stream Message Parser
---------------------
I discussed the idea with eric@ a bit and then came up with a prototype for a stream message parser that we would plug in the SMTP session.

```c
void    rfc2822_parser_init(struct rfc2822_parser *);
int     rfc2822_parser_feed(struct rfc2822_parser *, const char *);
void    rfc2822_parser_reset(struct rfc2822_parser *);
void    rfc2822_parser_release(struct rfc2822_parser *);
int     rfc2822_header_callback(struct rfc2822_parser *, const char *,
	void (*)(const struct rfc2822_header *, void *), void *);
void    rfc2822_header_default_callback(struct rfc2822_parser *,
	void (*)(const struct rfc2822_header *, void *), void *);
void    rfc2822_body_callback(struct rfc2822_parser *,
	void (*)(const char *, void *), void *);
```

Before, the workflow would basically go like "read a line from this client, write that line to this file".

Now, the SMTP server will associate a parser context to every session and register a set of callbacks. The default callbacks will simply receive the line as parameter and write it to the file, doing exactly what was done before. However, they now get a chance to modify it somehow before writing it.

For example, before the parser was integrated, the BCC remove code I had quicked-fix required adding code to the session reading function to detect if we were still in headers, if there was a continuation line, as well as explicit strcasecmp() checks to bypass the writing. It was not much, but it made it clear that the function would become a big pile of crap once we started adding other special cases.

Now, it is as simple as registering a callback for that header:

```c
rfc2822_header_callback(&s->rfc2822_parser, "bcc",
            header_bcc_callback, s);
```

And writing a callback that ... does nothing:

```c
static void
header_bcc_callback(const struct rfc2822_header *hdr, void *arg)
{
}
```

During a session, the lines read from the clients are passed to the rfc2822_parser_feed function which then takes care of all context handling, buffering and calling registered callbacks when needed. The callbacks will receive headers as a rfc2822_header structure holding a list of lines, and the lines can then either be written as is to a file or modified at will. A message crafted to exhaust resources by providing a single header with an insanely large number of lines will cause the parser to generate an error that the server will properly handle, so all cases solved.

With this in, the code in the SMTP session remains clear with no special cases and no "state" variables all over the place.

So how do we deal with addresses ?
----------------------------------
Let's get back to the main issue: fixing addresses.

So, the message parser has made it possible to extract specific headers and we could now do the following without having to kludge the session reading code:

```c
rfc2822_header_callback(&s->rfc2822_parser, "from",
	header_masquerade_callback, s);
rfc2822_header_callback(&s->rfc2822_parser, "to",
	header_masquerade_callback, s);
rfc2822_header_callback(&s->rfc2822_parser, "cc",
	header_masquerade_callback, s);
```

But how do we implement header_masquerade_callback ?

Instead of using the previous "parse-and-render" approach, I decided to give a try at another way: locate insert points.

While it is hard to parse the header into a series of per-address structures that I can work with and render back, it is not too hard to parse the header into a series of buffers in which I can check if a domain is missing and locate where it should have been. This technique has the advantage that once the processing has been done, the buffers can be written back in sequence preserving the structure of the original message.

For instance:

    From: "Gilles Chehade"      gilles,      "Eric Faurot" eric@poolp.org, "Charles Longeau" < chl >

Would now be parsed into three buffers containing (note that the various superfluous spaces were preserved):

    "Gilles Chehade"      gilles
    "Eric Faurot" eric@poolp.org
    "Charles Longeau" < chl >

The domain-appending logic would then go through the three buffers and determine:

is a domain appending necessary ?
if it is, where in the buffer should it be inserted ?
The second buffer would be unchanged while the first and third buffers would result in an insert point right after gilles and right after chl. The code would then simply copy the buffer to another one up to that insert point, insert the domain, then copy the remaining. Once all buffers have been processed, they are written sequentially resulting in:

    From: "Gilles Chehade"      gilles@poolp.org,      "Eric Faurot" eric@poolp.org, "Charles Longeau" < chl@poolp.org >

Again note that structure has been preserved, header has not be rendered multi-line and superfluous spaces in the original header are left as is.

So... we're all done ?
----------------------
This has been committed several weeks ago and there's been no complaints so I guess the solution was correct.

We've discussed about masquerading a bit and maybe we will reevaluate our position that it would be better off in a filter, after all ... the code to do the domain append is masquerading so if we're going to do it for the incoming case we might as well make it handle the outgoing case too.

However, our stance has not changed and while the message parser offers a lot of ease for in-daemon content altering, we still believe that the bulk of content altering is filters material. If only to benefit from privileges separation while parsing untrusted content.
