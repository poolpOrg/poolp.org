---
title: "Some OpenSMTPD news"
date: 2012-05-09 23:03:36
category: ["OpenSMTPD"]
authors:
 - Gilles Chehade
categories:
 - technology
---

It's been a whiiiiiiile since my last OpenSMTPD related post. Time to break the slacking, blog-wise.

Unlike what would appear from the outside, there's been a lot of activity, discussions and planning, I'll address some here and will discuss it further soon.

First of all, chl@ has committed to a private branch a major improvement to the filter API: async DNS lookups. Due to the design of OpenSMTPD, it is not acceptable for a filter to perform a blocking call as it will prevent it from processing other sessions. To solve this, one would have to link the filter against eric@'s resolver or fork a process per lookup and add complex code to simulate async behaviour.

When we discussed it, chl@ or I (can't recall who) proposed that queries be rerouted through the OpenSMTPD's LKA process so that it performs the lookup on behalf of the filter. I suggested that filters register a callback upon return of the DNS query and chl@ implemented it. He came up with a PoC working DNSBL filter which proves the solution works and we'll provide a real implementation later.

eric@ committed ASR to OpenBSD's libc, which will allow us to cleanup the code base for OpenSMTPD as it will no longer require shipping an async resolver. He also made various fixes and cleanups here and there.

Since r2k12, we've had discussions on design improvements and refactoring of some parts of OpenSMTPD. I've planned some changes in the scheduler API and a PoC SQLite backend for maps/scheduler/queue as SQLite is now in base and it will allow us to ensure the various backends API are well designed. I also have plans to completely refactor the statistics collecting by providing a backend API, removing the shared mmap-ed page and turning the static counters to a more dynamic structure allowing statistics to be collected at runtime (ie: collect all deliveries for a virtual host, something not doable currently). Also, I have a plan to allow the use of mappings in several contexts where it's not doable yet, like source address and MX relays, which would allow us some very nice features.

eric@ provided me with a doc describing the refactors he plans on MTA after discussions we had, which will allow OpenSMTPD to perform much more efficient and smarter remote deliveries.

Stay tuned, awesome stuff coming soon ...
