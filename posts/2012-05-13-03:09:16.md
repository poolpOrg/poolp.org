---
title: "OpenSMTPD, map_compare() and K_NETADDR"
date: 2012-05-13 03:09:16
category: OpenSMTPD
author: Gilles Chehade
---

Ok, I already posted about the new SQLite map_backend earlier today but it turns out that I've been very productive, and it was worth another post ;-)

So I decided to start improving our maps support and make them usable from places where they weren't.

primary domains

OpenSMTPD supports two kinds of domains: primary domains and virtual domains. The primary domains do not require providing a list of recipients as it will use system accounts; whereas virtual domains require a list of recipients and/or a fallback address to be provided within a map.

Until now, primary domains were static, they had to be listed one by rule:

accept for domain "opensmtpd.org" deliver to mbox accept for domain "poolp.org" deliver to mbox accept for domain "pool.ps" deliver to mbox

I fixed a few things so that maps can be used and while you can still list one rule for each domain, you can simplify your ruleset by referencing a map:

map "pdoms" source plain "/etc/mail/pdoms.txt"

accept for domain map "pdoms" deliver to mbox

With this new configuration, OpenSMTPD does not need to be restarted to add or remove a primary domain, they can be changed within the map directly.

map_compare()

The map_backend API is very simple and consists in 3 calls: map_open() to open the map, map_lookup() to find a value associated to a key, map_close() to cose the map.

The map_lookup() performs an exact match on the key so it can determine if x@foobar.org exists, but it cannot determine if any address ends with that domain. For a particular feature I needed to be able to check if a key was a subset of any key in a map.

I came up with map_compare() which is a function that will take a map, a key, a map kind and a function to compare the key with each of the keys sequentially. The comparison ends as soon as the function succeeds so that the sequential scan can be aborted early if necessary.

The API is very simple but inefficient as it should be reserved for those cases were you HAVE to iterate over relatively small sets. You'll understand in a minute ;-)

K_NETADDR map kind

Until now, OpenSMTPD used inet addresses in only one place: as a parameter to from:

accept from "127.0.0.1" [...]

This will change soon so it became necessary to have a map kind to represent inet addresses. I introduced the K_NETADDR map kind which can be used to represent either an address or a netmask for both AF_INET and AF_INET6 (though AF_INET6 netmasks are unsupported on OpenBSD yet [I have a diff]).

With this new map kind I decided to go and simplify the parse.y file which was doing much more than it should be doing as you can see.

Now, not only our parse.y file got much cleaner and easier to read, but the text -> inet conversion has been isolated to util.c, and the ruleset matching has been improved to support ... maps. It is now possible to:

map "src" source plain "/etc/mail/trusted"

 accept from map "src" for all relay

With /etc/mail/trusted containing a set of addresses and netmasks:

127.0.0.1 ::1 192.168.1.0/24

Now the need to iterate over all entries of a map becomes more clear :-)

I have other goodies but I'm exhausted so it'll have to wait...

Stay tuned !
