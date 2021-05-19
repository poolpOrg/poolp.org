---
title: "OpenSMTPD: fully virtual setups, updated DNS & MTA code, SQLite support"
date: 2012-12-07 21:17:57
category: OpenSMTPD
authors:
 - Gilles Chehade
---

OHAI,

This week I had intended to work on filters. After 3 days of pain and swearing, even though it did move forward, I finally decided to step back for a few days and work on something else to preserve my sanity.

As you may have guessed, this week has not been too productive... Oh wait, yes it has ! It's actually been incredibly productive :-)

Just as usual, I will only write a summary for this week, the complete changelog can be retrieved from our mirror on Github.

K_SOURCEADDR table service

OpenSMTPD has been taught how to fetch a source address from a table, but does not make use of it yet. This will, for example, allow users to force a source address for their outgoing mail when they get blacklisted by the monkeys at spamhaus.

The K_SOURCEADDR service performs a cyclic lookup returning each address of a table one after the other so that a table holding multiple addresses will cycle through them.

Improved DNS API

Eric has cleaned and improved the DNS API: dnsquery*() functions now have a more logical ordering of their arguments; struct dns has been killed and we now use two small imsg-specific structures; dns_query_mx_preference() has been introduced to retrieve the preference level of a specific MX for a domain; and finally all MX addresses are looked up in parallel instead of sequentially.

MTA improvements

Eric has also improved the MTA internal operations so that it uses better abstractions for relay domains, hosts, sources and routes. A MTA session now operates on a given route and reports errors on that route only.

The relay tries to open as many routes as possible, within various limits, and drains all mails dispatching them. Oh and it is ready to use the K_SOURCEADDR lookup service but doesn't do it yet, this will probably be part of next week's milestone.

Table API improvements and new SQLite backend

The table API now provides a simple mechanism for backends to support a configuration file without having to deal with the parsing. The backend can be declared in smtpd.conf using:

table foobar mybackend:/etc/mail/mybackend.conf

Then the backend may simply do:

static int table_mybackend_config(struct table *t, const char *configfile) { void *cf;

 cf = table_config_create();
if (! table_config_parse(cf, configfile, T_HASH)) {
    table_config_destroy(cf);
    return 0;
}

table_set_configuration(t, cfg);
return 1;
}

To have the /etc/mail/mybackend.conf file parsed into a key/value table. It can then fetch the values from any other handler using:

static void * table_mybackend_open(struct table *t) { void *cf = table_get_configuration(t);

 if (table_config_get(cf, "key") == NULL) {
    log_warnx("table_mybackend: open: missing key from config file");
    return NULL;
}

return t;
}

Since I needed a use case, I added support for SQLite as a table backend allowing the use of SQLite for any kind of lookup. I had already done it in the past as a proof of concept when we were still using the map API for lookups, but this time it's the real deal.

SQLite support is achieved using the same approach as that of Postfix where the schema is not imposed but rather the user provides the queries themselves to allow as much flexibiliy as possible.

To show you how it can be setup, here's a sample smtpd.conf:


smtpd.conf

table mytbl sqlite:/etc/mail/sqlite.conf

i could have another one configured differently

table mytbl2 sqlite:/etc/mail/sqlite-other.conf

and i can have the same one serve different kinds of lookups ;-)

accept for domain alias deliver to mbox

and here's the sample sqlite.conf that goes with it


Path to database

dbpath /tmp/sqlite.db

Alias lookup query

rows >= 0

fields == 1 (user varchar)

query_alias select value from aliases where key=?;

Domain lookup query

rows == 1

fields == 1 (domain varchar)

query_domain select value from domains where key=?;

Of course, you may have multiple smtpd tables using sqlite backends, they may use different configuration files, you can have two aliases databases for two different domains or use the same database table to hold all information for all lookups. It's as flexible as it gets ... ALL lookup services are supported by the SQLite backend so it can be used to store anything used by OpenSMTPD.

K_USERINFO lookup service

OpenSMTPD uses the table API for every lookups but there was still one kind of lookups that were performed using a different API: users informations.

The table API expects lookups to be done asynchronously but OpenSMTPD had some code that looked up users synchronously, like right before a delivery or to find the home directory of a user for a ~/.forward check.

I introduced a new lookup service, K_USERINFO, which allows processes to lookup for informations regarding a username, such as its uid, gid and home directory. I then reworked the ~/.forward check and the delivery code to ensure the user information lookup is performed asynchronously through the K_USERINFO service rather than through the user_lookup() API which was synchronous.

The only backend to implement K_USERINFO was table_getpwnam which was essentially doing the same as before, just asynchronously, and at that point, user_lookup() bit the dust.

REALLY VIRTUAL users

A feature that has been requested for a long time and which was very heard to implement was support for virtual users.

OpenSMTPD required that the end user be a real system-user that could be looked up using getpwnam(). The K_USERINFO lookup service changed this slightly by having OpenSMTPD require that the end user be a user that could be looked up using table_lookup().

And since we can write lookup services using any backend, I wrote K_USERINFO handlers for table_static, table_db and table_sqlite. I then added a new keyword to smtpd.conf to allow rules to specify a user table:

table bleh1 { vuser => vuser:10:100:/tmp/vuser } table bleh2 { vuser => vuser:20:200:/tmp2/vuser }

accept for domain poolp.org users deliver to maildir accept for domain opensmtpd.org users deliver to maildir accept for domain pool.ps deliver to maildir

With this, OpenSMTPD will accept mail for domain poolp.org but will only find users if they are part of the table bleh1. The domain opensmtpd.org has a different users database, that shares a username but does not share the uid, gid and homedir. In this example, I used static tables, but it could really be sqlite, db or whatever ;-)

The domain pool.ps has no users table and defaults to the system database which is what most users will expect.

Relay URL update and K_CREDENTIALS lookup service

Since several months, smtpd.conf supports a "relay URL" format to define relays we want to route via:

table creds { mail.poolp.org => gilles:mypasswd } accept for any relay via tls+auth://mail.poolp.org:31337 auth

When sending mail, the creds table will search for an entry matching the domain name of the relay and find the credentials there. This has annoyed me for a while because it meant that it was not possible to share credentials between multiple relays, it was not possible to have different credentials for two relays operating under the same name, etc, etc ...

Also, it annoyed me that outgoing authentication would use K_CREDENTIALS while incoming authentication would not use the table API but the auth_backend API instead.

I convinced Eric that it would be nice to provide a new mechanism in relay URL so that we could have a label, like tls+auth://label@mail.poolp.org:31337 and it would be used as the key for the credentials lookup.

This would allow multiple relays to refer to the same label, or different relays under the same hostname to refer to different labels. It would also allow two nice tricks: first, since the labels are looked up in a different service then we can update the creds table live and MTA will pick up the change; then, if for incoming authentication we assume the username to be the label, then K_CREDENTIALS can be use as THE mechanism to authenticate both in and out sessions and we can kill the auth_backend API.

So ... I wrote it and we can now do:

table in_auth { gilles => gilles:encryptedpasswd } table out_auth { bleh => gilles:cleartextpasswd }

listen on all tls auth

accept for domain poolp.org deliver to maildir accept for any relay via tls+auth://bleh@mail.poolp.org auth

And this closes the last issue with regard to assuming any locality of the users for any purpose. An OpenSMTPD instance no longer assumes users to be local, or to really exists, for any purpose whatsoever.

Next week should see a long awaited feature, I will not say much and leave it as a surprise. Meanwhile, you guys have a nice week-end, I need a zZzZ ;-)
