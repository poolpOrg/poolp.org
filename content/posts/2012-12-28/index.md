---
title: "OpenSMTPD: SSL & relay stuff"
date: 2012-12-28 20:11:23
category: OpenSMTPD
authors:
 - Gilles Chehade
---

OHAI,

I was unable to write a story last week as my desk broke into pieces for no apparent reason but my foot hitting it accidentally.

I still don't have a desk so I will make this short by not telling about the queue profiling and SQL log conversion I wrote, and not even writing about the filter work done by Eric, he'll write about it if he wants to or I'll do that next time.

You guys enjoy New Year's Eve, don't get too drunk and remember to send tons of mails to those not around using an awesome piece of software :-p

SSL verification and separation

OpenSMTPD has had code to deal with establishing secure channels for a very long time now, however what it didn't do was to perform certificate chain verification in both server and client modes. In server mode, it would never request a client certificate and in client mode it would never check that the certificate handed by the server was valid.

So I started adding support and hit the first issue: access to the CA bundle from within the chroot. After a discussion with reyk@ over what was the best way to deal with it, he convinced me that we could really improve our design by moving certs and keys from network-facing processes to a separate process and performing on-demand requests.

Dealing with OpenSSL was as nice as usual, it almost seemed like sharing a tasty dinner with Richard Stallman, but I eventually saw the light and got it to work as expected. The good part is that the client and server modes have symmetric operations which means that the code is identical for the most part.

We now have the sensitive stuff isolated in the lookup process. The smtp and mta processes use the imsg framework to request them and to pass over chains for verifications. It all fits in a very few lines of code which is cool because the less OpenSSL code I have to deal with, the better.

For now, we don't do a full verification of the X509 attributes so OpenSMTPD lies about the verification and pretends it didn't do it in the Received lines, however the admin can see the verification take place in the logs. I'll fix the Received line to tell the truth when I'm confident the verification is 100% accurate.

Now, before I switch subject, a couple related ideas for the future:

If we were to provide a K_CERT table service, we could fetch the certificates and chains from custom backends (ldap, sql, you name it, ...), this would take approximately one hour of work. The only reason I'm not doing it is that I don't have a need for that at the moment ;-)

Also, the certificate verification reply to mta and smtp is a simple status that is either success or failure. This means that implementing a certificate-based authentication is now trivial, probably an hour of work too. Guess why I didn't implement it yet ?

Fix relaying logic

While working on the SSL code, I spotted strange behaviour when relaying between my primary and secondary MX.

After some testing, I realized that we lost the "backup" flag somewhere. After a quick chat with eric@, I convinced him we should introduce the backup:// schema and get rid of the flag in the envelope. This way, we could make it obvious that mx2.poolp.org is a backup MX by having backup://mx2.poolp.org as a relay URL.

Then, I spotted some strange issues where I didn't request TLS and it would attempt it, or where I requested TLS but it would fallback to plaintext. It became obvious there was something fishy with the semantic of our relay URL schemas and that they needed to be more clearly defined.

After going through every possible schema, we defined them as follow:

smtp+tls:// -> attempt TLS, then fallback to plaintext, this is the default smtp:// -> plain text ONLY, no encryption tls:// -> STARTTLS only, encrypted channel guaranteed smtps:// -> SMTPS only, encrypted channel guaranteed ssl:// -> STARTTLS, fallback to SMTPS, encrypted channel guaranteed

With this, unless smtp+tls:// is specified, we never fallback to plaintext if an encrypted channel is requested and we never break the user assumption that a relaying will be secure by sending the data over a plaintext channel.

I guess that's all for this time, I really need to zZzZ
