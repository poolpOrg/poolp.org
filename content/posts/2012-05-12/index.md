---
title: "OpenSMTPD meets SQLite"
date: 2012-05-12 14:23:54
category: OpenSMTPD
author: Gilles Chehade
---

During the r2k12 hackathon in Paris, Marc Espie committed SQLite to OpenBSD's base system.

This has the side effect that OpenSMTPD can start using it and while we agreed that we did not want it as a strong dependency, the various backends API allow us to make it a soft dependency that can be removed without breaking the daemon if someone really does not want SQLite linked.

Today I decided to give it a try and implement a SQLite backend to the map API. About ten minutes later (yes, really ten minutes !), I had a working prototype that was suboptimal and that didn't make use of SQL capabilities.

An hour later, I have a SQLite backend that will use multiple tables with different structures and that can be used to lookup aliases, virtual domains and credentials for authenticated relaying.

First you create a database with the following schema.sql:


-- TABLES REQUIRED BY THE MAPS BACKEND

CREATE TABLE IF NOT EXISTS aliases ( id INTEGER PRIMARY KEY AUTOINCREMENT, name VARCHAR(255) NOT NULL, address VARCHAR(255) NOT NULL );

CREATE TABLE IF NOT EXISTS secrets ( id INTEGER PRIMARY KEY AUTOINCREMENT, relay VARCHAR(255) UNIQUE NOT NULL, username VARCHAR(255) NOT NULL, password VARCHAR(255) NOT NULL );

CREATE TABLE IF NOT EXISTS virtual ( id INTEGER PRIMARY KEY AUTOINCREMENT, name VARCHAR(255) NOT NULL, address VARCHAR(255) NOT NULL );

Then you declare your map with source "sqlite":

map "aliases" { source sqlite "/etc/mail/sqlite.db" } map "virtmap" { source sqlite "/etc/mail/sqlite.db" } map "secrets" { source sqlite "/etc/mail/sqlite.db" }

accept for local alias aliases deliver to mbox accept for virtual virtmap deliver to maildir accept for all relay via "mail.poolp.org" tls auth "secrets"

And voila ! The lookups are performed at runtime, as usual, which means that you can add virtual domains, aliases or new credentials through SQL queries to the sqlite.db database.

The diff will only apply to OpenSMTPD for OpenBSD -current, it will not work as is on -portable but it should be committed pretty soon.
