---
title: "OpenSMTPD: crypto/compress fixes and import initial stab at LDAP"
date: 2012-08-31 15:21:39
category: OpenSMTPD
author: Gilles Chehade
---

The crypto backend that has been committed a couple days ago started a little discussion and it was decided to switch the default cipher from Blowfish to AES-128 and make it a default.

Encryption can now be enabled as simply as:

queue encryption key "foobar"

That being said, Charles cleaned up a bit the compression code to make it use FILE* instead of file descriptors. This allows us to ensure we don't have to deal with any buffering and interruption handling code.

The fread/fwrite buffer sizes have been increased to make compression and encryption more efficient and have them spend last time in IO.

Finally, Mathieu (naabed- on #opensmtpd) has played puzzle with some old code of mine to bring LDAP support and provide a map_ldap backend. It's not ready yet, but we imported code in the tree so that it can be worked on more efficiently.

Stay tuned !
