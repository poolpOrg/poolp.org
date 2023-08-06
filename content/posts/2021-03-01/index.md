---
title: "March 2021: backups with Plakar"
date: 2021-03-26 16:52:00 +0200
authors:
 - "gilles"
categories:
 - technology
tags:
 - plakar
 - backups
---

{{< tldr >}}
I wrote a backup utility called plakar.
{{< /tldr >}}


# Let's start with some LoFi

Relax.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/3t0dxu6FkXk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

I have a [youtube channel](https://www.youtube.com/c/GillesChehade) (subscribe ! now !)


# Plakar: Yet another backup utility ?

Yes, I wrote another backup utility.

I backup many hosts on a daily basis and while there are many different tools,
I always fallback to using scripts built on top of `rsync` to do incremental backups,
which **always results in customization** for this or that host.

I want backups that are **easy to perform and restore**,
that **do not require me to remember a ton of options**,
that lets me work with snapshots **independant one of another** instead of increments,
that allow me to **browse the content easily** and **compare snapshots** without restoring them,
that **store data efficiently** (compressed and deduped),
and finally that lets me back things up locally or remotely to machines I own.

I've looked into various solutions and I figured that the best option **for me** was to write a tool that did exactly what I wanted.
Also, it's fun to write code.

# What is `plakar` ?

The `plakar` utility is a tool that performs backups as **snapshots of a tree structure**,
scanning subdirectories and files,
and storing them efficiently in a repository while deduplicating redundant data.
Once a snapshot has been taken of a directory,
it can be inspected and restored at will.

<center>
  <img src="feature.png">
</center>

# How does it work ?

To perform a backup,
you only need to call `plakar` with the `push` subcommand and a set of directories that should be part of the snapshot.
This will cause `plakar` to scan the directories,
collect inode information regarding every subdirectory and files,
split files into chunks and **push them to the repository**.

```sh
$ plakar push /bin
82711aee-6485-445b-98f4-6cae51c83035: OK
$
```

The `ls` subcommand allows **listing snapshots** present in the repository:

```sh
$ plakar ls
82711aee-6485-445b-98f4-6cae51c83035 [2021-03-27T22:05:00Z] (size: 10 MB, files: 36, dirs: 1)
$
```

The `pull` subcommand allows **restoring a particular snapshot**:
```sh
$ plakar pull 82711aee-6485-445b-98f4-6cae51c83035
82711aee-6485-445b-98f4-6cae51c83035: OK

$ ls -l bin
total 9336
-rwxr-xr-x  1 root  wheel   121104  1 Jan  2020 [
-r-xr-xr-x  1 root  wheel  1296640  1 Jan  2020 bash
-rwxr-xr-x  1 root  wheel   121968  1 Jan  2020 cat
-rwxr-xr-x  1 root  wheel   107552  1 Jan  2020 chmod
-rwxr-xr-x  1 root  wheel   123232  1 Jan  2020 cp
-rwxr-xr-x  1 root  wheel  1104640  1 Jan  2020 csh
-rwxr-xr-x  1 root  wheel   277408  1 Jan  2020 dash
-rwxr-xr-x  1 root  wheel   139264  1 Jan  2020 date
-rwxr-xr-x  1 root  wheel   122160  1 Jan  2020 dd
-rwxr-xr-x  1 root  wheel   121840  1 Jan  2020 df
-rwxr-xr-x  1 root  wheel   120848  1 Jan  2020 echo
-rwxr-xr-x  1 root  wheel   205648  1 Jan  2020 ed
-rwxr-xr-x  1 root  wheel   121504  1 Jan  2020 expr
-rwxr-xr-x  1 root  wheel   120864  1 Jan  2020 hostname
-rwxr-xr-x  1 root  wheel   121232  1 Jan  2020 kill
-r-xr-xr-x  1 root  wheel  2552352  1 Jan  2020 ksh
-rwxr-xr-x  1 root  wheel   329344  1 Jan  2020 launchctl
-rwxr-xr-x  1 root  wheel   105136  1 Jan  2020 link
-rwxr-xr-x  1 root  wheel   105136  1 Jan  2020 ln
-rwxr-xr-x  1 root  wheel   157360  1 Jan  2020 ls
-rwxr-xr-x  1 root  wheel   104752  1 Jan  2020 mkdir
-rwxr-xr-x  1 root  wheel   106176  1 Jan  2020 mv
-rwxr-xr-x  1 root  wheel   291152  1 Jan  2020 pax
-rwsr-xr-x  1 root  wheel   173568  1 Jan  2020 ps
-rwxr-xr-x  1 root  wheel   120832  1 Jan  2020 pwd
-rwxr-xr-x  1 root  wheel   106000  1 Jan  2020 rm
-rwxr-xr-x  1 root  wheel   104368  1 Jan  2020 rmdir
-rwxr-xr-x  1 root  wheel   120912  1 Jan  2020 sh
-rwxr-xr-x  1 root  wheel   120784  1 Jan  2020 sleep
-rwxr-xr-x  1 root  wheel   138656  1 Jan  2020 stty
-rwxr-xr-x  1 root  wheel   120464  1 Jan  2020 sync
-rwxr-xr-x  1 root  wheel  1104640  1 Jan  2020 tcsh
-rwxr-xr-x  1 root  wheel   121104  1 Jan  2020 test
-rwxr-xr-x  1 root  wheel   106000  1 Jan  2020 unlink
-rwxr-xr-x  1 root  wheel   120752  1 Jan  2020 wait4path
-rwxr-xr-x  1 root  wheel  1331248  1 Jan  2020 zsh
$ 
```


# Compression and deduplication

When `plakar` performs a snapshot,
it **splits every file into content-defined chunks** and **checks if they are available** in the repository before pushing them,
in a way similar to what `rsync` does.

Each chunk is **compressed** before being written to the repository,
so that between compression and deduplication,
**a saved snapshot may take less space than the original directory**:

```sh
$ du -sh /bin
4.6M    /bin

$ du -sh ~/.plakar
3.8M    /Users/gilles/.plakar
```

Furthermore,
since this deduplication takes place globally in the repository,
pushing multiple times the same content doesn't cause the plakar to grow much as **it only grows by the size of the snapshot index**:

```sh
$ plakar push /bin
f2645591-4b06-4512-89e2-dae8ad1e2360: OK

$ plakar push /bin
c0aa514b-452f-4122-a53f-99e984cb0548: OK

$ plakar push /bin
3630358c-c634-4ad9-960d-f864d1eb56f5: OK

$ plakar push /bin
e521e447-5c2a-429f-b637-41561daa3826: OK

$ plakar push /bin
a83fe7bb-341f-4cf1-bad6-c2b1933422f8: OK

$ plakar ls
82711aee-6485-445b-98f4-6cae51c83035 [2021-03-27T22:05:00Z] (size: 10 MB, files: 36, dirs: 1)
f2645591-4b06-4512-89e2-dae8ad1e2360 [2021-03-27T22:06:17Z] (size: 10 MB, files: 36, dirs: 1)
c0aa514b-452f-4122-a53f-99e984cb0548 [2021-03-27T22:06:17Z] (size: 10 MB, files: 36, dirs: 1)
3630358c-c634-4ad9-960d-f864d1eb56f5 [2021-03-27T22:06:18Z] (size: 10 MB, files: 36, dirs: 1)
e521e447-5c2a-429f-b637-41561daa3826 [2021-03-27T22:06:18Z] (size: 10 MB, files: 36, dirs: 1)
a83fe7bb-341f-4cf1-bad6-c2b1933422f8 [2021-03-27T22:06:19Z] (size: 10 MB, files: 36, dirs: 1)

$ du -sh ~/.plakar
3.8M    /Users/gilles/.plakar
```

# Snapshots UUID prefix-based lookup

Because using an UUID as parameter to subcommands is painful,
`plakar` performs **prefix-based lookup** and allows using the beginning of an UUID as long as **it is not ambiguous**:

```sh
$ plakar pull 3
3630358c-c634-4ad9-960d-f864d1eb56f5: OK
```

In case of ambiguity,
it warns so that a less ambiguous prefix can be provided:

```sh
$ plakar ls|grep ^4
45de4fd3-d177-4378-84cb-077b2ef297fb [2021-03-27T23:09:11Z] (size: 10 MB, files: 36, dirs: 1)
4c467893-f959-43cb-858f-c50b8c46a86d [2021-03-27T23:17:05Z] (size: 10 MB, files: 36, dirs: 1)

$ plakar pull 4
2021/03/28 00:17:15 plakar: snapshot ID is ambigous: 4 (matches 2 snapshots)

$ plakar pull 45
45de4fd3-d177-4378-84cb-077b2ef297fb: OK
```

# Snapshots health check

Backups are not useful **if they can't be restored**,
but it's also painful to create a backup and **try restoring just to know if it would work**.
To cope with this,
`plakar` supports health checking snapshots without actually restoring them:

```sh
$ plakar check 3
3630358c-c634-4ad9-960d-f864d1eb56f5: OK
```

What it does is that **it fetches the snapshot index**,
checks that **everything referenced in the snapshot exists** in the repository,
and does **a few sanity checks** to verify if **it could restore the entirety of the snapshot** if requested.


# Snapshots restoration

As shown in the example above,
restoring a snapshot is **as simple as typing**:

```sh
$ plakar pull 3
3630358c-c634-4ad9-960d-f864d1eb56f5: OK
```

But this performs a full snapshot restore,
whereas sometimes what you really want is a **partial restore**.
For this,
`plakar` supports **providing a path within a snapshot** pointing either **to a subdirectory or to a file**:

```sh
$ plakar pull 3:/bin/ls
3630358c-c634-4ad9-960d-f864d1eb56f5:/bin/ls: OK

$ ls -l bin     
total 312
-rwxr-xr-x  1 gilles  staff  157360 27 Mar 23:18 ls
```

While doing the restore,
`plakar` will **check every chunk** and their checksums to **detect any corruption**:

```sh
$ rm -rf ~/.plakar/default/chunks/00/008d4087e3e99c8f8c2c1ad76e9c6bc3d614e580e30ed45da06f93f912995538 

$ plakar pull 3
/bin/ksh: missing chunk 008d4087e3e99c8f8c2c1ad76e9c6bc3d614e580e30ed45da06f93f912995538
/bin/ksh: corrupt file: checksum mismatch
3630358c-c634-4ad9-960d-f864d1eb56f5: KO

$ ls -l bin
total 19504
-rwxr-xr-x  1 gilles  staff   121104 27 Mar 23:22 [
-r-xr-xr-x  1 gilles  staff   743708 27 Mar 23:22 bash
-rwxr-xr-x  1 gilles  staff   121968 27 Mar 23:22 cat
-rwxr-xr-x  1 gilles  staff   107552 27 Mar 23:22 chmod
-rwxr-xr-x  1 gilles  staff   123232 27 Mar 23:22 cp
-rwxr-xr-x  1 gilles  staff  1104640 27 Mar 23:22 csh
-rwxr-xr-x  1 gilles  staff   277408 27 Mar 23:22 dash
-rwxr-xr-x  1 gilles  staff   139264 27 Mar 23:22 date
-rwxr-xr-x  1 gilles  staff   122160 27 Mar 23:22 dd
-rwxr-xr-x  1 gilles  staff   121840 27 Mar 23:22 df
-rwxr-xr-x  1 gilles  staff   120848 27 Mar 23:22 echo
-rwxr-xr-x  1 gilles  staff   205648 27 Mar 23:22 ed
-rwxr-xr-x  1 gilles  staff   121504 27 Mar 23:22 expr
-rwxr-xr-x  1 gilles  staff   120864 27 Mar 23:22 hostname
-rwxr-xr-x  1 gilles  staff   121232 27 Mar 23:22 kill
-r-xr-xr-x  1 gilles  staff  1258618 27 Mar 23:22 ksh
-rwxr-xr-x  1 gilles  staff   329344 27 Mar 23:22 launchctl
-rwxr-xr-x  1 gilles  staff   105136 27 Mar 23:22 link
-rwxr-xr-x  1 gilles  staff   105136 27 Mar 23:22 ln
-rwxr-xr-x  1 gilles  staff   157360 27 Mar 23:22 ls
-rwxr-xr-x  1 gilles  staff   104752 27 Mar 23:22 mkdir
-rwxr-xr-x  1 gilles  staff   106176 27 Mar 23:22 mv
-rwxr-xr-x  1 gilles  staff   291152 27 Mar 23:22 pax
-rwsr-xr-x  1 gilles  staff   173568 27 Mar 23:22 ps
-rwxr-xr-x  1 gilles  staff   120832 27 Mar 23:22 pwd
-rwxr-xr-x  1 gilles  staff   106000 27 Mar 23:22 rm
-rwxr-xr-x  1 gilles  staff   104368 27 Mar 23:22 rmdir
-rwxr-xr-x  1 gilles  staff   120912 27 Mar 23:22 sh
-rwxr-xr-x  1 gilles  staff   120784 27 Mar 23:22 sleep
-rwxr-xr-x  1 gilles  staff   138656 27 Mar 23:22 stty
-rwxr-xr-x  1 gilles  staff   120464 27 Mar 23:22 sync
-rwxr-xr-x  1 gilles  staff  1104640 27 Mar 23:22 tcsh
-rwxr-xr-x  1 gilles  staff   121104 27 Mar 23:22 test
-rwxr-xr-x  1 gilles  staff   106000 27 Mar 23:22 unlink
-rwxr-xr-x  1 gilles  staff   120752 27 Mar 23:22 wait4path
-rwxr-xr-x  1 gilles  staff  1331248 27 Mar 23:22 zsh
```

The snapshot is still restored as a *"best effort"*,
but here a warning informs that `ksh` is **not the same as the one that was initially backed up**.
This is **not supposed to happen in practice** as long as I don't fiddle with the repository,
but it is a nice sanity check.

One that **would be caught** by `plakar check`:

```sh
$ plakar check 3
3630358c-c634-4ad9-960d-f864d1eb56f5: KO
```

# Other `plakar` subcommands

In addition to `pull`, `push`, `ls` and `check`,
`plakar` supports various other subcommands.

They are more interesting when playing with text files,
so let's push `/private/etc`:

```sh
$ plakar push /private/etc
open /private/etc/krb5.keytab: permission denied
open /private/etc/aliases.db: permission denied
open /private/etc/racoon/psk.txt: permission denied
open /private/etc/security/audit_user: permission denied
open /private/etc/security/audit_control: permission denied
open /private/etc/sudoers: permission denied
open /private/etc/sudo_lecture: permission denied
open /private/etc/master.passwd: permission denied
open /private/etc/openldap/slapd.conf.default: permission denied
open /private/etc/openldap/DB_CONFIG.example: permission denied
98b6658b-a975-47c4-a38f-f958d8d7359f: OK
```

Here `plakar` has warned about **files that couldn't make it into the snapshot**,
but lets inspect **what files are part of the snapshot** with `ls`:

```sh
$ plakar ls 9:/
acc57694b78a1ea669535547ac310e7682f141b75f6ba23658078b9000dbd9ac -rw-r--r--     root    wheel    20 kB /private/etc/php-fpm.d/www.conf.default
a4dda57401575878b78c5b0bc4a6bc020675d7f285dc8a836d07f1fda0938715 -rw-r--r--     root    wheel    12 kB /private/etc/postfix/LICENSE
863f779b43680f81799688e91c18a164047a1a8e9dfff650881e50ba35530ed1 -r--r--r--     root    wheel    141 B /private/etc/uucp/port
75a3d05049424cc17623d04c5d84038ca703ae673ad4c1bcf62d8293462d90b0 -rw-r--r--     root    wheel    152 B /private/etc/pam.d/login.term
591161270b2fbe74d7f1b96625c4b60551f33eb6fa28992c8ccc71dff9b7c70b -rw-r--r--     root    wheel   6.2 kB /private/etc/postfix/master.cf.proto
056514aa4105c96e0aad6a144d76f63461db7157123d43fa7c4121297a6c9200 -rw-r--r--     root    wheel    190 B /private/etc/asl/com.apple.authd
05f4215652c68201fd6aee81fc4eeaee4d5c395f3f4a3a155b986cb06ca0f768 -rw-r--r--     root    wheel    216 B /private/etc/asl/com.apple.cdscheduler
3c4aaff1783756368c28022658ebb238d6f9f32beeb0d04d79962654e0a0ae26 -rw-r--r--     root    wheel   3.3 kB /private/etc/ssh/sshd_config
23198c58755cd991531259cd290f62ea99cd9596595001ab303a15306fd03ec1 -rw-r--r--     root    wheel    127 B /private/etc/pam.d/authorization_ctk
48edf5460c6ad64cc681209902d2cfef856f4b68400e34cac41ec6704f45b580 -rwxr-xr-x     root    wheel    687 B /private/etc/periodic/daily/430.status-rwho
c998ee1b26049571ed20dc89e7dd9d3e87e85846bdf0d9ff1aae9f760774bf5b -rw-r--r--     root    wheel   1.1 kB /private/etc/apache2/original/extra/httpd-info.conf
b822424c1eba4ae12efa0bad98c6288f9111d5da88f43a847c69e660dc3c8c80 -r--r--r--     root    wheel   6.2 kB /private/etc/openldap/schema/collective.schema
c90d596b58cb588b3ae77312f9fe8b0511ea8a842caf058d130138a32cca8419 -r--r--r--     root    wheel   2.1 kB /private/etc/openldap/schema/fmserver.schema
5475edebf371c8c4771d6951a51a80e4c40871a01355122cac4146966d6aa58c -rw-r--r--     root    wheel   4.5 kB /private/etc/apache2/extra/httpd-mpm.conf
61b95be8351545f888956f733db55f810e3b5f8b4b221ad219428b95c55bdbc9 -rw-r--r--     root    wheel   1.1 kB /private/etc/asl.conf
b9611702dbb3c2d02e060d82cff7a542f6ec6de29aabaa0b4c8482e5ed1f78d0 -rw-r--r--     root    wheel    113 B /private/etc/pam.d/authorization_aks
9a31f3b43190281ce1320ac62b3d7672c0bd5bb8bf607c82767994bfa815068e -rw-r--r--     root    wheel    153 B /private/etc/asl/com.apple.eventmonitor
cc58ac4627390ef04037b7b77ed0359b267a8e8ceb99bdc3aa156dae6367094a -rw-r--r--     root    wheel    607 B /private/etc/apache2/original/extra/httpd-userdir.conf
17294d602f2d28944e6517a6a8a432548351d1eaf468062b8da6d84bbf7c5440 -r--r--r--     root    wheel    133 B /private/etc/uucp/passwd
27b21d21df689f2f097f56b907d2a936acbeb943fc98bfeb56d2c62cba33c451 -rw-r--r--     root    wheel     13 B /private/etc/paths.d/40-XQuartz
1f07bc400e932a1f36c65634b9f5e7d7a6249c58195f903fe21d4bf83df43c5b -rw-r--r--     root    wheel   3.4 kB /private/etc/wfs/httpd_webdavsharing_template.conf
444c716ac2ccd9e1e3347858cb08a00d2ea38e8c12fdc5798380dc261e32e9ef -r--r--r--     root    wheel    265 B /private/etc/bashrc
fa115d33bdc964acfe0b0241c511c2bed643820f8c574e45f443a17a331cf138 -rw-r--r--     root    wheel    318 B /private/etc/pam.d/screensaver_ctk
eab8fa9f3d43e099731db74d17733fb46c5578206f4dbb0db204bc2cef68664a -rw-r--r--     root    wheel   7.6 kB /private/etc/passwd
99c7c05d6e8690b32ea58d6fcda64a090eaae782026551d2d9e347694c626ce7 -r--r--r--     root    wheel   7.8 kB /private/etc/openldap/schema/nis.schema
025f5e31cd7b2248a0a661d8d346452c31d98e280f3420c3f6017c74c829ae73 -rw-r--r--     root    wheel    745 B /private/etc/ssl/openssl.cnf
9d70873703af389018ccc7b57a503d2692ba1d6b71271bcf00473d40f5095486 -rw-r--r--     root    wheel   3.2 kB /private/etc/apache2/original/extra/proxy-html.conf
69c9044d7bdcdd195249b13b8893abf7c60092391975408468ac5e8969fcd79a -rw-r--r--     root      _lp   6.5 kB /private/etc/cups/cupsd.conf
d7c000b62ab236b5c5c4db8ad7edec232b2afe95d85c8e064a5c9f6d5308620f -rw-r--r--     root    wheel   1.6 kB /private/etc/postfix/TLS_LICENSE
ed9d05e8ec15f676263a184a7c30858f2d5cd0258da168d3b262b76567bc7bae -rw-r--r--     root    wheel   2.9 kB /private/etc/apache2/original/extra/httpd-default.conf
bb3d6775b7fc8042b2b68232384694ac3bee6d1100b87f9dcd0840ee935dd2b5 -rw-r--r--     root    wheel    12 kB /private/etc/postfix/canonical
96618e0c2ba27c318f77262c22a99f039ecf2e1d83754fcb1a0f208e961da2f7 -r--r--r--     root    wheel   1.5 kB /private/etc/openldap/schema/openldap.schema
cf6e6fe1497d5b3983529174674b8a09941d67244727157e208d37b24b247bc1 -rw-r--r--     root    wheel    166 B /private/etc/pam.d/authorization_la
aa464696f1c0282b1728b79e5da52c6f44d130b93c6e3011ea4e67407461c191 -rw-r--r--     root    wheel   3.6 kB /private/etc/racoon/racoon.conf
1dc9a5dec35592b043715e6b5a1796df15540ebfe97b6f25fb4960183655eec9 -rw-r--r--     root    wheel   9.3 kB /private/etc/zshrc_Apple_Terminal
e2dd2ce55d57c625d241b8ddbc834fbab3e9c2061955b094326201dfd083d051 -rw-r--r--     root    wheel    202 B /private/etc/asl/com.apple.clouddocs
eb33ba8357c7166620b5c33fc8436a0081ba093f7cbf60fef7ddb35b491a4012 -rw-r--r--     root      _lp    128 B /private/etc/cups/snmp.conf
d7cbf00c6fb63b14e2314cb5aff696e6d01e6911e0bcebf2c540ddb28b614340 -r--r--r--     root    wheel   6.8 kB /private/etc/openldap/schema/nis.ldif
91d53776cbd13b565b93dfae32ceb7236c09df3f4629329b521fe1b523460613 -r--r--r--     root    wheel   3.3 kB /private/etc/openldap/schema/dyngroup.ldif
4a054f138fbed4a755b351fb51dd83fe3005e07fc7d075c3880e10ec8950873a -r--r--r--     root    wheel   123 kB /private/etc/openldap/schema/microsoft.schema
8e920f9fb045a6dfe53fd18bb6450c5b3051bd88c4c7eac88b7c41deee02e6a8 -r--r--r--     root    wheel    246 B /private/etc/pam.d/sudo
9d5aa72f807091b481820d12e693093293ba33c73854909ad7b0fb192c2db193 -rw-r--r--     root    wheel    189 B /private/etc/shells
81bac019ddc3523e67726f6159c180bd128b9ef1872d5359fe40e3fa78690694 -rw-r--r--     root    wheel    387 B /private/etc/asl/com.apple.AddressBookLegacy
309c09ccf0f826e2e753472a01fa32d7f1f326879992e8e925ef3ced57a6252b -rw-r--r--     root    wheel     11 B /private/etc/nanorc
a27af5a5be37a725abe8587a803e601e243739c2ea4abfde9707199b66369deb -rw-r--r--     root    wheel   1.3 kB /private/etc/newsyslog.conf
810dbd036a97217460ec5f56e543621dd2a735bb809fca45327400e2afc272ad -rw-r--r--     root    wheel    27 kB /private/etc/postfix/main.cf.proto
8aa44a4856b41d4f1dd7799b975eb68472df1d6b8f8268a0cc6aaa9975f55349 -rw-r--r--     root    wheel    891 B /private/etc/rtadvd.conf
49422ce34f9e3b5134cecdb73f5f54fa34480fc424279321dab18240b4bf8cde -r--r--r--     root    wheel   2.4 kB /private/etc/openldap/schema/misc.schema
eff9ba3a711059ac519e1914477a28db07ce3ed8c70eb36479e0da39cd41fc4e -rw-r--r--     root    wheel    197 B /private/etc/pam.d/passwd
e523e826b0192c39b42d71597caa8b3e5e166d1e6af98528f046f1ebe6727925 -rw-r--r--     root    wheel    24 kB /private/etc/postfix/header_checks
f63a90bd20d0baea6f3b7f5c0679d02e58dc294d5a37bd0c44f2046609d74b4a -r--r--r--     root    wheel    16 kB /private/etc/snmp/snmpd.conf.default
c7e32a8ad7f390bc508c245dfd929cccf18cf222889ca7eed281619495b3dc5f -rw-r--r--     root    wheel   346 kB /private/etc/ssl/cert.pem
49fde98b0963f27c0630072414be0ee0205d71eba529bbaf86b65d1e1603eead -rw-r--r--     root    wheel   2.9 kB /private/etc/apache2/extra/httpd-autoindex.conf
f918b0dec7a783521bd7efbe2e5aef28e2d3250bec1cb524752bb31d18ddbe0b -rw-r--r--     root    wheel    149 B /private/etc/asl/com.apple.performance
b2dc9fa1938c981ce201f1458afb373d97caf860d9bfba0ded9c450a64f8e65d -rw-r--r--     root    wheel     27 B /private/etc/ntp.conf
9e8dffed375e348782c5bc8e15b6a33ea002c96588b7686bd230693dd86898eb -rw-r--r--     root    wheel    181 B /private/etc/pam.d/checkpw
8a8a65b24957cbf2c5376a2a7a4dccb3e02f8cc3cfb338caa58cb93345c6a23c -r--r--r--     root    wheel    203 B /private/etc/pam.d/cups
c74b98809cf175d83699a095bf9f20689dcc66daeedc7e48119026e88103fb09 -rw-r--r--     root    wheel     44 B /private/etc/postfix/custom_header_checks
891232253c3bb2789de7cb01b8d313e3c3ed6820434457634775867cc50c657b -rw-r--r--     root    wheel   5.4 kB /private/etc/postfix/makedefs.out
ba4bb2c9087c046186ef306d4bd5046694e7bc0bc715503ba210e6f96e417642 -rw-r--r--     root    wheel    13 kB /private/etc/apache2/original/extra/httpd-ssl.conf
4348468f1850b16e8ad3f7be8963bde559cfd8dd7aed8ffd655a40aaa551e073 -rw-r--r--     root    wheel    156 B /private/etc/asl/com.apple.MessageTracer
b2aac03248e8f229c703561f5bb059f9be491e5db9f447692398d775a9fb12a5 -rw-r--r--     root    wheel    195 B /private/etc/auto_master
bf41a1579c3d12a4b8466fad91794157fcca4b2c611fb178ddce232cd9a4e0e5 -r--r--r--     root    wheel   8.1 kB /private/etc/openldap/schema/corba.schema
c78f581bf6c453c6cf4e4ba241fe081d7de69bbfa42d41fc65a475827c8bd627 -rwxr-xr-x     root    wheel    695 B /private/etc/periodic/daily/130.clean-msgs
7e94a499c84675258a9734b150a28950fa22cbaad5f1eedfff507541d9f7d9ef -rw-r--r--     root    wheel    13 kB /private/etc/apache2/magic
66dfdc46b6b66f0830252f20e1c74aa865656223e30f95fbef6b05051fdc3cde -rw-r--r--     root    wheel   9.3 kB /private/etc/bashrc_Apple_Terminal
f19f881084f599fa261243918d922373eab14623e78d23c41fcc031aa21ca7b6 -rw-r--r--     root    wheel    941 B /private/etc/emond.d/emond.plist
b19baf7d26dad9163a2e87c5b9731943e8dad7243c81aacfacf6bdcdcae03c61 -rw-r--r--     root    wheel    200 B /private/etc/pam.d/chkpasswd
effcfce27485ef1ca9142df4fe810d518e720092b00dce0a9504e2ef378cc42c -rw-r--r--     root    wheel    27 kB /private/etc/postfix/main.cf
d73737159f9e3bedf75645dd7c2254331beea55567da92f3b974d4ba8194ff81 -rw-r--r--     root    wheel    13 kB /private/etc/postfix/virtual
1b3f45ae268583561d2e9baa82474dd0035d974cf687991a20f885f107dd943d -r--r--r--     root    wheel   2.0 kB /private/etc/openldap/schema/collective.ldif
f99a4803d9f2f8c48c33866487067fbd3a0d02080e10749e70445813b4d60954 -rwxr-xr-x     root    wheel   1.6 kB /private/etc/periodic/daily/110.clean-tmps
7ac5924452faaf32bbfbd41f816947fccc3ff2fc17554be7bc5fff99d71ebe2e -rw-r--r--     root    wheel   6.9 kB /private/etc/postfix/relocated
7cb50dc544e9050e398ebc0844081e928ca52c09a0d304228489962a93b63692 -rw-r--r--     root    wheel    365 B /private/etc/hosts
fde802d853379ae3724693f05be7f273b92429ab500a7806683e7a21975f743e -r--r--r--     root    wheel   3.0 kB /private/etc/openldap/schema/java.ldif
dbec41507d3e3c390b0418d96c9ea5f61d3e9ac969bd025b6d23b78af25adaeb -rw-r--r--     root    wheel     18 B /private/etc/paths.d/go
2012882e055c2a2502cd7d3dd373c1804f0fce782225e90aca491abad720b53a -rw-r--r--     root    wheel   1.5 kB /private/etc/apache2/original/extra/httpd-vhosts.conf
dd89ec314b4e26aa2481a315c7e71404ccff929ead511269e934406be0e61143 -rwxr-xr-x     root    wheel    888 B /private/etc/periodic/monthly/199.rotate-fax
3efeec1e339725bfadd53138dde7822ebc57ba73275a1c0fc05e0d7d8abafc4e -rw-r--r--     root    wheel    624 B /private/etc/wfs/httpd_webdavsharing_proxy.conf.inactive
ede47b49ba30fbd0e02f10006fa967d2887f7e4c676e6b657ae0a20285ef1d4c -rw-r--r--     root    wheel   2.2 kB /private/etc/apache2/extra/httpd-multilang-errordoc.conf
ebbeeaa6c956e56727c1391e4443a7da9da70e5e9201fcf69194e9ce397f9a2d -rw-r--r--     root    wheel    339 B /private/etc/asl/com.apple.coreduetd
2a0ba79339b7112dbd24b5fdd4b54f32573971a0ad9906240bbeb6bad3d382bc -rw-r--r--     root    wheel   5.3 kB /private/etc/rc.netboot
6fb5b260918922ca5ca4dfb296967d8edb9c12f2c043f26b64590758441d682d -rw-r--r--     root    wheel   1.0 kB /private/etc/pf.conf
011fd8d7180df2a60de23826192aab50889f73488d01d6c35d186fcb7f64640d -rw-r--r--     root    wheel    61 kB /private/etc/apache2/mime.types
7c1f46f8dca762135990bdf0e3aa0395d4867858e35a3f2583549fa6cd5b082a -rw-r--r--     root    wheel    189 B /private/etc/csh.cshrc
cdfc5a48233b2f44bc18da0cf5e26df47e9424820793d53886aa175dfbca7896 -rw-r--r--     root    wheel     45 B /private/etc/paths
64d9fef3a2825ba4d770b38b5ccd12ad4df477535f345fa8649d65fea940fcbf -rw-r--r--     root    wheel    512 B /private/etc/pam.d/login
38b6d46f6924364dc3013ffad83a12b7bc10bf3a878b5ae0f53ef117851900b8 -rw-r--r--     root    wheel   1.7 kB /private/etc/apache2/extra/httpd-dav.conf
ba4bb2c9087c046186ef306d4bd5046694e7bc0bc715503ba210e6f96e417642 -rw-r--r--     root    wheel    13 kB /private/etc/apache2/extra/httpd-ssl.conf
ede47b49ba30fbd0e02f10006fa967d2887f7e4c676e6b657ae0a20285ef1d4c -rw-r--r--     root    wheel   2.2 kB /private/etc/apache2/original/extra/httpd-multilang-errordoc.conf
5475edebf371c8c4771d6951a51a80e4c40871a01355122cac4146966d6aa58c -rw-r--r--     root    wheel   4.5 kB /private/etc/apache2/original/extra/httpd-mpm.conf
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 -rw-r--r--     root    wheel      0 B /private/etc/hosts.equiv
e05e1b6f966477644849720be93b0707029323b732b4105b63475443fa740aa2 -rw-r--r--     root    wheel    376 B /private/etc/asl/com.apple.networking.boringssl
39bc7222686b37aea251957d5b0658600711bc91815a792459395188ace48e33 -rw-r--r--     root    wheel     23 B /private/etc/ntp_opendirectory.conf
ad2902787f6a2cc2728320d9bf5387371d88a8226c32bac08df9f6d7886bfee4 -r--r--r--     root    wheel    20 kB /private/etc/openldap/schema/ppolicy.schema
69a61ae495580e0e144792c9a809bffe707a6a13f292095f43a7a88c92b11ef0 -rwxr-xr-x     root    wheel   1.2 kB /private/etc/periodic/daily/310.accounting
49fde98b0963f27c0630072414be0ee0205d71eba529bbaf86b65d1e1603eead -rw-r--r--     root    wheel   2.9 kB /private/etc/apache2/original/extra/httpd-autoindex.conf
f1e89e3a029a3b72a30f82fc7346c13030b4de16b77ecc4c58f4a4de6f99c242 -rw-r--r--     root    wheel     82 B /private/etc/com.apple.screensharing.agent.launchd
a18ff587cf7d16939acf5a969da49a393ce503b1c4b83ad42e4b902991329b4f -r--r--r--     root    wheel    48 kB /private/etc/openldap/schema/apple.schema
6293f4cc430a3d8648fa60f26546d16e55f84d05b71f996ca930a95aa3ce312c -r--r--r--     root    wheel   3.3 kB /private/etc/openldap/schema/dyngroup.schema
c0dbae0bb16ea7cbc26c0d49286946ef8713335564f5c0a2f4ab7339b8f948f2 -r--r--r--     root    wheel   2.1 kB /private/etc/openldap/schema/misc.ldif
fc1166d771ec0092cb6280cadf54e9df3fa4eff6be0132f9064f87a7b0b6416a -r--r--r--     root    wheel   3.3 kB /private/etc/openldap/schema/openldap.ldif
b232f6b78ffdcfdf34697f1432b6cd9bcbad27da384d493d9285b02f398c13ea -rw-r--r--     root    wheel   1.7 kB /private/etc/rpc
9a40f8365adb49f0a63930316c475251631150f0f58bfad692a633c8e7486fc9 -rw-r--r--     root    wheel     96 B /private/etc/syslog.conf
b99ca2f2de327fd9a0374e1eacd3ccbc0f6fce17e0372c42d370ce4852bb75f4 -rw-r--r--     root    wheel    22 kB /private/etc/apache2/original/httpd.conf
b45fd6ac093bcb4aec44b366d14300cdb1a65f65d7a630630af0f89c9c937894 -rw-r--r--     root    wheel    265 B /private/etc/openldap/ldap.conf
36ff03121c003e884d67372d060c83886d73429fc62df37c831f604ecb59cb56 -r--r--r--     root    wheel    717 B /private/etc/openldap/schema/apple_auxillary.schema
b01b61f6b1486186b84ac009ccfff409fc43dd66e1b780a60de94090e8459dc2 -rw-r--r--     root    wheel    154 B /private/etc/newsyslog.d/wifi.conf
334c18bfe6d007a0bcb6f1e7b3619b9592acea6ce60775d6f8d3d56f7ca8d50b -r--r--r--     root    wheel    72 kB /private/etc/php.ini.default
1b3a1b45deb322db6df7d39dd52f67d1d49cb89e77388c41dccf640dcd83cd37 -rw-r--r--     root    wheel    10 kB /private/etc/postfix/aliases
8923cf7346a5536cc5fadf73e1f92681cf31e73d06548f8abd1e7854f5694290 -rw-r--r--     root    wheel    351 B /private/etc/notify.conf
a377ed0d15192ea50c9de076583030fe246bec85bd37a9fe1f3cd792e84231ab -r--r--r--     root    wheel    20 kB /private/etc/openldap/schema/pmi.schema
fbbd9eac3bb65ba340cda731e596fc52e76c49339e7c7dd6653f06d17edb36ad -r--r--r--     root    wheel    527 B /private/etc/pam.d/sshd
aab9982ea2af8b86b1fde1e8c199de6f12f45db1a831c1804c33ee65e53d6d25 -rw-r--r--     root    wheel    264 B /private/etc/pam.d/authorization
d0f8d2287d44031b79d9798dc6c68ea7d789ffc129ec2056b3630c938cc54068 -rw-r--r--     root    wheel   3.3 kB /private/etc/wfs/httpd_webdavsharing.conf
405101681e712d2727fd6131a84b0eb1b89ed3b72ad3722916368b5ea3cdf6f6 -rw-r--r--     root    wheel    943 B /private/etc/wfs/wfs.plist
be95a05eafeae5a7450eef61d6477d0b1080f314c3338b5f81c5d5fc65681fdd -r--r--r--     root    wheel   1.3 kB /private/etc/irbrc
f33710cbbb38b977b7ed1a10038f6f1911d0ce8dd82f65e1a54db13fbecc429f -r--r--r--     root    wheel    12 kB /private/etc/openldap/schema/cosine.ldif
422bfa0619b344ec6248e589333df261c4901feca2ce816fd23ef99b5229b7a0 -r--r--r--     root    wheel   5.7 kB /private/etc/openldap/schema/samba.schema
950f16a5623699af1fa0e03f9f286374229bbc84d7e887efa3984a1902e48d4b -rw-r--r--     root    wheel    175 B /private/etc/newsyslog.d/com.apple.slapconfig.conf
994263a50efad8a1b416754ea4ed463d5e1a7524ed2b3105b780dfe480dbfe94 -r--r--r--     root    wheel    145 B /private/etc/php-NOTICE-PLANNED-REMOVAL.txt
97a0a5a23df5f84d6c4396e5e1c4c6b10c690300ee392cfbf633002e300ce2cb -rw-r--r--     root    wheel    10 kB /private/etc/postfix/generic
b86accf42ae92d4e00b67c50010b7f278bc76c50755bdc9638e2096cbd596890 -rw-r--r--     root    wheel   6.4 kB /private/etc/protocols
24240ece7cf8aa30d242f178d7defd3a47d631bbbcf897d2f170e28b863564fa -rw-r--r--     root    wheel    178 B /private/etc/asl/com.apple.mail
71be406f45c35429621a43970b0a10debcfd319f0e0ecb1f9b5c5c0ce4cfc3bd -rw-r--r--     root    wheel    121 B /private/etc/csh.login
2e5298d1432df0d5f3b741a1e052cfdb73cc4d800db34651f083f36b3c64dcb8 -r--r--r--     root    wheel    10 kB /private/etc/openldap/schema/duaconf.schema
36816c4a31b3fbbc86312073f65ba5b514823f89864ad2f3ede7b6b456a8ed4d -r--r--r--     root    wheel    27 kB /private/etc/security/audit_event
d4a58e12a2e3a9aa4ca72cbe4c63b786b20d6a212a87759f6c93db594be3e3f0 -r-xr-xr-x     root    wheel   1.3 kB /private/etc/security/audit_warn
a744a313d0b95f6f15768b78d15cf3168ffd18abde4e4801a55b0fbb1c37ae86 -rw-r--r--     root    wheel    233 B /private/etc/asl/com.apple.install
a1b83027e0b929e389bde2984078b7debf7f885051d9f9be18545aa07bebac21 -rw-r--r--     root    wheel    409 B /private/etc/asl/com.apple.mkb.internal
533ad90f9c16d3d000a929513d72050bdef7e75bc8ea1790694227c20ab97f2e -rw-r--r--     root    wheel    176 B /private/etc/newsyslog.d/com.apple.xscertd.conf
b17912cf8ac845d1098829a3f876bdc7597bc8360bb8f260e49d1e3ca26d5029 -rw-r--r--     root    wheel   1.4 kB /private/etc/apache2/extra/httpd-manual.conf
28048538f6a15661bdefcf706f39d0003fb9107bdc5206df295db67b66b8345a -rw-r--r--     root    wheel    365 B /private/etc/pam.d/screensaver_la
16c13e23b179e6b325102eb285e78b7678ee64231f9bbb2ff6bd5311f38f89ae -rw-r--r--     root    wheel   7.4 kB /private/etc/postfix/master.cf
61b42416ea3c5c9d5850364a23ec3aa5b6ddc6e2055b0402d6435b8fed46f3c5 -r--r--r--     root    wheel    652 B /private/etc/security/audit_class
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 -rw-r--r--     root    wheel      0 B /private/etc/xtab
fb5827cb4712b7e7932d438067ec4852c8955a9ff0f55e282473684623ebdfa1 -r--r--r--     root    wheel   3.1 kB /private/etc/zshrc
1aac36c9a80ecab24b5d4346ac5c605a9614f89590e6d0957b82f781a8d40a49 -rwxr-xr-x     root    wheel   1.1 kB /private/etc/periodic/daily/140.clean-rwho
ca2ae7cf01205f3b961c70c607a31c5ed7cc4434dc94d5a5152e18844c5ecffe -rwxr-xr-x     root    wheel    378 B /private/etc/periodic/daily/199.clean-fax
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 -rw-r--r--     root    wheel      0 B /private/etc/rmtab
e02803e7d3435412c397478205b16c95486fde05ecde74959f9799c49c7ce009 -rw-r--r--     root    wheel    149 B /private/etc/auto_home
bd0ec71eab6515a902703163e65839498429e7d31860c9584b30b635f7423367 -r--r--r--     root    wheel    14 kB /private/etc/openldap/schema/java.schema
11eddeb0e0d55ea1bf43c180f04a53533d5cc71d2a69fbbcf7ca0488c8450d79 -rwxr-xr-x     root    wheel     23 B /private/etc/paths.d/100-rvictl
20909c75c14c9f5360a48c889d06a0d6cfbfa28080348940fc077761744f2aa5 -rw-r--r--     root    wheel    822 B /private/etc/emond.d/rules/SampleRules.plist
11ae0d388aed5d193821d33d060da5ed9119d19689ce6c220b407a4c5da3553f -rwxr-xr-x     root    wheel   1.0 kB /private/etc/periodic/monthly/200.accounting
8c6e2a2647ee854f469a3bb798e02ba5a8b1812cab229ff129f073e7a80c1202 -rw-r--r--     root    wheel   678 kB /private/etc/services
063a5972c7b72eda797684b2c034f37a000aaf7bd3b91b01f7cdd3ff74f170e7 -rw-r--r--     root    wheel   5.7 kB /private/etc/gettytab
b45fd6ac093bcb4aec44b366d14300cdb1a65f65d7a630630af0f89c9c937894 -rw-r--r--     root    wheel    265 B /private/etc/openldap/ldap.conf.default
bbe18692eb80dc6e27643a5392b358be548edaec210206eeb1a67c99892fc33e -rw-r--r--     root    wheel     39 B /private/etc/csh.logout
7de66c7adb93cb1e0da88874d27668c1d78488a9578268e8234460d9083b2e01 -rw-r--r--     root      _lp   6.5 kB /private/etc/cups/cupsd.conf.default
ee479a7a0dd839c4a08e73d5a6fcffbe750bd47b82cf7c633ed20c5e63b8ed03 -rw-r--r--     root    wheel   5.0 kB /private/etc/defaults/periodic.conf
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 -rw-r--r--     root    wheel      0 B /private/etc/kern_loader.conf
7bf0e7399139a3a478b9f447ceb042dee1137261b39dace27db18a93242ebdc2 -r--r--r--     root    wheel   1.8 kB /private/etc/openldap/schema/corba.ldif
c536e94effbd1e5890f6fb084590a091afc8369a65a9a8b3d71edbfde290da45 -r--r--r--     root    wheel    20 kB /private/etc/openldap/schema/core.schema
aea837b88e320abcd476f5839952d5e1a612c0e4a225cb9205d20071e6c37745 -rw-r--r--     root    wheel    106 B /private/etc/mail.rc
b14dbb949b10a0635b7303ff96d86406d8d6c64d414a1e0f91cdf8ba2a62c74a -r--r--r--     root    wheel    191 B /private/etc/pam.d/other
9d6f5bffb6fda39e61e1dd90109a42b7a1abf8a5995f3e81e87b7bf4f312b309 -rw-r--r--     root    wheel    119 B /private/etc/ftpusers
0cb5352ac33727fd7979e454f6ac1a56b7795a8ab5a25d7cc955133fb47cf9c4 -rwxr-xr-x     root    wheel    522 B /private/etc/periodic/daily/400.status-disks
16c13e23b179e6b325102eb285e78b7678ee64231f9bbb2ff6bd5311f38f89ae -rw-r--r--     root    wheel   7.4 kB /private/etc/postfix/master.cf.default
d9013690a0652573167ef63917f6ed2ad63a5e15b1f82d40010c75f33ec85da1 -r--r--r--     root    wheel    205 B /private/etc/apache2/other/mpm.conf
14ecbb4a93f277771924c3a9aa870ecd67a742fd6164572af329feef40061325 -rw-r--r--     root    wheel    318 B /private/etc/asl/com.apple.login.guest
b22a5ff409482104f804f04b0035f2fd1f6801974645f79c87e9025729a50286 -rw-r--r--     root      _lp   3.1 kB /private/etc/cups/cups-files.conf.default
18dc169426b3130e43cd1a5919f95dd58ccee6d49a9182a4b2ecdf5c5bb563bc -rw-r--r--     root    wheel   577 kB /private/etc/ssh/moduli
c64beac4afa42e602286728c1e0158fc90195768e97d5f6bc0d917208faeb3b3 -r--r--r--     root    wheel    21 kB /private/etc/openldap/schema/core.ldif
8c716d8131c6f1ae290b49beed057f06bd5cd18b5227cfe3b50556b35e21ce3b -rw-r--r--     root    wheel   5.3 kB /private/etc/php-fpm.conf.default
a3fe9f414586c0d3cacbe3b6920a09d8718e503bca22e23fef882203bf765065 -r--r--r--     root    wheel    189 B /private/etc/profile
f63a90bd20d0baea6f3b7f5c0679d02e58dc294d5a37bd0c44f2046609d74b4a -rw-r--r--     root    wheel    16 kB /private/etc/snmp/snmpd.conf
6e72e6f9366198e19491ccc785a0cdfb5a05510609c03fcf388b02aac43fec0b -rw-r--r--     root    wheel   1.0 kB /private/etc/ssl/x509v3.cnf
b86ff58053e9e930a0324f7fb5e0458213567feca9e97de92da20bff82f17e06 -r--r--r--     root    wheel   2.4 kB /private/etc/uucp/sys
146f0e625f4a1b8b11fabab45b66d989e85d0408348403678074b3db881e3587 -rw-r--r--     root    wheel    165 B /private/etc/newsyslog.d/com.apple.slapd.conf
726813f6f77f7b566a7c004d79cac51dfdc3b28ff3265179e7fbb032b706ba6d -r--r--r--     root    wheel   4.0 kB /private/etc/openldap/schema/ppolicy.ldif
8475720c1288108fa6c55c028a26e7aa12155009095dd8c04a547969b65a95b3 -rwxr-xr-x     root    wheel    548 B /private/etc/periodic/daily/420.status-network
abb7d93ea2ef352db65c05101a112264334e7e531522f576ee397371e13fe5d8 -rw-r--r--     root    wheel    183 B /private/etc/pam.d/authorization_lacont
4ae766c32277f60bba1b505e3837ccc29587c0cb4fbf65d20b17baa9463b0fb6 -rw-r--r--     root    wheel    13 kB /private/etc/postfix/transport
a9419086fc2b70f69130c3ee9f8761b0a12b0c7da47d3b779b04ab3827081cf0 -rw-r--r--     root    wheel   5.1 kB /private/etc/apache2/extra/httpd-languages.conf
e8c63560d75c0c459666f2d8c69bd23b83151080d571148b9a30667f680d6f3c -rw-r--r--     root    wheel   3.2 kB /private/etc/group
7ca5887133958e0ac30c92ecec70fe753977cfcc0137cfa93311aeed8fa38e24 -rw-r--r--     root    wheel   117 kB /private/etc/openldap/AppleOpenLDAP.plist
4277bb97ba7b51577a0d38151d3e08b40bdf946753f5b5bdeb814d6ff57a8a5e -rw-r--r--     root    wheel    515 B /private/etc/afpovertcp.cfg
f047c7a23d830f221f142861d2ae10eb4a27d74beff3fcaf599b4c441ae44f5e -r--r--r--     root    wheel    13 kB /private/etc/openldap/schema/microsoft.std.schema
94139c5762d820ac8a6d567ca5a5e9753af95e1c8560f23e65bc169be11f4042 -rwxr-xr-x     root    wheel    712 B /private/etc/periodic/daily/999.local
69f2f70c4d02fe6a0d4ecc99b8cfb676090aae0435c74eb45b2d79c72cb39a09 -rwxr-xr-x     root    wheel    606 B /private/etc/periodic/monthly/999.local
b99ca2f2de327fd9a0374e1eacd3ccbc0f6fce17e0372c42d370ce4852bb75f4 -rw-r--r--     root    wheel    22 kB /private/etc/apache2/httpd.conf
db16889d677588787af70881d7e0af827e847a5660e9a78c955da53d8edd614b -rw-r--r--     root    wheel    105 B /private/etc/asl/com.apple.iokit.power
bea9ba8f43b879adff12ba844aad204ae6be074b356a4ff917305725b4341eb5 -r--r--r--     root    wheel    154 B /private/etc/newsyslog.d/files.conf
0d5d7c3f51196286c17c894bf77ab0ba057256aa04a455a137c3860fbb46d98b -r--r--r--     root    wheel   3.5 kB /private/etc/openldap/schema/README
6c2621756bae333c3af6429407f949f6220bd75df956606ed911bf6b5c8348fd -rw-r--r--     root    wheel    140 B /private/etc/pam.d/smbd
38b19f16f2cfe9cb398e59544cbbeeb4998718dbf231251a41eb31893aed223b -rw-r--r--     root    wheel   3.5 kB /private/etc/postfix/bounce.cf.default
6dcf0856acd475d75f7d4f2cb5e6a743b2df97ddd394b78e07ae172ab7f04363 -rw-r--r--     root    wheel    20 kB /private/etc/postfix/postfix-files
05e4653d532915fca72e4352ade5040c3361340920eee047ac4dd313bc9d8e19 -rw-r--r--     root    wheel   1.6 kB /private/etc/ssh/ssh_config
49dede3f9e65fd6ea036bc1598e07f1f73a3ad9c70c4744de954d6d0a8fdbdb5 -r--r--r--     root    wheel    194 B /private/etc/apache2/other/php7.conf
a5ff0b83be70bdb0107e8e6f4d2bc8e3608e7e9b15e15145b5f24ca4c559f504 -rw-r--r--     root    wheel   1.9 kB /private/etc/autofs.conf
1c350ccbbf5e4e09923f5bd1d93fd1578e0a5b6b7d4b85c631d09e1a1a71dbd3 -r--r--r--     root    wheel   6.3 kB /private/etc/openldap/schema/inetorgperson.schema
3ada6707f5fd265c70176f24ff988acb15983df6b2b99a5b3cdb833c8c38ea9f -r--r--r--     root    wheel    74 kB /private/etc/openldap/schema/cosine.schema
c3a8ab17dd75c874d9da896e729c516273900f8e5b80538daf60e8e95ff86660 -rw-r--r--     root    wheel    22 kB /private/etc/postfix/access
effcfce27485ef1ca9142df4fe810d518e720092b00dce0a9504e2ef378cc42c -rw-r--r--     root    wheel    27 kB /private/etc/postfix/main.cf.default
768ff09154f6aacda857cb175ef29cf9d23ef9c38c69efdbf20354dbfd7875b1 -rw-r--r--     root    wheel   1.6 kB /private/etc/rc.common
0235d3c1b6cf21e7043fbc98e239ee4bc648048aafaf6be1a94a576300584ef2 -r--r--r--     root    wheel    255 B /private/etc/zprofile
a9419086fc2b70f69130c3ee9f8761b0a12b0c7da47d3b779b04ab3827081cf0 -rw-r--r--     root    wheel   5.1 kB /private/etc/apache2/original/extra/httpd-languages.conf
7c982d301cf583c578abb470f5f3782d1d4451e43f5b90c6555ae75be6060d1a -rw-r--r--     root    wheel     53 B /private/etc/networks
d8b2b88b7cdec55c5150ced01d4e0a90b275bb11483086ea74cfcd6687309bae -r--r--r--     root    wheel   177 kB /private/etc/openldap/schema/microsoft.ext.schema
13e0a0a4991f31f4864096846b379a1d96808c2f0a735b6661aae29481f4bb94 -rw-r--r--     root    wheel    356 B /private/etc/pam.d/su
0fd7f2c4f02e68615f137ed090cbba2d1facb40da6bf269e1e326ed6a10aa618 -rw-r--r--     root    wheel    329 B /private/etc/pf.anchors/com.apple
847e127803905f596f8c2553ddab8e2d0bca21b4c4ebc79e4be19dcb554df9cb -rw-r--r--     root    wheel   1.3 kB /private/etc/bootpd.plist
e93775ee26b46f1bfc6b5eab5eab05cee15a1718e87a4a01cfbd474b5409d872 -rw-r--r--     root    wheel    621 B /private/etc/locate.rc
0c4f41f80e397939b4a9ce2137a1d725efb4d373e0587c8551b9f1ac12ed0a56 -rw-r--r--     root    wheel    312 B /private/etc/pam.d/screensaver_aks
c9acb4b8a3e8fd249e672516e37a88d37a9bc8f224b0ab29e76d302c4179de04 -rw-r--r--     root    wheel    28 kB /private/etc/pf.os
38b6d46f6924364dc3013ffad83a12b7bc10bf3a878b5ae0f53ef117851900b8 -rw-r--r--     root    wheel   1.7 kB /private/etc/apache2/original/extra/httpd-dav.conf
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 -rw-r--r--     root    wheel      0 B /private/etc/find.codes
dbeff46b11216fc8b39dd3160f5085309c031044afdc77f394c94baa558abb60 -rw-r--r--     root    wheel     19 B /private/etc/manpaths.d/40-XQuartz
c998ee1b26049571ed20dc89e7dd9d3e87e85846bdf0d9ff1aae9f760774bf5b -rw-r--r--     root    wheel   1.1 kB /private/etc/apache2/extra/httpd-info.conf
b22a5ff409482104f804f04b0035f2fd1f6801974645f79c87e9025729a50286 -rw-r--r--     root      _lp   3.1 kB /private/etc/cups/cups-files.conf
411408289c874dfe0cb497c7d09e0e69601183ad7c9cbedcb9eae7543ff849c4 -r--r--r--     root    wheel   4.8 kB /private/etc/openldap/schema/duaconf.ldif
3c4f529ed0ba941121c3bb29d6e654299d9d9c77a98c2570465f15b0cce05aef -rw-r--r--     root    wheel    408 B /private/etc/pam.d/screensaver
081c0787da77e326dcf46a6d955cc807c42ad2362e99876c1c71257b38ff0c84 -rw-r--r--     root    wheel   1.3 kB /private/etc/ttys
2012882e055c2a2502cd7d3dd373c1804f0fce782225e90aca491abad720b53a -rw-r--r--     root    wheel   1.5 kB /private/etc/apache2/extra/httpd-vhosts.conf
43b85836499d3227d242c8ed7a9f33ee61a0b6be1df7bc41f962eff46b547ef1 -rw-r--r--     root    wheel   1.4 kB /private/etc/asl/com.apple.contacts.ContactsAutocomplete
41cdd0a28b1e13f3a475fafd45dec6d335cddecd6d2872a71450d5ab3336ee67 -rw-r--r--     root    wheel   4.6 kB /private/etc/man.conf
933a32202a6ca9d47c1a0af25f8577369b0134ce749002de1228f2b23a909142 -rw-r--r--     root    wheel     43 B /private/etc/nfs.conf
9b5ee3e68d2668a06a26bc29488c891a0182b4f2858f8bab97835d4238c8e892 -r--r--r--     root    wheel   3.5 kB /private/etc/openldap/schema/inetorgperson.ldif
6191e98d0228c084ec94f80af3639912b53c316db2cbf29a2123eab64491f709 -r--r--r--     root    wheel   4.1 kB /private/etc/openldap/schema/krb5-kdc.schema
7ba16d87de833e0b81898b544f981cd8b7dedcb8e9ad285bd954af560e089816 -rwxr-xr-x     root    wheel    620 B /private/etc/periodic/weekly/999.local
ed9d05e8ec15f676263a184a7c30858f2d5cd0258da168d3b262b76567bc7bae -rw-r--r--     root    wheel   2.9 kB /private/etc/apache2/extra/httpd-default.conf
b17912cf8ac845d1098829a3f876bdc7597bc8360bb8f260e49d1e3ca26d5029 -rw-r--r--     root    wheel   1.4 kB /private/etc/apache2/original/extra/httpd-manual.conf
eb33ba8357c7166620b5c33fc8436a0081ba093f7cbf60fef7ddb35b491a4012 -rw-r--r--     root      _lp    128 B /private/etc/cups/snmp.conf.default
9d70873703af389018ccc7b57a503d2692ba1d6b71271bcf00473d40f5095486 -rw-r--r--     root    wheel   3.2 kB /private/etc/apache2/extra/proxy-html.conf
b5847002ceeb4b141a882ba230eb84c186e2d59c74fbb718cc323c6b157ebde8 -r--r--r--     root    wheel   8.5 kB /private/etc/openldap/schema/netinfo.schema
cc58ac4627390ef04037b7b77ed0359b267a8e8ceb99bdc3aa156dae6367094a -rw-r--r--     root    wheel    607 B /private/etc/apache2/extra/httpd-userdir.conf
7ed4a5bae35e87e8815cb79dd3be70ac34a3056cb91216a22224a3c99b4c9720 -rw-r--r--     root    wheel    350 B /private/etc/asl/com.apple.mkb
125333636796ce03bacf4b079a93b90d3d458a81159c9593339183be5bf51080 -r--r--r--     root    wheel   6.9 kB /private/etc/openldap/schema/pmi.ldif
8a8b640738d46484ecc24c689f70702c5ca35dcd4868f7f413bd8727cd474f10 -rw-r--r--     root    wheel     36 B /private/etc/manpaths
```

Or list files **within a specific subdirectory**:

```sh
$ plakar ls 9:/private/etc/uucp            
b86ff58053e9e930a0324f7fb5e0458213567feca9e97de92da20bff82f17e06 -r--r--r--     root    wheel   2.4 kB /private/etc/uucp/sys
17294d602f2d28944e6517a6a8a432548351d1eaf468062b8da6d84bbf7c5440 -r--r--r--     root    wheel    133 B /private/etc/uucp/passwd
863f779b43680f81799688e91c18a164047a1a8e9dfff650881e50ba35530ed1 -r--r--r--     root    wheel    141 B /private/etc/uucp/port
```

These listings **rely on the snapshot index** and **do not fetch chunks** from the repository,
however it is possible to actually look at the file content with the `cat` subcommand:

```sh
$ plakar cat 9:/etc/uucp/passwd
#
# The passwd configuration file
#

#
# Specify a login name and password that a system can use when logging in.
#

uucp_tst sekret
```

Now let's make a change to that file and push it again:
```sh
$ sudo sed -i '' -e 's/sekret/notsosecret/' /etc/uucp/passwd

$ plakar push /private/etc                                  
open /private/etc/krb5.keytab: permission denied
open /private/etc/aliases.db: permission denied
open /private/etc/racoon/psk.txt: permission denied
open /private/etc/security/audit_user: permission denied
open /private/etc/security/audit_control: permission denied
open /private/etc/sudoers: permission denied
open /private/etc/sudo_lecture: permission denied
open /private/etc/master.passwd: permission denied
open /private/etc/openldap/slapd.conf.default: permission denied
open /private/etc/openldap/DB_CONFIG.example: permission denied
b41802cb-9424-47eb-8186-2847678a8646: OK
```

From there,
it is possible to perform an **inode diff**:
```sh
$ plakar diff 9 b
-  drwxr-xr-x     root    wheel    160 B 2020-01-01 08:00:00 +0000 UTC /private/etc/uucp
+  drwxr-xr-x     root    wheel    160 B 2021-03-27 22:38:17.685871971 +0000 UTC /private/etc/uucp
-  -r--r--r--     root    wheel    133 B 2020-01-01 08:00:00 +0000 UTC /private/etc/uucp/passwd
+  -r--r--r--     root    wheel    138 B 2021-03-27 22:38:17.685791098 +0000 UTC /private/etc/uucp/passwd
```

This shows that the `/private/etc/uucp` directory has had its **date** change,
and that `/private/etc/uccp/passwd` has had its **date** and **size** change,
and from there it is possible to **request a diff of the file** with the `diff` subcommand:

```sh
$ plakar diff 9 b /private/etc/uucp/passwd
--- 98b6658b-a975-47c4-a38f-f958d8d7359f:/private/etc/uucp/passwd
+++ b41802cb-9424-47eb-8186-2847678a8646:/private/etc/uucp/passwd
@@ -6,5 +6,5 @@
 # Specify a login name and password that a system can use when logging in.
 #
 
-uucp_tst sekret
+uucp_tst notsosecret
```

The snapshots are **not incremental** so it is possible to **generate diffs between any of them**.


# Removing old snapshots

Even though snapshots of similar content do not consume much space,
there's no use in keeping a lot of old snapshots hanging around.
Snapshots can be removed from the repository with the `rm` subcommand:

```sh
$ plakar ls       
98b6658b-a975-47c4-a38f-f958d8d7359f [2021-03-27T22:29:09Z] (size: 3.0 MB, files: 230, dirs: 39)
b41802cb-9424-47eb-8186-2847678a8646 [2021-03-27T22:38:22Z] (size: 3.0 MB, files: 230, dirs: 39)

$ du -sh ~/.plakar
2.2M    /Users/gilles/.plakar

$ plakar rm 9
98b6658b-a975-47c4-a38f-f958d8d7359f: OK

$ du -sh ~/.plakar
2.1M    /Users/gilles/.plakar

$ plakar rm b     
b41802cb-9424-47eb-8186-2847678a8646: OK

$ du -sh ~/.plakar
  0B    /Users/gilles/.plakar 
```

The repository **keeps chunks as long as they are referenced** by at least one snapshot,
but **removes them when they become orphans** which is why the **repository becomes empty after the last snapshot is removed**.



# Namespaces

Sometimes I want to make sure that my backups are **fully disjoint one from another**,
not sharing their snapshots and deduplication.
The `plakar` repository supports **namespaces** so that I can keep related snapshots together and apart from others.

For example,
I have backups for `poolp.org` and for `hypno.cat` and I want to manage the snapshots separately:

```sh
$ plakar -namespace poolp.org push data
351348b3-8dc3-4caa-b0be-334c43f4fd13: OK

$ plakar -namespace hypno.cat push data
1fa2dac2-8096-483a-a2d3-2a78ce3706ce: OK

$ plakar -namespace poolp.org ls            
351348b3-8dc3-4caa-b0be-334c43f4fd13 [2021-03-27T22:49:58Z] (size: 896 kB, files: 79, dirs: 4)

$ plakar -namespace hypno.cat ls            
1fa2dac2-8096-483a-a2d3-2a78ce3706ce [2021-03-27T22:50:40Z] (size: 896 kB, files: 79, dirs: 4)
$
```

# Stores

In addition,
I may want to provide an **alternate path** to the repository because `~/.plakar` is not ideal,
`plakar` supports providing alternate repository path with the `-store` option:

```sh
$ plakar -store /var/backups/gilles push data
f6cc2508-ea59-4552-aa4b-27900f0a9781: OK

$ plakar -store /var/backups/gilles ls
f6cc2508-ea59-4552-aa4b-27900f0a9781 [2021-03-27T22:52:47Z] (size: 896 kB, files: 79, dirs: 4)
```

But this doesn't stop here ...


# `plakar` server

So far I only showed `plakar` working on a **local repository**,
it however supports a network mode.

It is possible to start a `plakar` server on a machine with:

```sh
$ plakar server 192.168.0.1:2222    
$
```

and on another machine, point the repository to that server:
```sh
$ plakar -store plakar://192.168.0.1:2222 push /bin
3ac8f3ff-e723-46c3-9ae7-4b11bb27f3b2: OK
$
```

The plakar server can use `-store` itself,
so it is possible to configure it to store at at particular location:
```sh
$ plakar -store /var/backups server 192.168.0.1:2222
$
```

and because **I made a monster**,
it is possible that this location is **a different plakar server**,
effectively turning a plakar instance **into a proxy**:

```sh
box1$ plakar server 192.168.0.1:2222
box1$

box2$ plakar -store plakar://192.168.0.1:2222 server 192.168.0.2:2222
box2$

box3$ plakar -store plakar://192.168.0.2:2222 push /bin
998964ac-d3f6-4bf3-b784-50ba616e0776: OK
$
```

From the local user point of view,
pushing to a local plakar,
to a remote plakar,
or even to a remote plakar proxying to another plakar **work the same**.


# Is it released ?

Nope, **not yet**.

This is a PoC and **my first real project** in Golang,
so I need a bit of time to make sure it is in a state that I'm not ashamed to release.

If you are a sponsor and want to give it a try,
just mail me and I'll arrange that.


# What's next ?

A lot of things really...

First of all,
every single command needs **a bit of polishing** to have a nice feel because **the output is not always the best** to work with.

Then,
I have **not implemented any caching whatsoever** so even though it's not dead slow,
a lot of operations **that could be skipped** are **always performed** making it slower than it should.

Also,
the server mode **doesn't use encryption at this point** and while it is ok on a personal network,
I want to be able to deploy a `plakar` server on a dedicated server at some provider and use it safely.

Finally,
I began **building an UI to browse snapshots** and this requires building a global index of snapshots,
which is something I haven't tackled yet.
At this point,
**each snapshot has its own index** but I'd like to be able to have a search box which **looks up for stuff through all snapshots**.

I also have other private plans with this which I'll discuss later ;-)

