---
title: "OpenSMTPD 5.3 released"
date: 2013-03-17 19:08:55
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

OHAI !

I've stayed silent for the last month as we got ready for the big thing...

Charles, Eric and I are proud to announce the release of OpenSMTPD 5.3, our first official production-ready release !

OpenSMTPD has been in use at poolp.org since 2008 and these last five years have seen a considerable amount of work. At various times we have been contemplating a release then delaying to add that one thing that seemed essential. Today, the code we release is still not perfect but we are very happy with it and it has proved to do the work pretty fine. It will continue to improve and mature from now on, and we hope you will enjoy it ;-)

Below is a copy of the release mail, see you soon for updates on new features ;-)

OpenSMTPD 5.3 has just been released.

OpenSMTPD is a FREE implementation of the SMTP protocol with some common extensions. It allows ordinary machines to exchange e-mails with systems speaking the SMTP protocol. It implements a fairly large part of RFC5321 and can already cover a large range of use-cases.

OpenSMTPD has been under development for a long time now and many people are already using it in production, but this is our first stable release and if you were waiting for a GO! from us, here it is ;-)

The archives are now available from the main site at www.OpenSMTPD.org

We would like to thank the OpenSMTPD community for their help in testing the snapshots, reporting bugs, contributing code and packaging for other systems.

Features:
HUMAN READABLE CONFIGURATION
IPv4 and IPv6 support
STARTTLS and SMTPS support for both incoming and outgoing sessions
AUTH support: bsd_auth(3) and crypt(3)
SIZE support: limit the size of client-submitted messages
Listener-specific banner hostname
Listener-specific sessions tagging
Support for global and per-domain expiry for messages
Support for customizable delays for bounces
Support for primary and virtual domains
Support for alternate user database: db(3), file or smtpd.conf
Support for aliases and ~/.forward mappings
Delivery to mbox, maildir or third-party MDA
Support for LMTP relaying
Support for smarthost
Support for sending certificate when connecting to remote host
Support for backup MX
Support for relay source address override
Support for relay HELO override
Support for SMTP-level sender override
Support for connections reuse and optimization
Support for queue backends: filesystem and ram
Support for lookup backends: db(3), static
Run-time statistics through "smtpctl show stats"
Run-time tracing through "smtpctl trace "
Run-time monitoring through "smtpctl monitor"
Experimental:

SQLite lookup backend
LDAP lookup backend
Portable:

Support for PAM authentication
Known to build and work on FreeBSD, NetBSD, DragonFlyBSD and Linux
Limitations:

No filters support yet (work in progress)
No masquerading or address rewrite yet
Checksums:
SHA256 (opensmtpd-5.3.tar.gz) = 05efe80755e7fa01e79e6bba1a4e89244849406acb1152995d2c1da5e9e3a596

SHA256 (opensmtpd-5.3p1.tar.gz) = 618092f1f0b5aba5f8d4c933536a76d3a5a8e45c28b599a6420321cd4478f3d9

Support:
You are encouraged to register to our general purpose mailing-list: http://www.opensmtpd.org/list.html

The "Official" IRC channel for the project is at: #OpenSMTPD @ irc.freenode.net

Reporting Bugs:
Please read http://www.opensmtpd.org/report.html Security bugs should be reported directly to security@opensmtpd.org Other bugs may be reported to bugs@opensmtpd.org

OpenSMTPD is brought to you by Gilles Chehade, Eric Faurot and Charles Longeau.
