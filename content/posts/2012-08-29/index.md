---
title: "OpenSMTPD: crypto_backend and encrypted queue"
date: 2012-08-29 10:21:59
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

A few days ago, Charles committed the compress_backend API which allowed transparent deflation/inflation of envelopes and messages as they hit the queue.

The compression code is executed before the queue_backend gets the data so that any queue_backend can benefit transparently from compression without having to add any code to handle it. Due to our design, this also means that if the queue_backend stores envelopes and messages remotely, then the compression will take place before sending and after fetching, the remote end only ever sees compressed data.

The idea with the compress_backend was to not only to save disk-space on the queue, which is a very worthy improvement for hosts that deal with very large queues; but also to prove a point: the queue_backends can be made agnostic of the envelopes and messages content, effectively handling blobs.

And a side-effect of that is that since queue_backends don't need to inspect envelopes and messages, and since they can store data remotely, we could provide full queue encryption and ensure that the remote end only ever sees encrypted data.

The crypto_backend provides a transparent encryption/decryption service that can be used in any part of OpenSMTPD. As of now, the queue is the only consumer but we already have other use-cases and probably some ideas we haven't come up with yet will pop-up months from now ;-)

Of course, encryption is tricky so the configuration is hard ... NOT !

<b>queue encryption key "foobar"</b>
This will enable transparent encryption of the queue using Blowfish in CBC mode with a random IV and will internally expand the key "foobar" using sha256. This is just to provide sane defaults, one could override the cipher and digest using the following:

<b>queue encryption key "foobar" cipher aes-128-cbc digest sha512</b>
The IV is randomly chosen for every envelope and message, it is then encrypted and prepended to the message. This ensures that the same key used to encrypt the same envelope will produce different output.

Things will still be improved further but in my opinion it is already covering the use-cases of many who would need an encrypted queue, and to be honest with a quick look on the intertubez I couldn't find a competitor to compare to on that particular feature so we'll move it forward as we hit use-cases.

Oh... and of course this does work along the compression support so you can have both encryption and compression enabled.

This should hit the OpenBSD tree pretty shortly, this evening for sure.
