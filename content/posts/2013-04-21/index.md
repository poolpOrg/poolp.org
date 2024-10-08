---
title: "News from the front"
date: 2013-04-21 17:15:21
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

OHAI,

I have not updated this blog since the release of OpenSMTPD 5.3, over a month ago. In the meantime, some minor bugs have been fixed and we have released OpenSMTPD 5.3.1.

The main reason for the silence is that I was busy with completely unrelated day-time work, unrelated opensource code, the OpenSMTPD book in progress and some minor health issues that have caused endless side-effects keeping my mind away from this blog.

Anyway, starting next week I should be actively blogging about developments in progress as we have resumed working on new features that I'm looking forward to see committed to master. For the time being, here's a few features that have been committed or that are being worked on actively at the moment:

helo no longer requires source The "helo" features allows OpenSMTPD to advertise a particular helo hostname when relaying to another MX. It does so by looking in a table for a helo entry matching the IP address that it has used to connect to the MX. The feature required that both a source table AND a helo table be used for the helo feature to work, but this was an unrequired constraint and prevented some setups to be possible as reported by Todd Fries, so I simply removed the constraint. This has been committed to master and will be part of new snapshots.

"error" alias type When writing code, we often have to test how be behave when facing a success, permanent failure or temporary failure. To achieve this, we usually create mail accounts that discard content, that have broken ~/.forward files (temporary failure) or we simply mail non-existent accounts (permanent failure). Todd had another use case where he wanted a virtual domain to have accounts that returned errors. I implemented a new alias type, "error", which allows an alias to expand to an error code and message:

table aliases { foobar = "error:550 you're not allowed to mail foobar", barbaz = "error:450 woops, temporary failure" }

The error alias type can be used for regular aliases, virtual domains as well as ~/.forward files if a user temporarily wants to bounce mails to his account. It only supports 5xx and 4xx codes. It has been committed to master and will be part of new snapshots.

Offline queue is broken Charles Longeau had spotted a strange issue where mail submitted while the daemon was offline would not be enqueued correctly when the daemon would start. I had never seen this but my OpenSMTPD instance is almost never down so the offline enqueuer is barely used. I tried to reproduce without success and eventually managed to hit the issue.

It turns out that this was only a permission problem and that mails submitted by the root user would manage to be enqueued, while mails submitted by other users would fail and remain in the offline queue. The issue has been fixed, committed and upcoming snapshots will have the fix.

Queue spin Elbarto, a FreeBSD user, has reported that after changing permissions of files in the queue, he got the queue process to go crazy and spin. This looked strange because OpenSMTPD checks permissions at startup and should warn the admin if they are inappropriate. It turns out that the issue happened when the admin would change permissions of some directories in the spooler AFTER startup of the daemon.

This is an uncommon case and people should never hit in regular use, but the daemon should cope with these basic errors and not go crazy. It took some time to reproduce but I eventually found the culprit and came up with a fix that was committed. It will be part of upcoming snapshots.

Log envelope source A user reported that we didn't log the IP address that was used during a relaying when we log the status of the envelope. In setups that make use of multiple IP addresses, it makes it harder to track the path of an envelope while adding the source address is trivial. The log format for envelopes now contain an additional "source" field with the IP address that was used for the relaying. It will be part of new snapshots.

IPv6 over IPv4 preference ftigeot@ from DragonFlyBSD reported strange behaviour where IPv6 was tried AFTER IPv4 which is inappropriate on that system. This behaviour is caused by OpenBSD's default IPv4 before IPv6 search strategy, which turns out to be quite the opposite of what other systems assume. We decided to have portable OpenSMTPD use IPv6 before IPv4 strategy since that is what most people expect. Charles committed the fix to ASR with Eric's and my approval.

K_ADDRNAME not supported by db backend Todd Fries reported that he wasn't able to use a db(3)-backed table as a helo table. It turns out that the db backend didn't support the K_ADDRNAME service. The fix was trivial and committed shortly after. New snapshots will support it.

Encrypted queue I have started working again on encrypted queue and the implementation is currently being discussed with other OpenBSD hackers. I'm expecting it to be committed within the next two weeks. It will allow people to have their mail queue transparently encrypted and protected against tampering. It has absolutely no use in a self-hosted environment, but will allow users to host their own mail servers while delegating the hosting of the queue to a company they trust for hosting but not for privacy (ie: I'd trust Google to store my queue safely, but I don't trust them not to look at it).

Queue encryption is compatible with queue compression and can be enabled with:

queue encryption key "foobarbaz"

The key will be expanded using 100.000 iterations of pkcs5_pbkdf2 and a random salt for each envelope and message. Each envelope and message will be encrypted with AES-256 in GCM mode, providing encryption and integrity, with a random IV for each. I'm looking forward to that feature :-)

What's coming next ? A big surprise. Eric is working on a feature that I've been wanting for MONTHS. It will be huge.

Stay tuned ;-)
