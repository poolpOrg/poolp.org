---
title: "February 2021: nooSMTPD libtls-conversion, ciphers, curves and protocols"
date: 2021-02-28 23:00:00 +0200
category: opensource
share_img: "/images/2021-02-01-mq2.jpg"
author: Gilles Chehade
---

<blockquote>
<b>TL;DR:</b>
I converted nooSMTPD to libtls and implemented SMTP ciphers, curves and protocols selection.
</blockquote>


# Shout outs to my patrons !

As usual, a **huge** thanks goes to the people **sponsoring me** on **[patreon](https://www.patreon.com/gilles)**,
**[github](https://github.com/sponsors/poolpOrg)** or **[liberaPay](https://liberapay.com/poolpOrg)**,
the work in this post was made possible by my **[sponsorship](/sponsorship/)**.


# Let's start with some LoFi

Relax.

<center>
    <iframe width="560" height="315" src="https://www.youtube.com/embed/VbEfXZtETvE" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

I have a [youtube channel](https://www.youtube.com/c/GillesChehade) (subscribe ! now !)


# Nothing new in OpenSMTPD-portable

`eric@` sent a libtls-conversion diff to `tech@` last month but **there hasn't been much progress since then**.

I tested the diff extensively and I'm currently waiting for it to be merged to OpenBSD so I can bring it to OpenSMTPD-portable.


# nooSMTPD is libtls-enabled

In nooSMTPD,
I did the libtls-conversion by applying the diff and **making the appropriate changes to the libtls compat layer**.

I switched my machines from my custom FrankenOpenSMTPD to nooSMTPD and been running it since late January,
with no regressions.

It's interesting to note that while I ran OpenSMTPD from upstream repository **without the autoconf build**,
I'm running nooSMTPD **from autoconf** which led to **a lot of swearing** as it was never meant to be used on OpenBSD.
I **fixed a few unpleasant things** which I'll be able to push to OpenSMTPD-portable as well.


# The `smtp tls` global configuration

OpenSMTPD allows setting up SMTP global configuration with the `smtp` global keyword:

```
smtp max-message-size "30M"
smtp sub-addr-delim "+"
```

Because it may be desireable to setup an alternate list of ciphers,
the `ciphers` keyword was introduced a long time ago and a global option allows overriding the default ciphers:

```
smtp ciphers "HIGH:!aNULL:!MD5"
```

In nooSMTPD,
I decided to introduce the `tls` keyword for TLS related options.

The libtls conversion unlocks several TLS-related features that may be confusing below `smtp`,
without any reference to TLS.
For instance,
I have implemented TLS protocol selection and unless `tls` appears in the configuration this will result in:

```
smtp protocols [...]
```

which is much more confusing than:

```
smtp tls protocols [...]
```

as it is unclear that the protocols are TLS related.


As a result,
I moved `ciphers` below `smtp tls` so in nooSMTPD it reads as:

```
smtp tls ciphers "HIGH:!aNULL:!MD5"
```

I'll suggest this change to OpenSMTPD too.




# TLS ciphers selection

As mentionned above,
I have changed the grammar to introduce a `tls` namespace to the `smtp` keyword:

```
smtp tls ciphers "HIGH:!aNULL:!MD5"
```

Because libtls supports cipher suite groups,
the `secure`, `compat`, `legacy` or `insecure` groups can be used in place of a cipher suite,
and since `secure` is the default,
the global cipher selection only needs to be specified for other cases:

```
smtp tls ciphers "compat"
```

With this done,
I also added the ability to setup the ciphers at the listener level:

```
listen on 0.0.0.0 tls ciphers "compat"
listen on 192.168.0.1 tls ciphers "legacy"  # use on legacy subnet
```

This was committed to nooSMTPD and I'll submit a diff to OpenSMPTD when libtls-conversion is done upstream.


# TLS curves selection

The selection of curves was never merged in OpenSMTPD,
as a result **it is not possible to select specific curves at either one of the global or listener level**.
The default curves (`X25519,P-256,P-384`) in libtls are **sensible choices** but some use-cases require restricting to a specific curve,
or even an alternate one.

I introduced a `curves` option which allows specifying curves to override the default:

```
smtp tls curves "X25519"
```

While at it,
I added the ability to setup the curves at the listener level:

```
listen on 0.0.0.0 tls ciphers "X25519"
listen on 192.168.0.1 tls curves "P-256,P-384"
```

This was committed to nooSMTPD and I'll also submit a diff to OpenSMPTD when libtls-conversion is done upstream.


# TLS protocols selection

Just like for curves,
the selection of a TLS protocol was never merged in OpenSMTPD.

I introduce a `protocols` option which allows specifiying protocols to override the default:

```
smtp tls protocols "tlsv1.3:tlsv1.2"
```

And again,
I added the ability to setup the protocol at the listener level:

```
listen on 0.0.0.0 tls protocols "tlsv1.3"
```

Guess what...
Yes, this was also committed to nooSMTPD and I'll submit a diff to OpenSMPTD when libtls-conversion is done upstream.


# Global `smtp tls` grammar needs to be reworked

The `smtp tls` keyword was implemented as a placeholder for the `ciphers`, `curves` and `protocols` options,
but it was not implemented the way it should.

Basically,
`ciphers`, `curves` and `protocols` should be options to `tls` so that it is possible to write:

```
smtp tls ciphers "secure" protocols "secure"
```

At the time being,
this is not possible and it must be expressed as follows:

```
smtp tls ciphers "secure"
smtp tls protocols "secure"
```

This requires a bit of grammar refactor which I'll be working on in March.


# Listener `tls` grammar was reworked

The listener `tls` keyword suffered from the same issue as the global `smtp tls` keyword.

Initially,
it was implemented in such a way that `ciphers`, `curves` and `protocols` were **standalone options**.
This allowed avoiding the need to repeat `tls` before every option,
you needed one `tls` on the listener so it was flagged as TLS-enabled and the options would check for the flag.
This worked but it **was not the right way** to do it.

I reworked the grammar for `tls` in listeners so that `ciphers`, `curves` and `protocols` are `tls` options.
This avoids having to repeat `tls` before every keyword,
and it allows relying on the grammar rather than on a flag to determine if the configuration is valid:

```
listen on 0.0.0.0 tls ciphers "secure" curves "P-256,P-384" protocols "secure"
```

I'm happy with the result.


# What's next ?

I will rework the grammar for global settings and will implement these options for the MTA layer,
as these options were done only for the incoming path.

I intend to publish a technical article in March regarding software design and data.

I might unveil a project that's unrelated to computer programming too :-)

---- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/85](https://github.com/poolpOrg/poolp.org/issues/85)
