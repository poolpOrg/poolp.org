---
title: May 2019 report
date: 2019-06-02 14:57:00
category: opensource
author: Gilles Chehade
categories:
 - technology
---

{{< tldr >}}
    In this post I explain crudely how ca.c works and changes to OpenSMTPD related to ca.c
    I wrote an ECDSA privsep crypto engine
    I did some EEG work too
{{< /tldr >}}


This is the first report
--
I will now switch to a monthly report of my tech activities on this blog,
and this is the first post in that new format.

The focus will be put on interesting topics,
not necessarily every single commit and bug fix I do (ie: won't mention LMTP bug fix here),
this depends on the amount of slacking that ocurred during the month and how much I want to cover it up :-)

Don't forget to tip me on the link above if you like reading my reports,
donations cover my random geek expenses and keep me motivated to write about what I do.


About OpenSMTPD's ca.c file
--
A few years ago,
Neel Mehta blessed us with the Heartbleed bug in the OpenSSL library.
A bug which ultimately resulted in the disclosure of sensitive data,
including cryptographic private keys,
that resided in the memory of OpenSSL's custom allocator.

A good approach at mitigating the issue was taken by the LibreSSL folks.
Long story short,
a custom allocator is a bad idea because it potentially defeats any security measure provided by the standard allocator.
In OpenBSD,
the multiple security mechanisms in place to detect memory corruption and invalid access were bypassed by the use of the OpenSSL custom allocator,
and the first steps taken by LibreSSL were to ensure that functions such as `OpenSSL_malloc` and `OpenSSL_free` where REALLY calling `malloc` and `free`,
and not something else trying to be "smart".

Another approach was taken by `reyk@` and consisted in assuming that private keys in a network facing process was too dangerous.
To mitigate the risk,
a good option would be to isolate the crypto operations that required access to the private key in a separate process,
one that would be fairly isolated from untrusted users.
Here comes `ca.c`,
a process operating privileges separated crypto operations.

Here's how it works:

Upon startup,
OpenSMTPD declares a custom RSA engine which provides its own primitives for crypto operations.
It then registers that custom engine as the default RSA engine,
meaning that any use of the OpenSSL API to perform RSA operations will use the new primitives rather than the default ones.

The primitives provided by the RSA engine are actually of two kind:
either they rely on the private key in which case they will perform an IPC to the CA (crypti agent) process which will do the operation itself through the default primitive and provide the result,
or they don't rely on the private key in which case the primitive simply falls back to the default one.

```c
[...]

static int
rsae_pub_dec(int flen,const unsigned char *from, unsigned char *to, RSA *rsa,
    int padding)
{
	log_debug("debug: %s: %s", proc_name(smtpd_process), __func__);
	return (rsa_default->rsa_pub_dec(flen, from, to, rsa, padding));
}

static int
rsae_priv_enc(int flen, const unsigned char *from, unsigned char *to, RSA *rsa,
    int padding)
{
	log_debug("debug: %s: %s", proc_name(smtpd_process), __func__);
    	if (RSA_get_ex_data(rsa, 0) != NULL) {
        	return (rsae_send_imsg(flen, from, to, rsa, padding,
            		IMSG_CA_PRIVENC));
    	}
    	return (rsa_default->rsa_priv_enc(flen, from, to, rsa, padding));
}

[....]
```

The idea is that if the network facing process is compromised,
then no matter how bad the memory corruption and/or info leak is,
the private keys are not available to that process.


Fix OpenSMTPD ca.c build with OpenSSL
--
That private key privsep relies on using the OpenSSL crypto engine API.

Ever since OpenSSL switched to 1.1.x,
the crypto engine API changed because structures that were public and could be dereferenced would no longer be public.
They would require a set of getters and setters accessors to make any use of the structure.
The sad thing is that this is not the case with LibreSSL,
the structures are still public and accessors are not available.

This results in the same code not being able to be built for both,
either you use the opaque structures and build is broken on LibreSSL,
or you use the public structures and build is broken on OpenSSL.

I rewrote ca.c to pretend that the structure was opaque,
not performing any direct deref,
and added the missing accessors within an ifdef.
This is not ideal but it reduces the delta between our native branch and portable,
and as soon as accessors are available in LibreSSL we can simply remove the ifdef-ed part.

I submitted the diff to my fellow OpenBSD hackers,
it's pending review at this point,
should be committed soon.


Turn most of OpenSMTPD's SSL structures opaque
--
A lot of other structures that are also opaque did have accessors in LibreSSL,
notably the EVP interface which we use for encrypted queue support.

The code used to rely on automatic variables and dereferences,
I converted the code to heap allocated structures using the opaque structures interface.

Diff was submitted, okayed and committed.


Add support for opaque RSA_METHOD engine in LibreSSL
--
As mentioned earlier,
code in ca.c was broken due to the fact that LibreSSL considers RSA_METHOD as a public structure that can be dereferenced,
whereas OpenSSL considers it as an opaque structure that requires accessors.

Even considering that RSA_METHOD is a public structure,
this doesn't prevent us from providing accessors to allow a common idiom between LibreSSL and OpenSSL.

I submitted a diff to the LibreSSL folks to bring the missing accessors,
which will allow me to write code that builds for both LibreSSL (my target) and OpenSSL.
This will not be an OpenSMTPD only thing because any OpenBSD daemon that relies or will rely on ca.c and wants to be portable will face the same issue.

The diff has been reviewed and okayd,
it will require a LibreSSL (libcrypto, libssl and libtls) minor crank so I'm waiting for the appropriate time to do it.
I'm also waiting for an ok from ports to make sure it doesn't break ports as adding new symbols may have some configure scripts uncover other symbols we don't have.


Write an ECDSA privsep engine for OpenSMTPD
--
The crypto privsep engine only supported RSA and this means that starting OpenSMTPD with an ECDSA results in a fatal():
```
debug: init private ssl-tree
debug: SSL library error: ssl_ctx_create: error:06FFF07F:digital envelope routines:CRYPTO_internal:expecting an rsa key
debug: SSL library error: ssl_ctx_create: error:140AE006:SSL routines:func(174):EVP lib
pony express: ssl_ctx_create: could not fake private key: Invalid argument
```

More and more people have complained that we didn't support ECDSA certificates,
but it was not a trivial one-liner as adding support meant writing a custom ECDSA crypto engine akin to what `reyk@` did for RSA.
It took me time to find the motivation,
but ultimately I came around to implement such an engine this week.

```
debug: rsa_engine_init: using RSA privsep engine
debug: ecdsa_engine_init: using ECDSA privsep engine
```

The details of how I implemented the ECDSA crypto engine are not that interesting,
they follow the same pattern as the RSA crypto engine:

```c
ECDSA_SIG *
ecdsae_do_sign(const unsigned char *dgst, int dgst_len,
    const BIGNUM *inv, const BIGNUM *rp, EC_KEY *eckey)
{
        log_debug("debug: %s: %s", proc_name(smtpd_process), __func__);
        if (ECDSA_get_ex_data(eckey, 0) != NULL)
                return (ecdsae_send_enc_imsg(dgst, dgst_len, inv, rp, eckey));
        return (ecdsa_default->ecdsa_do_sign(dgst, dgst_len, inv, rp, eckey));
}
```

The real challenge was to teach ca.c that multiple crypto engines could coexist,
figuring out the primitives for ECDSA and doing the proper serialization in IPC.

On my laptop,
OpenSMTPD can now load and work with both RSA and ECDSA certificates,
accepting SMTP sessions on listeners presenting either one:

```
9362d76af3ab2882 smtp connected address=127.0.0.1 host=localhost
debug: looking up pki "localhost"
debug: session_start_ssl: switching to SSL
debug: pony: rsae_priv_enc
9362d76af3ab2882 smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256

8c25253d498c0990 smtp connected address=127.0.0.1 host=localhost
debug: looking up pki "localhost"
debug: session_start_ssl: switching to SSL
debug: pony: ecdsae_do_sign
8c25253d498c0990 smtp tls ciphers=TLSv1.2:ECDHE-ECDSA-AES256-GCM-SHA384:256
```

I have submitted the diff today,
pending review,
it should be committed shortly.


Troubleshoot `eegreader` and write initial support for an `eegviewer`
--
A while ago,
I got interested in reading electroencephalogram data to detect interesting patterns during self-hypnosis sessions.

I got myself an OpenEEG device and realized there were no simple utilities to extract data from it,
most were C++ bloated software that didn't build on my laptop and required dependencies.

I found out that the OpenEEG protocol was very easy to implement and wrote `eegreader` a utility to output an OpenEEG data stream to stdout,
displaying sequence, version, the six channels and buttons values.

There seemed to be a bug because packet loss kept triggering my recovery code,
skipping bits until finding the alignment for a new packet (eg: packets 53, 56, 58, 5a, 5c are missing in this sample):
```
[...]
4f|2|512|511|505|499|493|487|7
50|2|532|511|505|500|493|487|7
51|2|513|65281|63745|62465|60673|59143|5
53|2|497|511|505|499|493|487|7
54|2|508|511|505|499|493|487|7
55|2|513|65281|63745|62465|60929|59143|5
57|2|514|511|505|500|494|487|5
59|2|506|511|505|500|494|487|5
5b|2|535|511|505|500|494|487|5
5d|2|499|511|505|500|493|487|5
[...]
```

I investigated and realized that I didn't have the issue on some other OSes,
it seems to be an OpenBSD USB bug.
Furthermore,
with the latest OpenBSD snapshot,
I seem to be hitting kernel panics in USB so some investigation is definitely needed.
I pinged some of the USB gurus, we'll see how I can help.

I still managed to do something useful as I started the writing of `eegviewer`,
which is a `libxcb` graphical interface displaying the `eegreader` input as a real-time graph.
The idea is to simply pipe `eegreader` into `eegviewer` to obtain the graphs.

![device](device.jpeg)
![eegreader](eegreader.png)
![eegviewer](eegviewer.png)

The `eegreader` utility has already been published a while ago on Github under the ISC license.
I'll publish `eegviewer` under the same license when I'm happy with the code, libxcb not being something I'm familiar with.

Finally,
I will very likely write an ISC-licensed `libeeg` which will provide an event-driven interface to OpenEEG data streams,
allowing callbacks to be triggered upon certain events occuring on certain channels.
I haven't started that work yet but it is trivial so I might release that soon.


Oculus Quest
--
Thanks to my geek donations,
I got myself an Oculus Quest device :-)

I'm not a gamer but I'm genuinely curious what can be done with it,
when time allows I'll start digging the SDK.


What next ?
--
See you in a month for the June report,
very likely focusing around OpenSMTPD filters !
