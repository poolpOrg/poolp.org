---
title: "OpenSMTPD: LDAP support, selectable source, DKIM and Goodies"
date: 2012-12-14 21:43:44
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

	<p>OHAI,</p>

<p>This week has been crazy, my brain melted a little as I worked on filters, it melted a little as I worked on LDAP and it melted a little more as I tried to trick Eric into working on filters with me (yes, that actually worked :-).</p>

<p>Anyway, as usual, this is just going to be a summary of what we did as tons of little stuff have been committed here and there. You can keep track of the changes by checking the commit log of our Github mirror, or by joining our little gang on our IRC channel: #OpenSMTPD @ freenode.</p>

<p>Snapshots containing all the following features should be published tomorrow.</p>

<p>Let the fun begin !</p>

<p><strong>LDAP backend</strong></p>

<p>I had started working on LDAP support for OpenSMTPD <a href="https://poolp.org/0x52/Initial-code-for-LDAP-in-poolp-s-smtpd">a long time ago</a> but for some reason the support was never finished and ended up rotting in my sandbox.</p>

<p>A few months ago, I brought the bits back to a git branch so that I would keep running into it every few days as a reminder that I should not slack. But since I'm not too much of a LDAP fan, or a LDAP user for what it's worth, I made the branch public in hope someone would pick it up and move it forward.</p>

<p>A poolp user had started bringing the bits up to date and getting a working support in shape for aliases lookup. Resuming from there I simplified the code further and added support for almost all kinds of lookups making OpenSMTPD capable of using LDAP as a backend for the most common use-cases.</p>

<p>Here's a configuration file to authenticate local users, lookup a domain and perform aliases lookups against LDAP:</p>

<p><code></p>

<h1>/etc/mail/smtpd.conf</h1>

<p>table myldaptable ldap:/etc/mail/ldapd.conf</p>

<p>listen on egress tls auth <myldaptable></p>

<p>accept for domain <myldaptable> alias <myldaptable> deliver to maildir
accept for any relay
</code></p>

<p>and here's the table configuration:</p>

<p><code></p>

<h1>/etc/mail/ldapd.conf</h1>

<p>url             ldap://127.0.0.1
username        cn=admin,dc=opensmtpd,dc=org
password        thisbemypasswd
basedn          dc=opensmtpd,dc=org</p>

<h1>aliases lookup</h1>

<p>alias_filter            (&(objectClass=courierMailAlias)(uid=%s))
alias_attributes        maildrop</p>

<h1>credentials lookup</h1>

<p>credentials_filter      (&(objectClass=posixAccount)(uid=%s))
credentials_attributes  uid,userPassword</p>

<h1>domains lookup</h1>

<p>domain_filter           (&(objectClass=rFC822localPart)(dc=%s))
domain_attributes       dc
</code></p>

<p>The support is functional but it needs to be improved further as it currently has two drawbacks: the backend does not reconnect to the LDAP server should it lose the connection, and it uses the aldap synchronous API which means that queries that take time to complete will be heavy on the lookup process.</p>

<p>Also, I only tested with OpenBSD's ldapd(8) as it was dead simple and I didn't want to add more pain than required on my shoulders. Turns out, it did make my experiment far more enjoyable that I would have assumed ;-)</p>

<p>Oh, and I ordered a LDAP book to get more familiar with the service as I suspect I'll be getting questions regarding LDAP every now and then given how many times it's been requested in the past. I might as well know what I'm talking about :-)</p>

<p><strong>Source address selection</strong></p>

<p>Eric has plugged the K_SOURCE lookup service to relay rules, this allows OpenSMTPD to perform a lookup of the source address it should use when the transfer process establishes an outgoing connection to a relay.</p>

<p>Until now OpenSMTPD could not force the IP address it used for outgoing trafic without relying on a hardcoded hack that was committed to the poolp branch. It was done this way on purpose and we delayed this feature until the other parts were rewritten appropriately for the puzzle to fit right.</p>

<p>It is now possible to force an address using the <i>source</i> keyword:</p>

<p><code>
table myaddrs { 88.190.237.114, 91.121.164.52 }</p>

<p>accept for any relay source <myaddrs>
</code></p>

<p>If multiple addresses are provided, they will be cycled through, and the mta code will detect which ones are no longer usable.</p>

<p><strong>Intermediate bounces</strong></p>

<p>A feature we had a long time ago and which disappeared during a cleanup was the support of intermediate bounces.</p>

<p>When OpenSMTPD fails to deliver a mail it has to notify the sender that the message was never delivered. It sometimes happens immediately, but sometimes the failure may be temporary and the daemon keeps the message and tries to deliver it every now and then (ok, the logic is slightly more complex, but you get the idea). In such cases, the bounce will not be sent before OpenSMTPD gives up on trying after 4 days by default.</p>

<p>Obviously, getting a mail 4 days later to tell you that no one read yours when you assumed it was already in the recipients mailbox for a while is quite irritating. The intermediate bounce will instead notify the sender that an error occured after a few temporarily failed deliveries, and let him know that the daemon will keep trying to deliver for a while.</p>

<p>After discussions, Eric reimplemented intermediate bounces in OpenSMTPD but did it in a slightly different way than with other daemons. By default, an intermediate bounce will be sent after a mail has been sitting in the queue for over 4 hours without being delivered ... but in addition, a set of delays may be provided in smtpd.conf to send multiple intermediate bounces. For example, if I wanted intermediate bounces to be sent after 4 hours, after a day and after two days, I could simple use:</p>

<p><code>
bounce-warn 4h, 1d, 2d
</code></p>

<p>The keyword may change, but the idea and code is here and working.</p>

<p><strong>Tags & DKIM example</strong></p>

<p>I've implemented tagging of sessions a very long time ago, I <em>think</em> it was actually already there when OpenSMTPD was not yet OpenSMTPD but still a poolp project :-)</p>

<p>The feature was hidden and undocumented, it had uses but so limited that I did not want users to start using it in random situations that I would have to cope with later. Basically, a listener may tag all sessions initiated through it; then rules may apply to specific tags allowing some rules to apply only to some sessions.</p>

<p>Eric realized that this was perfect to deal with one of our use-cases: DKIM signatures.</p>

<p>We want DKIM signatures but we don't necessarily want to write a filter for that as there are already tools that do the work. So we need to accept a message, pass it to a DKIM signing tool, which will in turn pass it back to us, so that we can send it where it needs to be sent.</p>

<p>A tool to do this is DKIMproxy. I won't enter the details behind DKIMproxy, I'll actually post a description of how to setup OpenSMTPD with it very soon on this blog. But the idea is that using tags we can determine which sessions we want forwarded to DKIMproxy, and which sessions are coming back from it and need to be relayed to the final destination:</p>

<p><code></p>

<h1>listen on all interfaces that are attached to the default route</h1>

<p>listen on egress</p>

<h1>listen on loopback interface on port 10028 and tag DKIM</h1>

<p>listen on lo0 port 10029 tag DKIM</p>

<h1>only accept to relay the sessions that are tagged DKIM</h1>

<p>accept tagged DKIM for any relay</p>

<h1>this is reached by the sessions that are NOT tagged</h1>

<h1>and will cause OpenSMTPD to relay to the DKIMproxy</h1>

<p>accept for any relay via smtp://127.0.0.1:10028
</code></p>

<p>Now if I were to send mail to my gmail account, I would connect to the daemon, my session would not be tagged so it would match the last rule causing the message to be send to DKIMproxy. DKIMproxy would then relay back the mail to my loopback interface on port 10029 which would have the new session tagged DKIM causing the first rule to be matched. 4 lines. ridiculous.</p>

<p><strong>OpenSMTPD goodies</strong></p>

<p>Here it is, OpenSMTPD goodies, not for sale, limited to friends and a few spare I will give away or sell at low price depending on how many are left.</p>

<p><center><img src="https://pbs.twimg.com/media/A-GjSmUCUAAwfxB.jpg"></center></p>

<p>If you were kind enough to contribute and finish the FAQ section of the OpenSMTPD website, I might just send you a mug and a tshirt signed by Charles, Eric and I ;-)</p>

<p>Time to go zZz, stay tuned for more news !</p>
