---
title: "OpenSMTPD now supports regex in match rules"
date: 2018-12-21 22:36:00
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
    regex table lookups were introduced for builtin filters.
    After a few weeks of working solely on filters, I wanted to work on something else.
    Using the same mechanism, all match criterias using tables can support regex.
{{< /tldr >}}


K_REGEX lookups
--
The table mechanism is used within OpenSMTPD to perform all kinds of lookups.

Recently,
while working on builtin filters,
I introduced the K_REGEX lookup type allowing tables to serve `regex(3)` patterns.

When a K_REGEX lookup is performed,
the table API will use the lookup key as a string and iterate over entries in the table,
compiling them with `regcomp(3)` and performing a simple `regcomp(3)` against the key.


regex use in match criterias
--
In the new smtpd.conf,
`accept` rules where replaced with `match` rules.

The `match` rules consist of matching criterias which describe how an envelope should look like to be accepted for a specific action.
Among these criterias,
you'll find for example `from src`, `for domain`, `mail-from` or even `rcpt-to`,
which can either take a table, or a string literal which ... is actually an inlined on-element table.

The new feature introduces a `regex` keyword that can be used to instruct OpenSMTPD that the table will use regex.
This basically allows the following:

```
table regex_domains file:/etc/mail/list-of-regex.txt"

match for domain regex "^.*\.example.com$" action foobar
match for domain regex <regex_domains> action foobar
```

except that it also allows it for pretty much every criteria that can be used on a match rule and expect a parameter.


What next ?
--
This was just committed a few minutes ago in -current and the feature is essentially complete,
so there's not much in that front.

You can already play with that feature which has been documented in the `smtpd.conf(5)` man page.

So ... NOW, I'm taking a break until next week because I did everything I wanted to do for this year ;-)

