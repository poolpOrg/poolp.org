---
title: "Install OpenBSD on dedibox with full-disk encryption"
date: 2018-01-29 23:28:00
category: OpenBSD
authors:
 - Gilles Chehade
categories:
 - technology
---

	TL;DR:
	I run several "dedibox" servers at online.net, all powered by OpenBSD.
	OpenBSD is not officially supported so you have to work-around.
	Running full-disk encrypted OpenBSD there is a piece of cake.
	As a bonus, my first steps within a brand new booted machine ;-)

Step #0: choosing your server
--
OpenBSD is not officially supported,
I can't guarantee that this will work for you on any kind of server online.net provides,
however I've been running [https://poolp.org](https://poolp.org) on OpenBSD there since 2008,
only switching machines as they were getting a bit old and new offers came up.

Currently,
I'm running two SC 2016 (SATA) and one XC 2016 (SSD) boxes,
all three running OpenBSD reliably ever since I installed them.

Recently I've been willing to reinstall the XC one after I did some experiments that turned it into a FrankenBSD,
so this was the right occasion to document how I do it for future references.

[I wrote an article similar to this a few years ago relying on `qemu` to install to the disk](https://poolp.org/posts/2013-05-08/installer-openbsd-sur-une-dedibox-classic-gen2/),
since then online.net provided access to a virtual serial console accessed within the browser,
making it much more convenient to install without the `qemu` indirection which hid the NIC devices and disks duid and required tricks.

The method I currently use is a mix and adaptation from the techniques described in
[https://www.2f30.org/guides/openbsd-dedibox.html](https://www.2f30.org/guides/openbsd-dedibox.html)
to boot the installer,
and the technique described in
[https://geekyschmidt.com/2011/01/19/configuring-openbsd-softraid-fo-encryption.html](https://geekyschmidt.com/2011/01/19/configuring-openbsd-softraid-fo-encryption.html)
to setup the crypto slice.


Step #1: boot to rescue mode
--
The web console has a rescue mode which will essentially boot the server on a system running in RAM.

This usually allows you to unfuck a broken system by booting a Linux or FreBSD system,
mounting disks,
making appropriate changes to the disk,
then rebooting back to the original system.

We will actually make use of this rescue mode to write an OpenBSD boot disk image at the beginning of the real disk,
allowing us to reboot the server right into the OpenBSD intaller.

```text
laptop$ ssh gilles@163.172.61.249
[...]
gilles@163-172-61-249:~$ wget https://ftp.fr.openbsd.org/pub/OpenBSD/6.2/amd64/miniroot62.fs
[...]
gilles@163-172-61-249:~$ sudo dd if=miniroot62.fs of=/dev/sda
[sudo] password for gilles:
9600+0 records in
9600+0 records out
4915200 bytes (4,9 MB, 4,7 MiB) copied, 0,116668 s, 42,1 MB/s
gilles@163-172-61-249:~$
```

You can then reboot back to normal mode and activate the serial console from the web console.


Step #2: boot to the installer
--
On the serial console, you will be greeted by the bootloader prompt.
For some reason,
every couple seconds a keystroke gets triggered by the interface causing the 'n' character to be inserted.
It takes a bit of synchronization but you should be able to set tty to com1,
getting rid of the keystroke and allowing proper output to the terminal:

```text
boot> set tty com1
>> OpenBSD/amd64 BOOT 3.33
boot> boot
```
The bootloader will load a ramdisk kernel and drop you into the installer,
which is our next step.


Step #3: prepare softraid
--
Once the installer is started,
drop immediately into a shell:

```text
Welcome to the OpenBSD/amd64 6.2 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? s
#
```

First of all, we need to rewrite the MBR after we messed it up with our miniroot trick:
```text
# fdisk -iy sd0
Writing MBR at offset 0.
#
```

Then, we can enter the disklabel to setup a RAID slice and a swap slice,
keeping in mind that swap is already encrypted by default on OpenBSD.
The RAID slice will be used to setup softraid with the crypto discipline.
```text
# disklabel -E sd0
Label editor (enter '?' for help at any prompt)
> p M
OpenBSD area: 64-500103450; size: 244191.1M; free: 244191.1M
#                size           offset  fstype [fsize bsize   cpg]
  c:        244198.3M                0  unused
> a
partition: [a] 
offset: [64]
size: [500103386] 240000M
Rounding size to cylinder (16065 sectors): 491524676
FS type: [4.2BSD] RAID
> a
partition: [b]
offset: [491524740]
size: [8578710]
FS type: [swap]
> w
> q
No label changes.
#
```

Once this is done, we can use bioctl to setup the encrypted slice:

```text
# bioctl -c C -r auto -l /dev/sd0a softraid0
New passphrase:
Re-type passphrase:
sd1 at scsibus1 targ 1 lun 0: <OPENBSD, SR CRYPTO, 006> SCSI2 0/direct fixed
sd1: 240002MB, 512 bytes/sector, 491524148 sectors  
softraid0: CRYPTO volume attached as sd1
#
```

At this point, we're ready to perform a regular install... __on sd1, not sd0, beware__,:
```text
# install
At any prompt except password prompts you can escape to a shell by
typing '!'. Default answers are shown in []'s and are selected by
pressing RETURN.  You can exit this program at any time by pressing
Control-C, but this can leave your system in an inconsistent state.

Terminal type? [vt220]
System hostname? (short form, e.g. 'foo') pocs

Available network interfaces are: em0 em1 vlan0.
Which network interface do you wish to configure? (or 'done') [em0]
IPv4 address for em0? (or 'dhcp' or 'none') [dhcp]
em0: DHCPDISCOVER - interval 1
em0: DHCPOFFER from 163.172.61.1 (00:81:c4:f6:e9:17)
em0: DHCPREQUEST to 255.255.255.255
em0: DHCPACK from 163.172.61.1 (00:81:c4:f6:e9:17)
em0: bound to 163.172.61.249 -- renewal in 2147483647 seconds
IPv6 address for em0? (or 'autoconf' or 'none') [none]
Available network interfaces are: em0 em1 vlan0.
Which network interface do you wish to configure? (or 'done') [done]
Default IPv4 route? (IPv4 address or none) [163.172.61.1]
add net default: gateway 163.172.61.1
Using DNS domainname online.net
Using DNS nameservers at 62.210.16.6 62.210.16.7

Password for root account? (will not echo)
Password for root account? (again)
Start sshd(8) by default? [yes]
Change the default console to com1? [yes]
Available speeds are: 9600 19200 38400 57600 115200.
Which speed should com1 use? (or 'done') [9600]
Setup a user? (enter a lower-case loginname, or 'no') [no] gilles
Full name for user gilles? [gilles]
Password for user gilles? (will not echo)
Password for user gilles? (again)
WARNING: root is targeted by password guessing attacks, pubkeys are safer.
Allow root ssh login? (yes, no, prohibit-password) [no]
What timezone are you in? ('?' for list) [Europe/Paris]
```

I insist again, you want to write to the new softraid-backed disk:
```text
Available disks are: sd0 sd1.
Which disk is the root disk? ('?' for details) [sd0] sd1
No valid MBR or GPT.
Use (W)hole disk MBR, whole disk (G)PT or (E)dit? [whole]
Setting OpenBSD MBR partition to whole sd1...done.
The auto-allocated layout for sd1 is:
#                size           offset  fstype [fsize bsize   cpg]
  a:             1.0G               64  4.2BSD   2048 16384     1 # /
  b:            16.2G          2097216    swap
  c:           234.4G                0  unused
  d:             4.0G         36067392  4.2BSD   2048 16384     1 # /tmp
  e:            29.5G         44455968  4.2BSD   2048 16384     1 # /var
  f:             2.0G        106342848  4.2BSD   2048 16384     1 # /usr
  g:             1.0G        110537152  4.2BSD   2048 16384     1 # /usr/X11R6
  h:            10.0G        112634304  4.2BSD   2048 16384     1 # /usr/local
  i:             2.0G        133605824  4.2BSD   2048 16384     1 # /usr/src
  j:             6.0G        137800128  4.2BSD   2048 16384     1 # /usr/obj
  k:           162.7G        150383040  4.2BSD   4096 32768     1 # /home
Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout? [a]
```

Just for the purpose of simplifying, I will create a custom layout with only a root slice,
note that this is insecure and,
unless you know what you're doing you should avoid that as it exposes your system to a denial of service.

The swap slice is redundant with the one we already create along the RAID slice in sd0,
so the options are either to stick with the auto layout or to create a custom layout that's similar but without swap.

At the very very least,
you want to isolate / from /tmp, /var, /usr and /home,
though isolating it from /usr/local is not a bad idea.

```text
Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout? [a] C
Label editor (enter '?' for help at any prompt)
> p
OpenBSD area: 64-491508675; size: 491508611; free: 491508611
#                size           offset  fstype [fsize bsize   cpg]
  c:        491524148                0  unused
> a
partition: [a]
offset: [64]
size: [491508611]
FS type: [4.2BSD]
mount point: [none] /
Rounding size to bsize (64 sectors): 491508608
> w
> q
No label changes.
/dev/rsd1a: 239994.4MB in 491508608 sectors of 512 bytes
295 cylinder groups of 814.44MB, 26062 blocks, 52224 inodes each
Available disks are: sd0.
Which disk do you wish to initialize? (or 'done') [done]
/dev/sd1a (448be85498897147.a) on /mnt type ffs (rw, asynchronous, local)

Let's install the sets!
Location of sets? (disk http or 'done') [http]
HTTP proxy URL? (e.g. 'http://proxy:8080', or 'none') [none]
HTTP Server? (hostname, list#, 'done' or '?') [ftp.fr.openbsd.org]
Server directory? [pub/OpenBSD/6.2/amd64]

Select sets by entering a set name, a file name pattern or 'all'. De-select
sets by prepending a '-', e.g.: '-game*'. Selected sets are labelled '[X]'.
    [X] bsd           [X] base62.tgz    [X] game62.tgz    [X] xfont62.tgz
    [X] bsd.mp        [X] comp62.tgz    [X] xbase62.tgz   [X] xserv62.tgz
    [X] bsd.rd        [X] man62.tgz     [X] xshare62.tgz
Set name(s)? (or 'abort' or 'done') [done]
Get/Verify SHA256.sig   100% |**************************|  2152       00:00
Signature Verified
Get/Verify bsd          100% |**************************| 12777 KB    00:08
Get/Verify bsd.mp       100% |**************************| 12858 KB    09:46
Get/Verify bsd.rd       100% |**************************|  9565 KB    00:05
Get/Verify base62.tgz   100% |**************************|   139 MB    01:24
Get/Verify comp62.tgz   100% |**************************| 75525 KB    50:08
Get/Verify man62.tgz    100% |**************************|  7008 KB    00:03 
Get/Verify game62.tgz   100% |**************************|  2718 KB    01:56
Get/Verify xbase62.tgz  100% |**************************| 17964 KB    12:29    
Get/Verify xshare62.tgz 100% |**************************|  4417 KB    02:43    
Get/Verify xfont62.tgz  100% |**************************| 39342 KB    00:20    
Get/Verify xserv62.tgz  100% |**************************| 12572 KB    00:06    
Installing bsd          100% |**************************| 12777 KB    00:00    
Installing bsd.mp       100% |**************************| 12858 KB    00:00    
Installing bsd.rd       100% |**************************|  9565 KB    00:00    
Installing base62.tgz   100% |**************************|   139 MB    00:10    
Extracting etc.tgz      100% |**************************|   189 KB    00:00    
Installing comp62.tgz   100% |**************************| 75525 KB    00:08    
Installing man62.tgz    100% |**************************|  7008 KB    00:01    
Installing game62.tgz   100% |**************************|  2718 KB    00:00    
Installing xbase62.tgz  100% |**************************| 17964 KB    00:01    
Extracting xetc.tgz     100% |**************************|  7036       00:00    
Installing xshare62.tgz 100% |**************************|  4417 KB    00:01    
Installing xfont62.tgz  100% |**************************| 39342 KB    00:02    
Installing xserv62.tgz  100% |**************************| 12572 KB    00:01    
Location of sets? (disk http or 'done') [done] Location of sets? (disk http or 'done') [done] 
Saving configuration files...done.
Making all device nodes...done.
Multiprocessor machine; using bsd.mp instead of bsd.
Relinking to create unique kernel...done.

CONGRATULATIONS! Your OpenBSD install has been successfully completed!
To boot the new system, enter 'reboot' at the command prompt.
When you login to your new system the first time, please read your mail
using the 'mail' command.

# 
```


Step #4: reboot to encrypted OpenBSD system
--
The root slice being encrypted, you'll need to type your password every time you reboot.

Connect to the serial console,
you should see the boot prompt asking for a password.
The 'n' glitch is still around,
so either you break out of password by typing enter so you can do the 'set tty com1' trick,
or you do as I do and synchronize with the 'n' keystroke to delete it and type password really fast.

```text
# reboot
syncing disks... done
sd1 detached
rebooting...
[...]
Using drive 0, partition 3.                                                     
Loading......                                                                   
probing: pc0 com0 com1 mem[632K 2009M 14336M a20=on]                            
disk: hd0+ sr0*                                                                 
>> OpenBSD/amd64 BOOT 3.33                                                      
Passphrase: 
boot> set tty com1
>> OpenBSD/amd64 BOOT 3.33
boot> boot
Passphrase: 
[...]

OpenBSD/amd64 (pocs.online.net) (tty01)

login: gilles
Password:
OpenBSD 6.2 (GENERIC.MP) #134: Tue Oct  3 21:22:29 MDT 2017

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

You have new mail.
$
```

As suggested by `semarie@`,
once logged in you can take opportunity to fix the 'n' glitch by setting tty in boot.conf:

```text
$ su
Password:
# echo set tty com1 > /etc/boot.conf
```


Bonus: further tightening your system
--
These are the few steps I immediately do to tighten up my systems furthers:

# enable doas

The 'gilles' account I created at install is part of the 'wheel' group,
which turns out to be exactly what the example doas.conf allows:
```text
$ su
Password:
# cat /etc/examples/doas.conf |tail -1
permit keepenv :wheel
# cp /etc/examples/doas.conf /etc
# exit
$ doas sh
doas (gilles@pocs.online.net) password:
#
```

# disable the root account

Now that 'gilles' can use `doas` we no longer ever need to authenticate as root,
so disable it by setting the password to '*'.
This will prevent 'root' from being usable directly or through `su`,
yet if really needed 'gilles' can still `doas su` to obtain a shell running as user 'root':
```text
# usermod -p'*' root
#
```

# update system with syspatch

The brand new system may require some patches to be applied,
the `syspatch` command written by ajacoutot@ performs a binary patching of the system,
then causes the kernel to be relinked using the KARL mechanism to shuffle objects order:
```text
# syspatch
Get/Verify syspatch62-001_tcb_inv... 100% |*************|   465 KB    00:00    
Installing patch 001_tcb_invalid
Get/Verify syspatch62-002_fktrace... 100% |*************|   785 KB    00:21    
Installing patch 002_fktrace
Get/Verify syspatch62-003_mpls.tgz 100% |***************|   837 KB    00:00    
Installing patch 003_mpls
Get/Verify syspatch62-004_libssl.tgz 100% |*************|  2515 KB    00:01    
Installing patch 004_libssl
Relinking to create unique kernel... done.
```

# add my ssh public key to my ~/.ssh/authorized_keys

Password for accessing SSH are bad,
copy the SSH public key generated on my laptop with `ssh-keygen` to the authorized_keys file of my account on the server:
```text
# echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII19dxt4hf1t4Lo05TaPZQqrtwtszHHyAF7ctPYPRIvp gilles@debug.poolp.org'>>~gilles/.ssh/authorized_keys
#
```

# disable password authentication within ssh

No reason to allow PasswordAuthentication anymore,
disable in sshd_config and restart sshd:
```text
# echo PasswordAuthentication no >> /etc/ssh/sshd_config
# /etc/rc.d/sshd restart                                                       
sshd(ok)
sshd(ok)
#
```

# reboot so you boot on a brand new up-to-date system with latest stable kernel
```text
# reboot
syncing disks... done
sd1 detached
rebooting...
[...]
Using drive 0, partition 3.                                                     
Loading......                                                                   
probing: pc0 com0 com1 mem[632K 2009M 14336M a20=on]                            
disk: hd0+ sr0*                                                                 
>> OpenBSD/amd64 BOOT 3.33                                                      
Passphrase: 
boot> set tty com1
>> OpenBSD/amd64 BOOT 3.33
boot> boot
Passphrase: 
[...]

OpenBSD/amd64 (pocs.online.net) (tty01)

login:
Password:
```

VOILA !

---
Want to comment ? [Open an issue on Github](https://github.com/poolpOrg/poolpOrg.github.io/issues/)
