---
title: "Out of my cave, lots of updates"
date: 2024-06-11 21:00:00 +0100
authors:
 - "gilles"
categories:
 - technology
tags:
 - OpenBSD
 - OpenSMTPD
 - plakar
 - backups
---

{{< tldr >}}
I have been silent for a while due to personal matters but I did a ton of stuff on OpenSMTPD, plakar and a handful of other projects.
{{< /tldr >}}


# Why the silence ?
My last post was a bit unusual,
far off from the usual light tone,
and followed by months of silence despite my habit of writing almost monthly.

Quite frankly, I don't want to expand on why so I'll keep it short.

I had a very, very, very rough year and a half on the personal plan.
I did a ton of things to keep my mind busy,
I had a ton of things to write about,
but I just couldn't be fucked to sit down in front of an empty page and pour the contents of my brain for these last few months.

**I'm back on track now.**

As I haven't written anything in a long time,
I have content for multiple articles...
but I know that I won't find time to write several of them.
I'd rather braindump a ton of things in this article,
even if it is a bit dense,
so I can log what I did and renew with writing.

Feel free to use the table of contents to skip over topics.


# OpenBSD and OpenSMTPD

## Back in the game
You may have noticed that **I reappeared on the OpenBSD and OpenSMTPD mailing lists**,
and I even committed a handful of things in both projects.
I will not be as active as I used to,
but won't be as inactive as these last few years,
I'll figure a balance between contributing there and working on my own personal projects.



## OpenSMTPD security improvements
Early March,
someone posted a question on the misc@opensmtpd.org mailing-list.

I had not been active on the list for a while and didn't pay much attention as members of the community tends to answer most questions now.
In this case,
however,
the behavior contradicted what I recalled writing a few years ago and immediately struck me as being buggy.

For some reason,
delivery through LMTP had been changed to use the daemon user rather than the end-user account which was not the finest idea as I knew this meant a local DoS was possible for any LMTP-enabled setup (not the default).

I notified OpenBSD and provided a diff to revert the behavior back to what it was and since I was looking at that code again, I suggested two additional improvements which were accepted and commited.

**The first improvement was to push for disallowing the evaluation of `.forward `files for users that do not match the destination user in an action.**

Basically, if the delivery user is `gilles` then if a `.forward` file is found that's not owned by `gilles`, it should be skipped even if part of the aliases expansion. In most cases, you don't run into this kind of situation, but occasionally, if you have setups with alternate delivery users, like:

    action foobar lmtp ... user _foobar

... then mailing `gilles@` would look at `gilles`' .forward file if found, which is not desired as only `_foobar` should be looked at. This probably qualifies more as a bug than an "improvement" to be honest but noone ever complained, I just happened to have a good rationale for why it's a bad idea: this would have prevented the DoS that was introduced with the LMTP behavior change, making OpenSMTPD a tiny bit more robust.

**The second improvement was to push for the killing of commands in `root`'s `.forward` file.**

The root account should really not be allowed to receive mail and should be aliased but it is hard to achieve because it is the only account you know exists out of the box. OpenSMTPD disallowed the execution of any command as `root` except for the hardcoded constant string `mail.local`, but I realised that there was a way to bypass this: `.forward` files.

The `.forward` files allow plugging custom commands to be ran as the delivery user, so `root` could technically plug any command there that would be ran as `root`. This also meant that if a hacker managed to alter the `.forward` file with arbitrary content OR to corrupt memory so that the `.forward` file parser got fed with arbitrary content, they could get a command executed as `root` and bypass the `mail.local` restriction.

I made an argument that there's no reason `root` should be allowed to execute any command in OpenSMTPD so we should kill the ability for root to put anything but e-mail addresses or local accounts in `.forward` files and this got committed, increasing considerably the security of the daemon by locking `root` to `mail.local`.

We verified by injecting envelopes that attempted to execute code as `root` through custom MDA but also by instrumenting code to "patch" the delivery command as if a memory corruption had rewritten it.

**Security improved by a great bunch with these minor changes.**


## OpenSMTPD 7.5.0p0 released
Omar Polo (op@) did a great job **reviving OpenSMTPD-portable**,
bringing all OpenBSD changes to it and testing on various systems.

Seeing him take care of this,
which is something I _had_ to do by myself in the past,
was very pleasing.
He took care of everything by himself,
I offered minor guidance and help with understanding prior technical decisions,
but he single-handedly took care of bringing back OpenSMTPD-portable.

**All I had to do was...
proof-read him and sign his tarballs**.
That was lovely :-)

OpenSMTPD 7.5.0 was [announced](https://www.mail-archive.com/misc@opensmtpd.org/msg06238.html) the 10th of April,
I'll let you read the changelog.


## New table API protocol
Since I was reading Omar's work,
I rejoined the #OpenSMTPD IRC channel,
and caught a discussion about table_ldap.

For history,
**I wrote table_ldap as a proof of concept** that OpenSMTPD could provide LDAP support through OpenBSD's aldap API...
but I could not care less about LDAP: **I don't use LDAP**.
I wrote the most basic working implementation I could in hope someone with actual need would care for it,
but noone ever did.

A user was starting to improve on it but the table was part of a bigger repository with a ton of outdated code which I considered deprecated and unmaintained.

I explained that we had in mind to **convert the table protocol into one similar to the filter protocol**, switching from the `imsg(3)` API to an stdio-based API that would allow writing table backends very easily in any language by splitting lines coming to stdin and providing responses to stdout. The model has proved to be very nice for filters and tables are considerably simpler.

I had a **3-steps plans** for it:

- implement a table_stdio that would convert the imsg protocol to new stdio protocol
- convert tables from imsg to new protocol, considering that there was already an API abstracting it, this was just changing the plumbing to convert binary packets to string packets
- have a chat with the OpenBSD crowd to convince them that first step could be done in the daemon to skip the need for a custom table proxying old protocol to new protocol.

Omar bought the idea and implemented... the three steps.

I sent a mail to explain the rationale to OpenBSD hackers,
there was no opposition so we moved forward:
Omar created ports for the table backends we supported,
and committed both the ports and the protocol change in OpenSMTPD.

I [sent a mail with more details](https://www.mail-archive.com/misc@opensmtpd.org/msg06266.html) to the misc@opensmtpd.org list to explain.


## New K_AUTH table lookup service
Authentication in OpenSMTPD is done in two ways.

The first one is the system authentication,
which uses the `bsd_auth(3)` framework on OpenBSD,
the `pam` framework on other systems where it's available, 
and falls back to `crypt(3)`-ed validation of `getpwnam(3)`'s `pw->pw_passwd` field.

The second one is the table authentication,
where credentials are not matched against the system but against a table backend,
like for instance if you decided to store your users' passwords in a database of some sort (sqlite, mysql, postgrsql, ldap, ... or even a file).

In that second case,
OpenSMTPD performs a `K_CREDENTIALS` lookup through the table API which will retrieve a `crypt(3)`-ed password from the table backend,
and it'll do the `crypt(3)` password validation itself so table backends do not have to implement the logic.

While this works and has the benefit of allowing any backend to propose `K_CREDENTIALS` lookup as long as they can store a key-value pair,
the problem is that some backends have more custom ways of authenticating users and we didn't take advantatge of them.

I suggested that we implement a new `K_AUTH` lookup services to **offload authentication to a table backend**.
If a table implements `K_AUTH`,
instead of being requested for credentials through `K_CREDENTIALS`,
OpenSMTPD will request a `K_AUTH` check of a username and password pair.
The table will then be responsible of confirming if they match or not without exposing the underlying logic,
allowing them to implement authentication in any suitable way.

The diff is very simple.
It declares a new K_AUTH lookup service.
It introduces it in the table protocol so tables can register the service.
Then when an authentication is requested,
it checks if the table supports `K_AUTH` and does the new lookup,
or it falls back to what it has done until now:

```diff
Index: lka.c
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/lka.c,v
diff -u -p -r1.248 lka.c
--- lka.c	20 Jan 2024 09:01:03 -0000	1.248
+++ lka.c	26 May 2024 20:56:02 -0000
@@ -720,6 +720,7 @@ static int
 lka_authenticate(const char *tablename, const char *user, const char *password)
 {
 	struct table		*table;
+	char	       		 offloadkey[LINE_MAX];
 	union lookup		 lk;
 
 	log_debug("debug: lka: authenticating for %s:%s", tablename, user);
@@ -730,7 +731,27 @@ lka_authenticate(const char *tablename, 
 		return (LKA_TEMPFAIL);
 	}
 
-	switch (table_lookup(table, K_CREDENTIALS, user, &lk)) {
+	/* table backend supports authentication offloading */
+	if (table_check_service(table, K_AUTH)) {
+		if (!bsnprintf(offloadkey, sizeof(offloadkey), "%s:%s",
+			user, password)) {
+			log_warnx("warn: key serialization failed for %s:%s",
+			    tablename, user);
+			return (LKA_TEMPFAIL);
+		}
+		switch (table_match(table, K_AUTH, offloadkey)) {
+		case -1:
+			log_warnx("warn: user credentials lookup fail for %s:%s",
+			    tablename, user);
+			return (LKA_TEMPFAIL);
+		case 0:
+			return (LKA_PERMFAIL);
+		default:
+			return (LKA_OK);
+		}
+	}
+
+	switch (table_lookup(table, K_CREDENTIALS, user, &lk)) {
 	case -1:
 		log_warnx("warn: user credentials lookup fail for %s:%s",
 		    tablename, user);
Index: smtpd-api.h
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/smtpd-api.h,v
diff -u -p -r1.36 smtpd-api.h
--- smtpd-api.h	23 Dec 2018 16:06:24 -0000	1.36
+++ smtpd-api.h	26 May 2024 20:56:03 -0000
@@ -135,8 +135,9 @@ enum table_service {
 	K_RELAYHOST	= 0x200,	/* returns struct relayhost	*/
 	K_STRING	= 0x400,
 	K_REGEX		= 0x800,
+	K_AUTH		= 0x1000,
 };
-#define K_ANY		  0xfff
+#define K_ANY		  0xffff
 
 enum {
 	PROC_TABLE_OK,
Index: table.c
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/table.c,v
diff -u -p -r1.52 table.c
--- table.c	7 May 2024 12:10:06 -0000	1.52
+++ table.c	26 May 2024 20:56:03 -0000
@@ -83,6 +83,7 @@ table_service_name(enum table_service s)
 	case K_RELAYHOST:	return "relayhost";
 	case K_STRING:		return "string";
 	case K_REGEX:		return "regex";
+	case K_AUTH:		return "auth";
 	}
 	return "???";
 }
@@ -116,6 +117,8 @@ table_service_from_name(const char *serv
 		return K_STRING;
 	if (!strcmp(service, "regex"))
 		return K_REGEX;
+	if (!strcmp(service, "auth"))
+		return K_AUTH;
 	return (-1);
 }
```


## Return of the kicker
There is a problem with OpenSMTPD which is that **clients can keep sessions alive indefinitely as long as they are active**,
hogging a connection unnecessarily.

In theory,
since the number of transactions allowed in a session is limited,
so you'd expect a disconnection to happen because a limit is hit...
but in practice the SMTP protocol allows clients to play various sequences that are legal RFC-wise and that maintain a session active without submitting messages.
For example,
a client could do a NOOP loop,
or it could do a RSET loop,
or it could do a NOOP-RSET loop,
or it could begin a transaction then RSET it without a commit,
or it could AUTH-RSET loop,
... you get the idea.

You can use firewall and rate-limiting to block misbehaving clients,
but this is a bit painful on the postmaster side,
and you have to tackle this differently on different systems with different firewall and rate-limiting mechanisms.

A loooooong time ago (about 12 years), eric@ and I introduced a mechanism called the kicker.

The philosophy was simple:

- a session is supposed to move forward with exchanging messages
- each SMTP command transitions the session from a state to another
- each transition can be considered as a backwards move, an idle move or a forward move in the session progression
- if we detect a pattern that implies the session is not progressing enough, we kick the offender out

The mechanism was backed out 8 years ago for what I consider to be bad reasons,
and since I was unhappy with the decision I didn't work on trying to bring it back.

Time has passed,
now [OpenSSH introduces options to penalize undesirable behavior](https://undeadly.org/cgi?action=article;sid=20240607042157),
so I made a suggestion to work on bringing it back which was well received.

I wrote two versions of the feature,
one that plugs right into the smtp session layer,
and one that works as a builtin filter.

I turns out that **plugging it as a builtin filter**,
which is intuitively the right place,
has several downsides and is **considerably more complex than slapping a handful of conditions in the smtp session layer**.

Anyways,
the idea is that the kicker is a feature you can now enable/disable globally or on specific listeners:
```conf
smtp kicker on

listen on all kicker off
```

It uses a kickcount counter which gets increased with every command and reset whenever a session is progressing.
If the kickcount reaches a certain threshold,
it means that too many commands have been supplied without making a move forward:
```term
$ nc mx-in.poolp.org 25
220 mx-in.poolp.org ESMTP OpenSMTPD
NOOP
250 2.0.0 Ok
NOOP
250 2.0.0 Ok
NOOP
250 2.0.0 Ok
NOOP
421 4.4.2 4.0.0 Other/Undefined: Session terminated due to lack of progress

$ nc mx-in.poolp.org 25
220 mx-in.poolp.org ESMTP OpenSMTPD
HELO localhost
250 mx-in.poolp.org Hello localhost [82.65.169.200], pleased to meet you
MAIL FROM:<gilles>
250 2.0.0 Ok
RSET
250 2.0.0 Reset state
MAIL FROM:<gilles>
250 2.0.0 Ok
RSET
250 2.0.0 Reset state
MAIL FROM:<gilles>
250 2.0.0 Ok
RSET
250 2.0.0 Reset state
MAIL FROM:<gilles>
250 2.0.0 Ok
RSET
421 4.4.2 4.0.0 Other/Undefined: Session terminated due to lack of progress
```




The diff is still a work in progress,
it hasn't even been proposed yet as I'm currently running it on my own mail server:
```diff
Index: parse.y
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/parse.y,v
diff -u -p -r1.299 parse.y
--- parse.y	19 Feb 2024 21:00:19 -0000	1.299
+++ parse.y	8 Jun 2024 23:23:19 -0000
@@ -98,8 +98,8 @@ static uint32_t		 last_dynchain_id = 1;
 
 enum listen_options {
 	LO_FAMILY	= 0x000001,
-	LO_PORT		= 0x000002,
-	LO_SSL		= 0x000004,
+	LO_PORT	       	= 0x000002,
+	LO_SSL 		= 0x000004,
 	LO_FILTER      	= 0x000008,
 	LO_PKI      	= 0x000010,
 	LO_AUTH      	= 0x000020,
@@ -107,12 +107,14 @@ enum listen_options {
 	LO_HOSTNAME   	= 0x000080,
 	LO_HOSTNAMES   	= 0x000100,
 	LO_MASKSOURCE  	= 0x000200,
-	LO_NODSN	= 0x000400,
-	LO_SENDERS	= 0x000800,
+	LO_NODSN		= 0x000400,
+	LO_SENDERS		= 0x000800,
 	LO_RECEIVEDAUTH = 0x001000,
 	LO_MASQUERADE	= 0x002000,
-	LO_CA		= 0x004000,
+	LO_CA			= 0x004000,
 	LO_PROXY       	= 0x008000,
+	LO_KICKER_ON	= 0x010000,
+	LO_KICKER_OFF	= 0x020000,
 };
 
 #define PKI_MAX	32
@@ -174,11 +176,11 @@ typedef struct {
 %token	HELO HELO_SRC HOST HOSTNAME HOSTNAMES
 %token	INCLUDE INET4 INET6
 %token	JUNK
-%token	KEY
+%token	KEY KICKER
 %token	LIMIT LISTEN LMTP LOCAL
 %token	MAIL_FROM MAILDIR MASK_SRC MASQUERADE MATCH MAX_MESSAGE_SIZE MAX_DEFERRED MBOX MDA MTA MX
 %token	NO_DSN NO_VERIFY NOOP
-%token	ON
+%token	OFF ON
 %token	PHASE PKI PORT PROC PROC_EXEC PROTOCOLS PROXY_V2
 %token	QUEUE QUIT
 %token	RCPT_TO RDNS RECIPIENT RECEIVEDAUTH REGEX RELAY REJECT REPORT REWRITE RSET
@@ -543,6 +545,12 @@ SMTP LIMIT limits_smtp
 	}
 	conf->sc_subaddressing_delim = $3;
 }
+| SMTP KICKER ON {
+	conf->sc_smtp_kicker = 1;
+}
+| SMTP KICKER OFF {
+	conf->sc_smtp_kicker = 0;
+}
 ;
 
 srs:
@@ -2490,6 +2498,22 @@ opt_if_listen : INET4 {
 			}
 			listen_opts.sendertable = t;
 		}
+		| KICKER ON {
+			if (listen_opts.options & (LO_KICKER_ON|LO_KICKER_OFF)) {
+				yyerror("kicker already specified");
+				YYERROR;
+			}
+			listen_opts.options |= LO_KICKER_ON;
+			listen_opts.flags |= F_KICKER;
+		}
+		| KICKER OFF {
+			if (listen_opts.options & (LO_KICKER_ON|LO_KICKER_OFF)) {
+				yyerror("kicker already specified");
+				YYERROR;
+			}
+			listen_opts.options |= LO_KICKER_OFF;
+			listen_opts.flags &= ~F_KICKER;
+		}
 		;
 
 listener_type	: socket_listener
@@ -2686,6 +2710,7 @@ lookup(char *s)
 		{ "inet6",		INET6 },
 		{ "junk",		JUNK },
 		{ "key",		KEY },
+		{ "kicker",		KICKER },
 		{ "limit",		LIMIT },
 		{ "listen",		LISTEN },
 		{ "lmtp",		LMTP },
@@ -2704,6 +2729,7 @@ lookup(char *s)
 		{ "no-dsn",		NO_DSN },
 		{ "no-verify",		NO_VERIFY },
 		{ "noop",		NOOP },
+		{ "off",		OFF },
 		{ "on",			ON },
 		{ "phase",		PHASE },
 		{ "pki",		PKI },
@@ -3239,6 +3265,7 @@ create_sock_listener(struct listen_opts 
 	l->ss.ss_len = sizeof(struct sockaddr *);
 	l->local = 1;
 	conf->sc_sock_listener = l;
+
 	config_listener(l, lo);
 }
 
@@ -3358,12 +3385,17 @@ config_listener(struct listener *h,  str
 			h->flags |= F_MASQUERADE;
 	}
 
+	if (lo->options & LO_KICKER_ON)
+		h->flags |= F_KICKER;
+	else if (conf->sc_smtp_kicker && !(lo->options & LO_KICKER_OFF))
+		h->flags |= F_KICKER;
+
 	if (lo->ssl & F_TLS_VERIFY)
 		h->flags |= F_TLS_VERIFY;
 
 	if (lo->ssl & F_STARTTLS_REQUIRE)
 		h->flags |= F_STARTTLS_REQUIRE;
-	
+
 	if (h != conf->sc_sock_listener)
 		TAILQ_INSERT_TAIL(conf->sc_listeners, h, entry);
 }
Index: smtp_session.c
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/smtp_session.c,v
diff -u -p -r1.442 smtp_session.c
--- smtp_session.c	20 Mar 2024 17:52:43 -0000	1.442
+++ smtp_session.c	8 Jun 2024 23:23:20 -0000
@@ -145,6 +145,9 @@ struct smtp_session {
 	const char		*filter_param;
 
 	uint8_t			 junk;
+
+#define MAX_KICKER_SCORE 3
+	uint8_t			 kicker_score;
 };
 
 #define ADVERTISE_TLS(s) \
@@ -180,6 +183,7 @@ static void smtp_command(struct smtp_ses
 static void smtp_rfc4954_auth_plain(struct smtp_session *, char *);
 static void smtp_rfc4954_auth_login(struct smtp_session *, char *);
 static void smtp_free(struct smtp_session *, const char *);
+static int smtp_kicker(struct smtp_session *);
 static const char *smtp_strstate(int);
 static void smtp_auth_failure_pause(struct smtp_session *);
 static void smtp_auth_failure_resume(int, short, void *);
@@ -1231,6 +1235,8 @@ smtp_command(struct smtp_session *s, cha
 	char			       *args;
 	int				cmd, i;
 
+	log_trace(TRACE_KICKER, "smtp: kicker progression score=%d", s->kicker_score);
+
 	log_trace(TRACE_SMTP, "smtp: %p: <<< %s", s, line);
 
 	/*
@@ -1276,6 +1282,10 @@ smtp_command(struct smtp_session *s, cha
 			break;
 		}
 
+	/*  do not increase kicker_score, they progress session */
+	if (cmd != CMD_RCPT_TO && cmd != CMD_MAIL_FROM)
+		s->kicker_score++;
+
 	s->last_cmd = cmd;
 	switch (cmd) {
 	/*
@@ -1742,6 +1752,10 @@ smtp_filter_phase(enum filter_phase phas
 static void
 smtp_proceed_rset(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+
 	smtp_reply(s, "250 %s Reset state",
 	    esc_code(ESC_STATUS_OK, ESC_OTHER_STATUS));
 
@@ -1755,6 +1769,11 @@ smtp_proceed_rset(struct smtp_session *s
 static void
 smtp_proceed_helo(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+	s->kicker_score = 0;
+
 	(void)strlcpy(s->helo, args, sizeof(s->helo));
 	s->flags &= SF_SECURE | SF_AUTHENTICATED | SF_VERIFIED;
 
@@ -1773,6 +1792,11 @@ smtp_proceed_helo(struct smtp_session *s
 static void
 smtp_proceed_ehlo(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+	s->kicker_score = 0;
+
 	(void)strlcpy(s->helo, args, sizeof(s->helo));
 	s->flags &= SF_SECURE | SF_AUTHENTICATED | SF_VERIFIED;
 	s->flags |= SF_EHLO;
@@ -1806,6 +1830,11 @@ smtp_proceed_auth(struct smtp_session *s
 	char tmp[SMTP_LINE_MAX];
 	char *eom, *method;
 
+	if (smtp_kicker(s)) {
+		return;
+	}
+	s->kicker_score = 0;
+
 	(void)strlcpy(tmp, args, sizeof tmp);
 
 	method = tmp;
@@ -1828,6 +1857,11 @@ smtp_proceed_auth(struct smtp_session *s
 static void
 smtp_proceed_starttls(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+	s->kicker_score = 0;
+
 	smtp_reply(s, "220 %s Ready to start TLS",
 	    esc_code(ESC_STATUS_OK, ESC_OTHER_STATUS));
 	smtp_enter_state(s, STATE_TLS);
@@ -1842,7 +1876,7 @@ smtp_proceed_mail_from(struct smtp_sessi
 	(void)strlcpy(tmp, args, sizeof tmp);
 	copy = tmp;
 
-       	if (!smtp_tx(s)) {
+	if (!smtp_tx(s)) {
 		smtp_reply(s, "421 %s Temporary Error",
 		    esc_code(ESC_STATUS_TEMPFAIL, ESC_OTHER_MAIL_SYSTEM_STATUS));
 		smtp_enter_state(s, STATE_QUIT);
@@ -1883,6 +1917,10 @@ smtp_proceed_quit(struct smtp_session *s
 static void
 smtp_proceed_noop(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+
 	smtp_reply(s, "250 %s Ok",
 	    esc_code(ESC_STATUS_OK, ESC_OTHER_STATUS));
 }
@@ -1892,6 +1930,10 @@ smtp_proceed_help(struct smtp_session *s
 {
 	const char *code = esc_code(ESC_STATUS_OK, ESC_OTHER_STATUS);
 
+	if (smtp_kicker(s)) {
+		return;
+	}
+
 	smtp_reply(s, "214-%s This is " SMTPD_NAME, code);
 	smtp_reply(s, "214-%s To report bugs in the implementation, "
 	    "please contact bugs@openbsd.org", code);
@@ -1902,6 +1944,10 @@ smtp_proceed_help(struct smtp_session *s
 static void
 smtp_proceed_wiz(struct smtp_session *s, const char *args)
 {
+	if (smtp_kicker(s)) {
+		return;
+	}
+
 	smtp_reply(s, "500 %s %s: this feature is not supported yet ;-)",
 	    esc_code(ESC_STATUS_PERMFAIL, ESC_INVALID_COMMAND),
 	    esc_description(ESC_INVALID_COMMAND));
@@ -1910,6 +1956,8 @@ smtp_proceed_wiz(struct smtp_session *s,
 static void
 smtp_proceed_commit(struct smtp_session *s, const char *args)
 {
+	s->kicker_score = 0;
+
 	smtp_message_end(s->tx);
 }
 
@@ -2085,6 +2133,7 @@ smtp_send_banner(struct smtp_session *s)
 {
 	smtp_reply(s, "220 %s ESMTP %s", s->smtpname, SMTPD_NAME);
 	s->banner_sent = 1;
+	s->kicker_score = 0;
 	smtp_report_link_greeting(s, s->smtpname);
 }
 
@@ -2213,6 +2262,18 @@ smtp_free(struct smtp_session *s, const 
 	free(s);
 
 	smtp_collect();
+}
+
+static int
+smtp_kicker(struct smtp_session *s) {
+	if (s->kicker_score > MAX_KICKER_SCORE) {
+		smtp_reply(s, "421 4.4.2 %s %s: Session terminated due to lack of progress",
+		    esc_code(ESC_STATUS_TEMPFAIL, ESC_OTHER_STATUS),
+		    esc_description(ESC_OTHER_STATUS));
+		smtp_enter_state(s, STATE_QUIT);
+		return 1;
+	}
+	return 0;
 }
 
 static int
Index: smtpd.c
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/smtpd.c,v
diff -u -p -r1.351 smtpd.c
--- smtpd.c	7 May 2024 12:10:06 -0000	1.351
+++ smtpd.c	8 Jun 2024 23:23:20 -0000
@@ -555,6 +555,8 @@ main(int argc, char *argv[])
 				tracing |= TRACE_TABLES;
 			else if (!strcmp(optarg, "queue"))
 				tracing |= TRACE_QUEUE;
+			else if (!strcmp(optarg, "kicker"))
+				tracing |= TRACE_KICKER;
 			else if (!strcmp(optarg, "all"))
 				tracing |= ~TRACE_DEBUG;
 			else if (!strcmp(optarg, "profstat"))
Index: smtpd.conf.5
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/smtpd.conf.5,v
diff -u -p -r1.271 smtpd.conf.5
--- smtpd.conf.5	24 Mar 2024 06:22:18 -0000	1.271
+++ smtpd.conf.5	8 Jun 2024 23:23:20 -0000
@@ -478,6 +478,10 @@ The
 table contains a mapping of IP addresses to hostnames.
 If the address on which the connection arrives appears in the mapping,
 the associated hostname is used.
+.It Cm kicker Ar on | Ar off
+Enable or disable the SMTP kicker mechanism for the current listener.
+When enabled, sessions that do not progress for too long are terminated.
+Inherits the global setting by default.
 .It Cm mask-src
 Omit the
 .Sy from
@@ -909,6 +913,12 @@ string for
 .Xr SSL_CTX_set_cipher_list 3 .
 The default is
 .Qq HIGH:!aNULL:!MD5 .
+.It Ic smtp kicker Ar on | Ar off
+Enable or disable the SMTP kicker mechanism globally.
+When enabled, the kicker will terminate sessions that do not progress
+towards submission of a message.
+The default is
+.Ar on .
 .It Ic smtp limit Cm max-mails Ar count
 Limit the number of messages to
 .Ar count
Index: smtpd.h
===================================================================
RCS file: /cvs/src/usr.sbin/smtpd/smtpd.h,v
diff -u -p -r1.686 smtpd.h
--- smtpd.h	2 Jun 2024 23:26:39 -0000	1.686
+++ smtpd.h	8 Jun 2024 23:23:20 -0000
@@ -89,6 +89,7 @@
 #define	F_MASQUERADE		0x1000
 #define	F_FILTERED		0x2000
 #define	F_PROXY			0x4000
+#define F_KICKER		0x8000
 
 #define RELAY_TLS_OPPORTUNISTIC	0
 #define RELAY_TLS_STARTTLS	1
@@ -626,6 +627,7 @@ struct smtpd {
 	char				       *sc_srs_key;
 	char				       *sc_srs_key_backup;
 	int				        sc_srs_ttl;
+	int				        sc_smtp_kicker;
 
 	char				       *sc_admd;
 };
@@ -645,6 +647,7 @@ struct smtpd {
 #define	TRACE_EXPAND	0x1000
 #define	TRACE_TABLES	0x2000
 #define	TRACE_QUEUE	0x4000
+#define TRACE_KICKER 0x8000
 
 #define PROFILE_TOSTAT	0x0001
 #define PROFILE_IMSG	0x0002
```


## Unofficial OpenSMTPD framework
I have **ideas for some filters I really want to implement**,
but I also **have to maintain several ones** among which some are fairly popular like `filter-rspamd` and `filter-senderscore`,
thanks to my 2019 article: [Setting up a mail server with OpenSMTPD, Dovecot and Rspamd](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/).

These filters were the first ones I wrote,
they do their own parsing of the wire protocol,
and whenever there's a low-level change to the filters API...
I have to make sure they don't break.

I have been wanting to **write an OpenSMTPD framework for developing extensions** in Golang,
one that would abstract low-level details,
and I finally got around to do it:

{{< github repo="poolpOrg/OpenSMTPD-framework" >}}

Using this framework,
which is still a work in progress,
you can easily **write a table or filter backend by declaring what function should be called on what event**.
The registration of hooks,
the parsing of wire protocol,
and even the handling of sessions and concurrency locking is handled transparently.

I included a tool to help test table implementations and am currently struggling with making a similar tool for filters,
since the API is considerably larger,
but it'll improve and I intend to port my existing filters to it shortly.

I also included an example table and an example filter which can be used as templates to implement your owns.


## filter-kicker
Using the OpenSMTPD-framework,
I have implemented **a clone of my kicker feature described above**:

{{< github repo="poolpOrg/filter-kicker" >}}

It took me twenty minutes from scratch,
while fixing shortcomings in the framework,
and **I ended with a working proof of concept** that I can include in this article since it's so small:

```go
package main

/*
 * Copyright (c) 2024 Gilles Chehade <gilles@poolp.org>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

import (
	"net"
	"time"

	"github.com/poolpOrg/OpenSMTPD-framework/filter"
)

type SessionData struct {
	kickcount int
}

func kicker(session filter.Session) filter.Response {
	if session.Get().(*SessionData).kickcount >= 3 {
		return filter.Disconnect("421 4.7.0 Too many commands without progressing, goodbye.")
	} else {
		return filter.Proceed()
	}
}

func filterConnectCb(timestamp time.Time, session filter.Session, rdns string, src net.Addr) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterHeloCb(timestamp time.Time, session filter.Session, helo string) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterEhloCb(timestamp time.Time, session filter.Session, helo string) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterStartTLSCb(timestamp time.Time, session filter.Session, tls string) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterAuthCb(timestamp time.Time, session filter.Session, mechanism string) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterMailFromCb(timestamp time.Time, session filter.Session, from string) filter.Response {
	return filter.Proceed()
}

func filterRcptToCb(timestamp time.Time, session filter.Session, to string) filter.Response {
	return filter.Proceed()
}

func filterDataCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterCommitCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount = 0
	return filter.Proceed()
}

func filterNoopCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount += 1
	return kicker(session)
}

func filterRsetCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount += 1
	return kicker(session)
}

func filterHelpCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount += 1
	return kicker(session)
}

func filterWizCb(timestamp time.Time, session filter.Session) filter.Response {
	session.Get().(*SessionData).kickcount += 1
	return kicker(session)
}

func main() {
	filter.Init()

	filter.SMTP_IN.SessionAllocator(func() filter.SessionData {
		return &SessionData{}
	})

	filter.SMTP_IN.ConnectRequest(filterConnectCb)
	filter.SMTP_IN.HeloRequest(filterHeloCb)
	filter.SMTP_IN.EhloRequest(filterEhloCb)
	filter.SMTP_IN.StartTLSRequest(filterStartTLSCb)
	filter.SMTP_IN.AuthRequest(filterAuthCb)
	filter.SMTP_IN.MailFromRequest(filterMailFromCb)
	filter.SMTP_IN.RcptToRequest(filterRcptToCb)
	filter.SMTP_IN.DataRequest(filterDataCb)
	filter.SMTP_IN.CommitRequest(filterCommitCb)
	filter.SMTP_IN.NoopRequest(filterNoopCb)
	filter.SMTP_IN.RsetRequest(filterRsetCb)
	filter.SMTP_IN.HelpRequest(filterHelpCb)
	filter.SMTP_IN.WizRequest(filterWizCb)

	filter.Dispatch()
}
```

The code is straightforward:
a custom session data structure is allocated and attached to the actual session,
the kickcount is increased or reset upon various events,
the `kicker()` function determines if the session needs to be kicked or if it can continue based on the kickcount value.

The filter is built as a standalone executable which can be copied to OpenSMTPD's libexec directory,
then plugged in the configuration as follows:

```conf
filter kicker proc-exec filter-kicker

listen on all filter kicker
```



# Plakar
The other big project I worked on was `plakar`,
my project for a backup solution.

In no particular order,
here's a list of what I did in this first semester.


## Began working on adaptative resources consumption
First of all,
I began instrumenting code to **obtain runtime statistics about CPU, memory and I/O consumption**.
The goal being to teach `plakar` how to play nice on a machine,
avoid exhausting resources and **detect when it should slow down** because it is putting too much pressure on the host system.

I have figured various control points.
These are positions in the code path where I can look at a specific statistics and adapt either parallelism or introduce artificial latency to slow down plakar and reduce pressure on a specific resource.

I haven't done anything worthy of an article yet,
but I prepared the mechanism and wrote the plumbing so that I can later focus on the algorithm to trigger it.


## Backup exclusions
I introduce an exclusion mechanism so that `plakar push` can be passed a file holding patterns of files to exclude from the backup while performing a snapshot.

I decided to go with **a globbing format**,
it has been tested lightly,
I'll have to test on real use-cases to see if it is complete or if it should be improved.


## Exporters: fs and s3
About a year ago,
I wrote on [VFS importers](https://poolp.org/posts/2023-08-06/plakar-vfs-importer-interface/#the-vfs-importer-interface),
connectors that **allow importing data from an external data source** into a plakar snapshot.
I ended the article saying that **an exporter interface would be nice to be able to restore snapshots on an external data store**.

I eventually came up with a VFS exporters interface,
reimplemented the filesystem restore logic into an FS exporter and adapted `plakar` to use that exporter instead of accessing the filesystem directly,
allowing me to have... made no change at all.

Except...

Except that now I was able to write an S3 exporter and restore a filesystem snapshot to an S3 bucket,
just like I was able to snapshot an S3 bucket and restore the filesystem.
**What the exporter interface brought is a filesystem-agnostic `plakar` that can snapshot from any importer,
store in any repository backend and restore to any exporter**.


## Teach ls how to list tags
A very small and minor feature but:
snapshots supported carrying tags but there was no way to list all snapshots carrying a tag.
It's now doable.


## Teach rm how to rm tags and old snapshots
Until now, `plakar rm` had to be called with one or many snapshot prefixes.
The only way to garbage collect snapshots was using the `plakar keep` command to keep the `n` latest snapshots or write a scrip to call `plakar rm` on snapshots conditionally.

I improved `plakar rm` so it would accept an `-older` option that can take a human-formatted date or a duration in various formats,
so you can delete snapshots older than a threshold.

Below is an example of **deleting all snapshots older than 15 minutes**:

```term
$ plakar ls
2024-06-12T08:36:10Z  51b910e6     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:12Z  bed7e7c3     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:12Z  d682a721     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:13Z  4317a1bb     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:13Z  3e8fb86f     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:13Z  9df0d74d     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:14Z  a3777787     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:36:14Z  a9a4f028     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
$ plakar rm --older 15m
deleting snapshot 51b910e6-1667-4fcb-be7e-4c4cea5a6e26
deleting snapshot bed7e7c3-a0e5-4804-b2ac-f4efd792f1cc
deleting snapshot d682a721-044e-40e2-8807-6fe6c454b1b4
deleting snapshot 4317a1bb-d3d2-45cf-8c75-f3a59a1142b5
deleting snapshot 3e8fb86f-7690-45c0-87fb-6a43fd4fd771
deleting snapshot 9df0d74d-4575-4dea-bf1c-7b96fe714355
deleting snapshot a3777787-f737-4592-b6cc-05074a4e9111
deleting snapshot a9a4f028-856f-45b4-b2cd-2d02595bf62e
$
```

I also introduced the ability to delete by a tag:

```
$ plakar ls           
$ plakar push         
$ plakar push
$ plakar push -tag foobar 
$ plakar ls
2024-06-12T08:55:29Z  d7c98f67     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:55:31Z  d085d68b     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
2024-06-12T08:55:40Z  dae33fa4     38 MB        0s /Users/gilles/wip/github.com/PlakarLabs/plakar/cmd/plakar
$ plakar rm -tag foobar
deleting snapshot dae33fa4-5b64-4f98-bc84-12a2f357b303
$
```

And these options can be used mutually to remove old snapshots carrying a specific tag,
I'll skip the example as you should be able to figure out what it does ;-)



## Agent mode
An additional feature that was introduced is the agent mode which allows `plakar` to run as a daemon with a configuration of tasks to perform:
```
tasks:
 - name: "task-1"
   source: "/private/etc"
   interval: "10s"
   keep: "2m"

 - name: "task-2"
   source: "/Users/gilles/Wip/github.com"
   interval: "1m"
   keep: "5m"
```

I won't expand much on this.
For now,
think of it as a glorified crontab embedded in `plakar`,
and I'll talk about agents more in details in a future article.


## Plakman
Since I had an agent mode,
I wrote the `plakman` server.

The `plakman` server controls a set of authenticated  `plakar` agents,
**distributing them tasks and waiting for completion notifications**.

This will be the topic of a dedicated article so,
I'll leave it there.


# Assorted side projects

And finally,
here's a list of **assorted stuff I did in no particular order** over the last few months.

## Wrote an article for a french security magazine
As part of a freelance gig,
I wrote an article for a French security magazine.

The article covers the philosophy of mitigations on OpenBSD,
the introduction of the `pledge(2)` system call on OpenBSD 5.9 several years ago,
how userland applications can make use of it but also how it is implemented on the kernel side.

If they publish it and order other ones,
I'll probably discuss other techniques used in OpenBSD development,
and if they don't I might just write them for this blog instead.

Reading back into OpenBSD code was very enjoyable,
and despite generally knowing how it worked I had not looked at the kernel side of `pledge(2)` so it was lots of fun to study.



## go-cdc-chunkers
The `plakar` project has been using the FastCDC chunking algorithm for a long time,
but I have bought a copy of the UltraCDC algorithm paper and made an implementation for it.

Since this has uses outside of `plakar`,
I decided to create the `go-cdc-chunkers` package to **abstract CDC chunking algorithms as I play with them**.
I released the package standalone and you are free to use it.
There are examples.

There were improvements done to FastCDC and UltraCDC since papers were published.
I haven't had the time yet to read them,
let alone implement them,
but I'll work on this package as it is a critical dependency for `plakar`.



## go-urlwatcher
The `go-urlwatcher` package allows you to **watch URLs and notify you when their content changes**.

I figured I kept hitting that same need over and over on different projects,
both at work and at home,
so I took the opportunity on a week-end to implement it and release under the ISC license so that I could use it at work on the next Monday.

Not much to say about it, here's an example of use:

```go
package main

import (
    "fmt"
    "time"
    "crypto/sha256"

    urlwatcher "github.com/poolpOrg/go-urlwatcher"
)

func notifyMe(timestamp time.Time, key string, data []byte) {
    fmt.Printf("%s: content has changed at %s, new checksum: %x\n",
        timestamp, key, sha256.Sum256(data))
}

func main() {
    r := urlwatcher.NewWatcher(&urlwatcher.DefaultWatcherConfig)
    r.Watch("https://poolp.org")
    r.Watch("https://poolp.org/test")

    // notify me forever of any change in https://poolp.org content
    r.Subscribe("https://poolp.org", notifyMe)

    // notify me of all changes in https://poolp.org/test ...
    unsubscribe := r.Subscribe("https://poolp.org/test", notifyMe)

    // ... and in a minute, I'll unsubscribe from these events
    time.Sleep(1 * time.Minute)
    unsubscribe()

    // wait forever
    <-make(chan struct{})
}
```


## go-ipcmsg
I [already wrote](https://poolp.org/posts/2021-10-26/november-2021-a-bit-of-go-ipcmsg-a-bit-of-go-privsep-and-a-ton-of-plakar/) on the `go-ipcmsg` package in the past,
it provides a simplified interface to do imsg-like (but incompatible) IPC in Golang.

I brought various improvements to it,
both in terms of interface and reliability.
It's not necessarily interesting to develop them in this article,
though I'd appreciate feedback if you play with it.

Unfortunately,
I'm using it for a project on macOS and **there seems to be a kernel bug that makes intensive fd passing unreliable**.
In theory,
when doing fd passing,
`sendmsg(2)` should ensure that the descriptor is properly referenced in the kernel side so that the sending process can `close(2)` it without preventing `recvmsg(2)` from obtaining it in the receiving process (assuming descriptors limits are correct and both `sendmsg(2)` and `recvmsg(2)` succeeded).

Yeah, well no. It seems that this is not so much guaranteed on macOS.

I hit a bug where the receiving process obtained ... a closed descriptor.
I am able to 100% reproduce the bug, with this package but also with C code, and even with OpenSMTPD.
I could also make the bug go away by deferring the `close(2)` on the sending side or by leaking descriptors there.
I have used intensive fd passing code on macOS in the past so I was surprised,
until I managed to pinpoint the bug to a specific kernel function... and see that it had been refactored not too long ago.

It may or may not be that,
I may be wrong despite my observations,
but either way the XNU kernel is considerably harder to test for me as I have no idea how I can build it and how I can boot a usable system with it.
My kernel skills are very limited and rusty,
so if I invest time troubleshooting one... it might as well be for a system I want to contribute to.

For the time being,
the project I wrote this package for is on hold,
so I don't expect any short-term improvement to `go-ipcmsg` unless there are users requesting them.


## go-agentbuilder
For some of my needs, I wanted a package that allows me to quickly implement small clients and mono process servers to be embedded in other projects.
The `go-agentbuilder` package provides a small framework for building clients and servers with a simple protocol for exchanging packets.

It uses `gob` encoding for exchanges and lets you declare what kind of structures are to be exchanged,
how to respond upon receiving them,
and supports a method for responding to a specific message with a different message while ensuring the packets are matched even if the response is asynchronous.

I discourage you from using it because there are probably nicer ways to achieve this in your own projects,
but it's tailored to my own use and there's no reason I don't publish it so... there you go.




# What's next ?
I'm expecting a little girl in the upcoming weeks.

A summer slow down is in order but...
I don't sleep so much and my son was a heavy sleeper when he was born,
so there's hope I can take care of the baby and still find time to hack on stuff ;-)

I'm also spending considerable time improving `plakar` with the idea to build a commercial project out of it,
so there's that.