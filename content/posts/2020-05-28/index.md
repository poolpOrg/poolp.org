---
title: "May 2020: OpenSMTPD 6.7.1p1 release, table-procexec and many PoCs"
date: 2020-05-28 01:57:00 +0200
authors:
 - "gilles"
categories:
 - technology
---

{{< tldr >}}
Worked on the <mark>OpenSMTPD 6.7</mark> release;
Did a lot of work on <mark>the new table API</mark>;
Wrote several PoCs;
{{< /tldr >}}

---
<font color="red"><b>WARNING:</b></font>

Examples of **code** and **configuration** that appear in this article are
here to help illustrate and explain **development stages** of my work.

They are subject to **changes** and **must not** be considered as user documentation.
By the time you're reading this, they will likely no longer work or reflect reality.

---

# OpenSMTPD 6.7.1p1 release

On May 19, 2020 was released [OpenBSD 6.7](https://www.openbsd.org/67.html) which ships OpenSMTPD 6.7,
the latest stable version of OpenSMTPD. 

<center>
	<img src="https://www.openbsd.org/images/puffy67.gif" />
</center>

I released the portable version of OpenSMTPD on the same day,
followed a few days later by a minor update to fix a packaging issue spotted by Debian maintainer [Ryan Kavanagh](https://github.com/ryanakca) and a possible crash when relaying over IPv6 spotted post-release by Void Linux maintainer [Leah Neukirchen](https://github.com/leahneukirchen).

I did **a lot of work on portable**,
extending the CI to new targets and such,
I won't talk about this work because this article contains a fair amount of more interesting topics.


## Some highlights in the 6.7.1p1 release

Before all,
one of the most visible change for package maintainers is that `libasr` is no longer a dependency.
It is shipped in the compat layer and built conditionally if the host system doesn't have the library installed,
this will solve a few headaches.

This release brings various improvements to the filtering protocol,
not visible to users for the most part,
but most notably three features deserve a special mention.

First, the new `bypass` action which was
[already discussed here](/posts/2019-12-24/december-2019-opensmtpd-and-filters-work-articles-and-goodies/).
It allows writing exclusion rules for certain sessions in builtin filters whereas,
prior to that,
all sessions connecting to a listener with filters would always have all filters applied to them:

```
filter trusted phase mail-from match src <trusted_sources> bypass
filter no_rdns phase mail-from match !rdns reject "550 go away"
filter no_fcrdns phase mail-from match !fcrdns  "550 go away"

listen on all filter { trusted, no_rdns, no_fcrdns }
```

Then, the `smtp-out` reporting stream which is very similar to `smtp-in` but reports SMTP events for outgoing sessions.
This allows writing filters that do all kind of accounting on outgoing trafic and,
as will be shown in this article,
can be used to make interesting filters.

Finally, all built-in filters can now match authenticated sessions and specific authenticated users.
It becomes possible to apply some filters to non-authenticated sessions only,
or to avoid applying some filters to specific authenticated users.

```
filter auth_only phase mail-from match !auth reject "550 you must be authenticated to send mail"
filter gilles_only phase mail-from match !auth gilles "550 only gilles is allowed to send mail"
```

And because `auth` takes a table parameter,
we can also do table-based filter matching using a credentials table:

```
table poolp_users { gilles = XXX, eric = XXX }
filter poolp_only phase mail-from match !auth <poolp_users> reject "550 must be a poolp.org user"
```

Outside of the filtering area,
other improvements were made to `smtpd.conf` that makes it possible to simplify existing configurations as well as express configurations that were not possible before.

There are two notable improvements in my opinion:

In previous releases, it was possible to have rules match authenticated session as follows:

```
match from any auth for any action "relay"
```

but this was an all or nothing mechanism,
either you wanted to match all authenticated sessions or none.

The mechanism was extended to support matching specific authenticated users so that you can write:

```
match from any auth gilles for any action "relay_gilles"
match from any auth eric for any action "relay_eric"
```

Similarly to filters,
the `auth` parameter is a table so that it is possible to pass a list of specific users and create rules targeting specific users:

```
match from any auth { gilles, eric } for any action "relay_custom"
match from any auth for any action "relay"
```

This is very powerful as `auth` accepts credentials tables and allows setups serving multiple domains to have rules match their specific users,
something that was not doable previously:

```
table users_poolp_org file:/etc/mail/users_poolp_org
table users_opensmtpd_org file:/etc/mail/users_opensmtpd_org

listen on $ip_poolp [...] auth <users_poolp_org>
listen on $ip_opensmtpd [...] auth <users_opensmtpd_org>

[...]

match from any auth <users_poolp_org> for any action "relay_poolp"
match from any auth <users_opensmtpd_org> for any action "relay_opensmtpd"
```

Finally,
a rework of the `from` and `for` logic in `smtpd.conf` was done to better express rules.
Instead of assuming that `from` and `for` always attempted to match a source address and a destination,
they now match the user intent:

```
match from auth [...]
match from mail-from gilles@poolp.org [...]
match [...] for rcpt-to gilles@poolp.org
```

This change allows expressing many configurations in a much more compact and user-friendly way:

For example, the following:
```
match from any auth for domain poolp.org rcpt-to gilles@poolp.org action "deliver"
```

becomes:
```
match from auth for rcpt-to gilles@poolp.org action "deliver"
```

They are both functionally equivalent but the change is semantic,
the first rule requires that the envelope matches both `from any` and `auth` for origin and both `for domain poolp.org` and `rcpt-to gilles@poolp.org` for destination,
whereas in the second rule it only requires that `auth` and `rcpt-to gilles@poolp.org` be matched regardless of the source address and destination domain.
This semantic shift is backwards compatible so existing configuration are not impacted,
they just have some rules that could be simplified in some cases.

Most of the other work is either not visible to users,
but feel free to [browse the commit history](https://github.com/OpenSMTPD/OpenSMTPD/commits/master) for more as there's been a LOT of work poured in the release polishing and cleaning stuff.



# filter-prometheus
As I mentionned above,
the new release comes with an `smtp-out` events reporting stream that makes it possible to write filters that are aware of outgoing trafic,
so I decided to write a PoC filter that would highlight how this can be used in a useful way.

[Prometheus](https://prometheus.io/) is an open-source monitoring system that's fairly popular and that seems to have the favours of highly competent people at ${DAYJOB}.
Since I need to get familiar with it at work,
I thought I'd get my hands on it on my spare time too.

**Basically** the idea is that you run an instance (or a cluster, but lets keep it simple) that will contact configured endpoints that expose various metrics.
The metrics are then available in Prometheus through a query language, PromQL, and through graphs in a web user interface.

I wrote a filter-prometheus which registers `smtp-in` and `smtp-out` events,
creates some metrics based on them,
and exposes these metrics on an HTTP endpoint.

<center>
	<img src="2020-05-28-filter-prometheus-metrics.png" />
</center>


Prometheus is configured to hit that endpoint and...
voila, that's all,
they are available through prometheus.

<center>
	<img src="2020-05-28-filter-prometheus.png" />
</center>

On the configuration standpoint,
this is straightforward.
The filter itself doesn't have configuration,
it only supports a command line flag to specify which IP and port are used to exposed the metrics.

Prometheus has a simple configuration:

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
    - targets: ["localhost:9090"]

  - job_name: "OpenSMTPD: in.mailbrix.mx"
    static_configs:
    - targets: ["in.mailbrix.mx:31333"]
```

OpenSMTPD has an even simpler configuration:

```
filter prometheus proc-exec "filter-prometheus -exporter 0.0.0.0:31333"

# attach filter to a listener for smtp-in metrics
listen on all filter prometheus

# attach filter to a relay action for smtp-out metrics
action "outgoing" relay filter prometheus

[...]
```

The filter is already available in a
[Github repository](https://github.com/poolpOrg/filter-prometheus)
but note that this is a work in progress and that it was mostly done to play with prometheus and exporting metrics,
not intended to be used for real as I don't have any significant experience to design metrics properly at this point.

**If you are familiar with prometheus and want to give me a hand in making this filter production-ready,
I'll be more than happy to make this happen with your help.**


# table-procexec

OK,
let's get to the hairy part now,
but first a bit of a refresher.

OpenSMTPD uses a mechanism called `tables` for all kinds of internal lookups.
The table API support **three operations**,
`check`, `lookup` and `fetch`.
It uses these three operations to check if an envelope criteria is met (ie: does this user exists ?),
lookup informations based on a specific key (ie: what are the aliases for root ?),
or fetch a value from a set (ie: what's a valid source IP to use for this outgoing session ?).

In addition to this,
the table API supports various **lookup services** (ie: `credentials`, `aliases`, `mailaddr`, ...)
which determine **what kind of data the table is supposed to return**.
If you lookup a `mailaddr` in a table,
the expected return value is an e-mail address that can be validated by the daemon.
The three table operations are aware of this,
so OpenSMTPD will for example "check if table has a mailaddr matching key `gilles@poolp.org`".


There are **multiple table backends available**,
both builtin to OpenSMTPD (memory, file and db) and third-party (sqlite, postgres, mysql, ...).
Some backends,
like table-passwd,
only support **specific** lookup services (ie: credentials, userinfo) while others support all.
As long as you have a data source upon which you can implement the three operations,
you can implement a backend for some or all of the lookup services.
The good part is that this happens outside of the daemon,
so there's **no need for cooperation from OpenSMTPD**,
no spill of dependencies and such,
table backends are **fully independant standalone programs**.



The API is very simple so in theory anyone can write a backend for their own custom use-cases,
but in practice the communication with OpenSMTPD happens through the `imsg` API which limits the interface to C developers.
Eric and I [wrote the necessary bits a long time ago to abstract the details](https://github.com/OpenSMTPD/OpenSMTPD-extras/blob/master/api/table_api.c) so that C developers [only need to focus on implementing the operations](https://github.com/OpenSMTPD/OpenSMTPD-extras/blob/master/extras/tables/table-stub/table_stub.c) and not deal with the IPC.
It helped a lot but any backend doing something a bit tricky,
[ldap for example](https://github.com/OpenSMTPD/OpenSMTPD-extras/blob/master/extras/tables/table-ldap/table_ldap.c),
still requires being comfortable with C programming and a sysadmin without that knowledge can't really get anything done.
A bit sad given that sysadmins are the primary users of this...


I later [implemented a table-python backend](https://github.com/OpenSMTPD/OpenSMTPD-extras/blob/master/extras/tables/table-python/table_python.c) which acted as a **bridge**.
It allowed to implement the three operations in Python functions and have the backend call the functions,
taking care of converting back and forth between the two languages.
This made table-backends much more user-friendly but then came the Perl people, then the Lua people, then the Go people, ...
and a bridge would have to be written and maintained for all of them,
something I don't have the time or motivation to do.

We had already agreed with `eric@`,
for other reasons,
that **the API should be switched to a line-based protocol**,
like was done for filters,
but this is a very tricky task in the daemon at this point and requires solving non-trivial limitations first.
I'll skip on this because this article would grow considerably,
let's just say that my first two attempts at this were unsuccessful.


It still annoyed me that we couldn't move forward with this so I decided to tackle the issue differently.
I wrote [table-procexec](https://github.com/OpenSMTPD/OpenSMTPD-extras/tree/table-procexec/extras/tables/table-procexec)
a table backend which translate the `imsg` protocol into a filter-like query/response line-based protocol:

```
table|0.1|1590631921.2944137|table-name|check|deadbeefdeadbeef|mailaddr|gilles@poolp.org
table-response|deadbeefdeadbeef|found
table|0.1|1590632034.2909038|table-name|lookup|deadbeefdeadbeef|alias|root
table-response|deadbeefdeadbeef|found|gilles
```

The `table-procexec` backend communicates with OpenSMTPD through `imsg`,
forks a child table backend **talking the new protocol** and takes care of proxying queries and responses **translating** imsg to this new protocol.

```
OpenSMTPD <-- imsg --> table-procexec <-- line-protocol --> table-foobar
```

The child table backend can be any program written in any language,
just like with filters,
and all it has to do is to implement the three operations expected from a backend,
read the queries from stdin and respond to stdout.

This isn't as elegant as if OpenSMTPD was producing the line-based protocol itself,
allowing us to skip the `table-procexec` layer,
but it allows moving forward in a way that's **transparent to the daemon**.
We can already test the API, improve it, make sure it works, implement new table backends on top of it and use them.
When the limitations in OpenSMTPD are solved,
the `table-procexec` can be thrown away transparently:

```
OpenSMTPD <-- line-protocol --> table-foobar
```

This is a work in progress and while it can be plugged to OpenSMTPD today using nothing but the existing framework,
it doesn't allow passing a configuration file to the child backend (yet).

In terms of configuration,
it works similarly to any existing backend:
```
# table foobar uses procexec backend to fork table-foobar
table foobar procexec:table-foobar
```

Ideally, we should switch to a filter like syntax, but we're not there yet:
```
table foobar proc-exec "table-foobar -f /etc/mail/foobar.conf"
```

Testing this require building the `table-procexec` backend from the branch of the same name in the OpenSMTPD-extras repository,
I'll leave this as an exercise to the reader.




# go-opensmtpd
Of course,
testing the new table API required a PoC so I decided to start with a Go implementation.

The `go-opensmtpd/table` package provides a work-in-progress implementation of the protocol parser,
as well as a table API to ease development.
Writing a table backend in Go using this package is simple,
here's a simple dummy example of a table that only contains my e-mail address:

```go
package main

import (
	"github.com/poolpOrg/go-opensmtpd/table"
)

func check(token string, service table.LookupService, key string) {
	if key == "gilles@poolp.org" {
		// found in table
		table.Boolean(token, true)
	} else {
		// not found
		table.Boolean(token, false)
	}
}

func lookup(token string, service table.LookupService, key string) {
	if service == "alias" && key == "root" {
		// if looking up alias for root, return my address
		table.Result(token, "gilles@poolp.org")
	} else {
		// empty result, not-found
		table.Result(token)
	}
}

func fetch(token string, service table.LookupService) {
	// only my address is in the set
	table.Result(token, "gilles@poolp.org")
}

func main() {
	table.OnCheck(check)
	table.OnLookup(lookup)
	table.OnFetch(fetch)
	table.Dispatch()
}
```

Note that the Dispatch() function never returns as it is the event loop for the protocol parser.
I don't have much to add here, this is trivial to understand.

The package is a work in progress,
it is already [available in a Github repository](https://github.com/poolpOrg/go-opensmtpd) but expect it to change every few days at this point,
don't use it unless you are ready to suffer.


# py-opensmtpd
The goal not being to have a Go centric API,
testing with a different language required another PoC so I decided to go with Python this time.

The `py-opensmtpd/table` module also provides a work-in-progress implementation of the protocol parser,
as well as a table API to ease development.
Writing a table backend in Python using this module is no more complex than in Go,
here's the exact same dummy example of a table that only contains my e-mail address:

```python
from opensmtpd import table

def check(token, tableName, service, key):
	if key == "gilles@poolp.org":
		return table.boolean(token, true)
	return table.boolean(token, false)

def lookup(token, tableName, service, key):
	if service == "alias" and key == "root":
		# if looking up alias for root, return my address
		return table.result(token, "gilles@poolp.org")
	return table.result(token)

def fetch(token, tableName, service):
	# only my address is in the set
	table.result(token, "gilles@poolp.org")

def main():
	table.on_check(check)
	table.on_lookup(lookup)
	table.on_fetch(fetch)
	table.dispatch()

if __name__ == "__main__":
	main()
```

Again, the dispatch() function never returns as it is the event loop for the protocol parser.
The interface is the exact same as what I did with `go-opensmtpd`.

The package is also a work in progress,
it is also already [available in a Github repository](https://github.com/poolpOrg/py-opensmtpd) but expect it to change every few days,
don't use it unless you are also ready to suffer.

# py-consul
And because these PoC were dummy and do not show how the API can be used for real cases,
I wrote a PoC for a table-backend that would actually **do something more useful**.

[Consul](https://www.consul.io/) is a service to *"automate network configurations, discover services, and enable secure connectivity across any cloud or runtime"*.
It is another tool highly valued at work and it conveniently comes with a key-value store hidden behind a REST API,
something that we can work with to implement the three table backend operations.

I implemented a `table-consul` python backend using the key-value store to perform its lookups.
**The code is a work in progress, not meant to be used in production, but it highlights how simple it is to integrate with existing tools**.
It took about 20 minutes to write the backend from scratch with no prior experience in Consul.
The backend itself is ~70 lines including the license and empty lines.
Using `table-consul`,
one can setup a consul instance and use the key-value store to hold table informations used in OpenSMTPD lookups.

In the following example,
I will setup and OpenSMTPD to use `table-consul` for aliases lookups:

```
table consul procexec:table-consul

listen on all

action "local_users" maildir alias <consul>

[...]
```

I then start a brand new empty consul instance and send a mail to root from my OpenSMTPD:

<center>
	<img src="2020-05-28-empty-consul.png" />
</center>

<center>
	<img src="2020-05-28-consul-mail-root-noalias.png" />
</center>


Then,
after adding a key `root` with value `gilles` in the `foobar/alias` bucket and sending a mail again,
without restarting the daemon:

<center>
	<img src="2020-05-28-consul-fed.png" />
</center>

<center>
	<img src="2020-05-28-consul-mail-root-alias.png" />
</center>

The alias lookup went to `table-consul` through `table-procexec`,
`table-consul` did an HTTP request to consul to retrieve the alias,
passed it back to an unsuspecting OpenSMTPD just as if it came from a regular table.


Because `table-procexec` can't pass configuration to child backend yet,
I decided to use `<table>/<service>` as the bucket for key-values which is not always very
practical as it doesn't allow sharing same keys between multiple service lookups but this was good enough for a PoC.

Like for `filter-prometheus`,
the table is already available in a
[Github repository](https://github.com/poolpOrg/py-table-consul),
but **this is a work in progress** and was mostly to play with the new API,
**not intended to be used for real** as I don't have any significant experience with consul to make sound choices.

**If you are familiar with consul and want to give me a hand in making this table production-ready,
I'll be more than happy to make this happen with your help.**


# What's next ?

I started this article at ~2:00AM and spent four hours writing,
I can't even remember my name at this point so I have no idea what I'll prioritize next month,
I just need some sleep.

