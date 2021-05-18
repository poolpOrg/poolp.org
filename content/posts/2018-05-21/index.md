---
title: "switching to OpenSMTPD new config"
date: 2018-05-21 18:32:00
category: OpenSMTPD
author: Gilles Chehade
---

    TL;DR:
    Switching to new config is not too hard and can be done in minutes.
    The new config is also a new queue that is not backwards compatible.
    The easiest way is to flush the mail queue before switching.
    We came up with a solution to help maintainers of more complex setups.


Switching from old config to new config
--
The new OpenSMTPD configuration grammar is slightly different from the current one,
rules are no longer stated as single lines,
but the conversion from previous ruleset to new ruleset isn't that hard.

Let's do the exercise with poolp.org's smtpd.conf which is a fairly complex ruleset,
making use of several features including TLS,
authentication,
multi-domain hosting with primary and virtual domains,
different aliases mappings,
backup MX,
relaying through a DKIM proxy,
and more...
```
pki mx1.poolp.org certificate "/etc/ssl/poolp.org.fullchain.pem"
pki mx1.poolp.org key "/etc/ssl/private/poolp.org.key"

pki mail.poolp.org certificate "/etc/ssl/poolp.org.fullchain.pem"
pki mail.poolp.org key "/etc/ssl/private/poolp.org.key"

table sources           { 212.83.181.8 }
table helonames         { 212.83.181.8 = mx1.poolp.org }

table aliases           "/etc/mail/aliases"
table opensmtpd-aliases "/etc/mail/aliases-opensmtpd.org"
table pdomains          "/etc/mail/primary-domains"
table vdomains          "/etc/mail/virtual-domains"
table vusers            "/etc/mail/virtual-users"
table bdomains          "/etc/mail/backup-domains"

table shithole		{ "@qq.com" }

listen on lo0
listen on lo0 port 10028 tag DKIM
listen on egress tls-require pki mx1.poolp.org hostnames { 212.83.181.7 = mail.poolp.org, 212.83.181.8 = mx1.poolp.org }
listen on egress smtps pki mail.poolp.org auth hostname mail.poolp.org
listen on egress port submission tls-require pki mail.poolp.org auth hostname mail.poolp.org

reject from any sender <shithole> for any

accept for local alias <aliases> deliver to maildir
accept from any for domain <pdomains> alias <aliases> deliver to maildir
accept from any for domain opensmtpd.org alias <opensmtpd-aliases> deliver to maildir
accept from any for domain <vdomains> virtual <vusers> deliver to maildir
accept from any for domain <bdomains> relay backup mx1.poolp.org

accept tagged DKIM for any relay source <sources> hostnames <helonames>
accept for any relay via smtp://127.0.0.1:10027
```

Fixing `pki` directives
--
First of all,
the `pki` directives which contain the certificates and private keys are not really affected.
They work exactly as with the old grammar,
we only shortened the `certificate` keyword to `cert` by popular demand.
This results in:
```
pki mx1.poolp.org certificate "/etc/ssl/poolp.org.fullchain.pem"
pki mx1.poolp.org key "/etc/ssl/private/poolp.org.key"

pki mail.poolp.org certificate "/etc/ssl/poolp.org.fullchain.pem"
pki mail.poolp.org key "/etc/ssl/private/poolp.org.key"
```

being rewritten as:
```
pki mx1.poolp.org cert "/etc/ssl/poolp.org.fullchain.pem"
pki mx1.poolp.org key "/etc/ssl/private/poolp.org.key"

pki mail.poolp.org cert "/etc/ssl/poolp.org.fullchain.pem"
pki mail.poolp.org key "/etc/ssl/private/poolp.org.key"
```

The `ca` directive which allows setting a custom CA certificate,
and which is not used in this configuration file,
is subjected to the same change so if you have a directive such as:
```
ca mail.poolp.org certificate "/etc/ssl/poolp.org.ca"
```

all you have to do is replace `certificate` with `cert`:
```
ca mail.poolp.org cert "/etc/ssl/poolp.org.ca"
```

Other `pki` options are unchanged and default to the sanest option.


Fixing `table` directives
--
![Good news, everyone!](https://brillianceinsight.files.wordpress.com/2014/02/card-2-24-2014-1_thumb.jpg)

No changes in `table` directives:
```
table sources           { 212.83.181.8 }
table helonames         { 212.83.181.8 = mx1.poolp.org }

table aliases           "/etc/mail/aliases"
table opensmtpd-aliases "/etc/mail/aliases-opensmtpd.org"
table pdomains          "/etc/mail/primary-domains"
table vdomains          "/etc/mail/virtual-domains"
table vusers            "/etc/mail/virtual-users"
table bdomains          "/etc/mail/backup-domains"

table shithole		{ "@qq.com" }
```


Fixing `listen` directives
--
The `listen` directive works as before, however like for `pki` some keywords were shortened.
For instance, `mask-source`, which is not used in my config, was shortened to the more compact versions `mask-src`.

In my smtpd.conf, there was no change to `listen` directives despite using a large subset of `listen` features:
```
listen on lo0
listen on lo0 port 10028 tag DKIM
listen on egress tls-require pki mx1.poolp.org hostnames { 212.83.181.7 = mail.poolp.org, 212.83.181.8 = mx1.poolp.org }
listen on egress smtps pki mail.poolp.org auth hostname mail.poolp.org
listen on egress port submission tls-require pki mail.poolp.org auth hostname mail.poolp.org
```


Fixing the ruleset
--
This leaves us with the complex part of the change,
switching from the one-line rules that used to define a decision, an envelope matching pattern and an action as a whole,
to the new two-line rules defining a set of valid actions and a distinct set of matching patterns referencing the actions.

Don't worry, be happy.
The change is mechanical and doesn't need to completely rethink how you used to do your configuration files.
The rules are now split into two parts, the actions and the matching patterns.
The actions must be declared before matching patterns but may be declared in any order.
The matching patterns work like previous ruleset,
they are listed in 'first match wins' order and therefore require the most specific rules first as the ruleset is essentially cascading on mismatches.
There is no change whatsoever to that logic,
so converting a former ruleset to a new ruleset is basically...
splitting previous ruleset into `action` and `match` directives with the `match` directives following the exact same order as with current ruleset.

In my case, the following ruleset needs to be rewritten:
```
reject from any sender <shithole> for any

accept for local alias <aliases> deliver to maildir
accept from any for domain <pdomains> alias <aliases> deliver to maildir
accept from any for domain opensmtpd.org alias <opensmtpd-aliases> deliver to maildir
accept from any for domain <vdomains> virtual <vusers> deliver to maildir
accept from any for domain <bdomains> relay backup mx1.poolp.org

accept tagged DKIM for any relay source <sources> hostnames <helonames>
accept for any relay via smtp://127.0.0.1:10027
```

First of all, we need to identify what are the unique actions within that ruleset,
and this boils down to "aliases/virtual/userbase and anything after `relay`, `deliver to`":
```
alias <aliases> deliver to maildir
alias <opensmtpd-aliases> deliver to maildir
virtual <vusers> deliver to maildir
relay backup mx1.poolp.org
relay source <sources> hostnames <helonames>
relay via smtp://127.0.0.1:10027
```

The new grammar for these will result in:
```
action act01 maildir alias <aliases>
action act02 maildir alias <opensmtpd-aliases>
action act03 maildir virtual <vusers>
action act04 relay backup mx mx1.poolp.org
action act05 relay src <sources> helo-names <helonames>
action act06 relay host smtp://127.0.0.1:10027
```

Now that all actions have been defined, we need to identify what are the matching patterns within the former ruleset,
rewrite them with new grammar and attach them to their respective action.
The change is very mechanical but keywords were shortened and made less ambiguous:
```
match from any mail-from <shithole> for any reject

match for local action act01
match from any for domain <pdomains> action act01
match from any for domain opensmtpd.org action act02
match from any for domain <vdomains> action act03
match from any for domain <bdomains> action act04
match tag DKIM for any action act05
match for any action act06
```

There is exactly as many `match` rules as there used to be `accept`+`reject` rules,
they are in the same order,
they perform the same action,
they are just expressed differently.

Putting it all together
--
This is the resulting smtpd.conf for poolp.org which should be considerably more complex thant most setups,
we use a large part of OpenSMTPD's feature across all domains,
most setups will have two or three actions at most:
```
pki mx1.poolp.org cert "/etc/ssl/poolp.org.fullchain.pem"
pki mx1.poolp.org key "/etc/ssl/private/poolp.org.key"

pki mail.poolp.org cert "/etc/ssl/poolp.org.fullchain.pem"
pki mail.poolp.org key "/etc/ssl/private/poolp.org.key"

table sources           { 212.83.181.8 }
table helonames         { 212.83.181.8 = mx1.poolp.org }
table aliases           file:/etc/mail/aliases
table opensmtpd-aliases file:/etc/mail/aliases-opensmtpd.org
table pdomains          file:/etc/mail/primary-domains
table vdomains          file:/etc/mail/virtual-domains
table vusers            file:/etc/mail/virtual-users
table bdomains          file:/etc/mail/backup-domains
table shithole		{ "@qq.com" }

listen on lo0
listen on lo0 port 10028 tag DKIM
listen on egress tls-require pki mx1.poolp.org hostnames { 212.83.181.7 = mail.poolp.org, 212.83.181.8 = mx1.poolp.org }
listen on egress smtps pki mail.poolp.org auth hostname mail.poolp.org
listen on egress port submission tls-require pki mail.poolp.org auth hostname mail.poolp.org

action act01 maildir alias <aliases>
action act02 maildir alias <opensmtpd-aliases>
action act03 maildir virtual <vusers>
action act04 relay backup mx mx1.poolp.org
action act05 relay src <sources> helo-src <helonames>
action act06 relay host smtp://127.0.0.1:10027

match from any mail-from <shithole> for any reject
match for local action act01
match from any for domain <pdomains> action act01
match from any for domain opensmtpd.org action act02
match from any for domain <vdomains> action act03
match from any for domain <bdomains> action act04
match tag DKIM for any action act05
match for any action act06
```

Easing the conversion
--
The [smtpd.conf.5](https://man.openbsd.org/smtpd.conf) man page has been adapted but given that it is essentially a rewrite,
it is possible that we missed stuff and we will do our best to bring them to the usual quality you expect from OpenBSD man pages.

You can find [here](https://poolp.org/~gilles/smtpd.conf) a sample smtpd.conf that lists all directives the configuration parser supports.
The file is not exhaustive as this is not doable given the flexibility of the pattern matching,
but if you have a doubt on how to rewrite an old rule to a new rule this along the man page should be enough to help you.

The #OpenSMTPD channel on irc.freenode.net,
as well as our mailing lists are also available to help you do the conversion.


The queue is not backward compatible
--
Just switching from old config to new config and restarting is not going to work.

While the grammar change is very mechanical and seems to be very close to previous grammar,
the underlying structures have drastically changed,
the action resolving too and the result is that on-disk envelopes are not compatible and CAN'T be made compatible.
There's no way we can convert them because there's runtime resolving logic that expects to find an information we can't make up out of nowhere.

So there are two conversion paths.

The first one, the prefered of course, is to flush the mail queue before upgrading to new smtpd.
You pause incoming sessions with `smtpctl pause smtp` so your primary MX stops accepting mail,
which is never an issue because you have a secondary MX of course,
and you issue a `smtpctl schedule all` so all pending mails are sent right away.
When the mail queue is empty, which you can check with `smtpctl show queue`, you stop the daemon, upgrade and start the new one on your new config.

The second one is for busy mail hosts that can't flush a mail queue.
This is basically the case for a mailing list server which can almost never flush a queue empty due to unreachable hosts,
remote host limits requiring the server to be down for an extended period of time, etc...
For these, we came up with the `smtpd-salvage` daemon.

Basically, OpenSMTPD has envelopes versionning so a new config OpenSMTPD will not process old envelopes it can't support,
and an old config OpenSMTPD will not process the new envelopes it can't support.
If you can't flush the queue,
start the new config OpenSMTPD which will skip old envelopes and start creating new envelopes,
then start `smtpd-salvage` with the older config.
The `smtpd-salvage` daemon will then drain the queue from old messages.
After a few days, all messages that weren't delivered should be expired so `smtpd-salvage` can be stopped and deleted forever.

USE queue flush if you can.


--- 
Comments: [https://github.com/poolpOrg/poolp.org/issues/49](https://github.com/poolpOrg/poolp.org/issues/49)
