---
title: "OpenSMTPD: stress and features"
date: 2013-01-18 22:11:07
category: OpenSMTPD
authors:
 - "gilles"
categories:
 - technology
---

	<p>Hey,</p>

<p>I wanted to publish something last Friday but I unfortunately got my hands on a copy of Super Nintendo's Zelda which led to an instant loss of motivation.</p>

<p>Here's a quick summary of what happened recently in OpenSMTPD-land, it's not an exhaustive description and to be honest I'm keeping a lot of information private at this point and probably until release ;-)</p>

<p>A snapshot should be published tomorrow.</p>

<p><strong>Stressing the daemon</strong></p>

<p>First of all, we have done a lot of stress testing lately.</p>

<p>We have tested the incoming path, accepting hundreds of thousands of mails from hundreds of sessions, arriving in chaotic order and with random data. We're quite confident that our incoming path is rock solid now, we will be performing our final test very soon with one billion messages.</p>

<p>We have also tested our outgoing path, first in a confined environment which revealed no bugs, and then in a live environment which revealed minor bugs that where fixed in the process. There is still one "memory usage"-related issue but it applies to very special and stressed setups, not something people would usually experience, and we happen to have a diff for it which requires some additional work before being in a state proper for commit.</p>

<p>During these stress tests, we have gathered a lot of information to prove some of our theories right and wrong. We now know where we stand with regard to other MTA and we have built tools which allow us to have a very precise understanding of our areas of improvements. Clearly, we have no reason to be ashamed, far from it.</p>

<p><strong>optimization</strong></p>

<p>Our queue code has a design that allows for very efficient queues to be written.
We know that because we have already written several backends, we know the time spent in the API, the time spent in the backends and for disk-based backend the exact cost of the disk IO.</p>

<p>You'd assume our queue would be very fast, but the default backend we ship is sub-optimal <strong>by design</strong> because the admin that is in me wants the queue to provide some features that are incompatible with performances.</p>

<p>One of these features is to provide per-envelope files as well as a locality between envelopes and messages so that SMTP transactions can be backed up and restored easily on another machine. Stuff like that.</p>

<p>This user-friendliness means that we can't rely on tricks to avoid hitting the disk too often, we have as many open()-fsync()-close() as there are envelopes; we have as many mkdir()/rename() as there are messages; and you can add many more file-system related calls used to deal with the atomicity of our queue commits. A <strong>lot</strong> of this is unnecessary for OpenSMTPD and could be handled in a much more efficient way... but is really only done so that a human can inspect and manipulate the queue more easily.</p>

<p>A queue that allowed for envelopes to be written sequentially in a single binary file that would require only one fsync() could increase drastically our performances, and we will surely do that at some point, but our default queue will always be the user-friendly one even at the cost of a slower incoming path.</p>

<p>That being said, we still want our queue to operate fast and limit the impact of our design. Ideally, our queue shouldn't be more than 10% slower than the other software with their optimized queues. So...</p>

<p>I spent some time tracing our queue code and spotted that the system calls pattern was not really matching what I'd expect. Our queue logic is very simple and has a very "linear" pattern, several system calls should have a matching number of calls. I tracked and fixed until a kdump output displayed the optimal number of calls for each system calls according to the numbers I had on paper. I added a couple functions to help profile every single queue operation, Eric came up with a better interface for these functions and we now have an invaluable tool for queue backend development :-)</p>

<p>Eric also spent time improving our memory usage and removing some pressure from inter-process IO by coming up with an API that allows to encode/decode the data more efficiently. Until now we passed structures, which could contain huge buffers for which we only consumed a few bytes; now the new API not only passes only the required data but also provides type checking which allows us to make sure we don't pass the wrong data by mistake. As a bonus, the API can use the types to know the average size of the data and only reallocate in cases where we exceed that size.</p>

<p>When we were done with this, we were <strong>very</strong> good with the outgoing path, and we were pretty much equivalent to the other MTA with regard to the incoming path. There is still a lot of room for improvement, but given the constraints we imposed ourselves I'm really glad that we're not twice as slow as the slowest MTA out there :-)</p>

<p><strong>More resistant fs-queue</strong></p>

<p>Our default queue had a design that was very strict with "strange" situations.</p>

<p>Any time an unexpected situation happened, the daemon would fatal. Since unexpected situations are not supposed to happen, this shouldn't be a problem right ?</p>

<p>No. Not right. On a regular setup this never happens, but sometimes a human does something as innocent as a chmod on a file or directory... and OpenSMTPD figures something is not normal and commits suicide.</p>

<p>To be honest not only did I not receive a report of a queue fatal in months, but I also don't recall ever hitting any of these on poolp.org... until the stress.</p>

<p>I hit a couple fatal() which turned out to be related to an error in our use of the fts(3) API which for some reason didn't trigger until the stress. I fixed the issue then decided to go track every single fatal() in the queue code and try to convert it into a temporary failure condition so that even if a failure happened, the daemon would deal with it gracefully.</p>

<p>It turned out to be much simpler than I assumed and our fs-queue is now capable of coping with an admin messing with the queue. Of course, an admin should never tweak with the queue, but being able to not fatal() on a chmod or mv felt like essential ;-)</p>

<p><strong>Per-listener hostname</strong></p>

<p>Our smtpd.conf file had a "hostname" directive which allowed setting the hostname to display on the greeting banner.</p>

<p>The directive was removed and it is now possible to specify the hostname for each listener:</p>

<p><code>
listen on lo0
listen on 192.168.1.1 hostname mx1.int.poolp.org
listen on 192.168.2.1 hostname mx2.int.poolp.org
</code></p>

<p>Not specifying one will use the machine's hostname.</p>

<p><strong>Per-source HELO</strong></p>

<p>When relaying mail, smtpd.conf allows for overriding the source address using a table containing an address or a list of addresses:</p>

<p><code>
table myaddrs { 192.168.1.2, 192.168.1.3 }</p>

<p>accept for all relay source <myaddrs>
</code></p>

<p>The above causes the relaying to bind one of these addresses a the source address. However, during a SMTP session, our MTA has to advertise itself at the HELO/EHLO stage and tell its hostname. The hostname is sometimes checked and if it doesn't match the source address used, the MTA is rejected.</p>

<p>So we needed a way to have our MTA provide the remote host with a HELO/EHLO parameter that corresponds with the source address used. We had ideas that involved performing a DNS lookup from MTA but it would not work due to NAT.</p>

<p>I suggested that we use a new lookup service K_ADDRNAME which allows for a mapping of an IP address to a name:</p>

<p><code>
table myaddrs { 192.168.1.2, 192.168.1.3 }
table myhelo  { 192.168.1.2 => mx1.poolp.org, 192.168.1.3 => mx2.poolp.org }</p>

<p>accept for all relay source <myaddrs> helo <myhelo>
</code></p>

<p>With the following, MTA will use a source from the table myaddrs and, at HELO/EHLO time, will use the name from the table myhelo that matched the address it used to connect.</p>

<p><strong>Sender filtering</strong></p>

<p>Another feature that people have been requesting very frequently is the ability to use a sender email address as a matching condition in the ruleset.</p>

<p>Until recently, the matching of a rule was done by looking at the client address and the destination domain. It was not possible to express something like "accept all mail coming with sender <a href="mailto:gilles@poolp.org">gilles@poolp.org</a>" or "reject all mail coming with sender @redhat.com".</p>

<p>I have introduced the "sender" filtering which can apply to a full email address or to a domain, both in accept and reject rules. It works as follow:</p>

<p><code></p>

<h1>accept from any source, if sender has domain @openbsd.org [...]</h1>

<p>accept from any sender "@openbsd.org" for any relay</p>

<h1>accept from localhost, if sender has domain @poolp.org [...]</h1>

<p>accept sender "@poolp.org" for any relay</p>

<h1>accept from any source, only if sender is <a href="mailto:gilles@poolp.org">gilles@poolp.org</a> [...]</h1>

<p>accept sender <a href="mailto:gilles@poolp.org">gilles@poolp.org</a> for any relay
</code></p>

<p>It can apply to relay or deliver rules, and allows the use of tables to apply different relay rules to different domains or users coming from different networks.</p>

<p><code>
table hackers { "@opensmtpd.org", "@poolp.org" }
table slackers { <a href="mailto:richard@foot-cheese.org">richard@foot-cheese.org</a>, <a href="mailto:lennart@thepig.org">lennart@thepig.org</a> }</p>

<p>accept from 192.168.1.0/24 sender <hackers> relay
accept from 192.168.2.0/24 sender <slackers> relay via smtp://example.org
</code></p>

<p><strong>Various little fixes</strong></p>

<p>You have no idea.</p>

<p>We have fixed various little bugs that triggered in very specific cases which you just can't hit out of a live test.</p>

<p>We also fixed/improves/rework minor things, like making the "relay backup" parameter optional by picking the machine hostname, changing the API for queue remove to take an evpid instead of a full envelope, removing userinfo from a structure that wasn't using it, switching the queue code to use the first two bytes of a msgid instead of the last two bytes to create the bucket, fixing a segfault with a specific configuration file, allowing authentication to fail temporarily, etc, etc, etc ...</p>

<p>Oh and on a completely useless and unrelated note:</p>

<p>I installed OpenSMTPD on a raspberry, so here's a picture of my mailserver at home:</p>

<p><center>
    <img src="https://pbs.twimg.com/media/BAq-pN0CAAEMMM5.jpg:large">
</center></p>
