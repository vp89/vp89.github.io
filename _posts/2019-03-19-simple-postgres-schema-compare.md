---
layout: post
title:  "A simple Postgres schema compare"
date:   2019-03-19 00:00:00 -0000
categories: programming
---

I thought I would share with you a shell script that uses pg_dump and a diff GUI to do a schema comparison between a development and staging/production database.

First let's create your script with the necessary permissions. I like to put these kind of things in a `scripts` folder at the root of my repo:

```
touch scripts/schema-compare.sh
chmod 711 scripts/schema-compare.sh
```

Open `scripts/schema-compare.sh` in your editor and add the following contents:

```
pg_dump --schema-only -xO -h 127.0.0.1 -p 5432 -U <localUsername> -d <localDatabase> | sed -e '/^--/d' > local.sql
pg_dump --schema-only -xOw -h <remoteHost> -p <remotePort> -U <remoteUser> -d <remoteDatabase> | sed -e '/^--/d' > remote.sql
/Applications/p4merge.app/Contents/Resources/launchp4merge local.sql remote.sql
rm local.sql
rm remote.sql
```

Replace `<localUsername>`, `<localDatabase>`, `<remoteHost>`, `<remotePort>`, `<remoteUser>`, `<remoteDatabase>` with your relevant values.

The third line of the script opens `p4merge` which is my diff GUI of choice on Mac. Feel free to swap in your preferred tool here, or [read my guide on using p4merge on Mac](https://vincepergolizzi.com/programming/2017/11/22/setup-p4merge-git-diff-osx.html).

You will notice I am using the `-w` flag only on my remote call, this is because I don't want it to prompt me for a password, but rather take it from a local dot file. This makes the schema compare a convenient one-liner and allows me to quickly iterate on my database schema as I am building the rest of my app.

I'm using `-x` and `-O` to remove any permissions related DDL, as well as `sed` to remove comments. This is to avoid diff lines which don't represent a real diff to my schema. Fast iteration and reducing friction around using schema is the name of the game here.

Before we can go ahead and run that, we will need to create or add to our `pgpass` file. This is a dot file that allows you to pass in database passwords to the various Postgres CLI utilities. If you are already familiar with this file, ensure that there is an entry to match the remote you setup earlier in the schema compare script.

If you have never used a `pgpass` file before, let's first create it:

```
touch ~/.pgpass
chmod 0600 ~/.pgpass
```

Open `~/.pgpass` in your editor and add the following contents:

```
# hostname:port:database:username:password
<remoteHostname>:<remotePort>:<remoteDatabase>:<remoteUsername>:<remotePassword>
```

Now, from the root of your repo run the script using `./scripts/schema-compare.sh`. If everything is setup correctly you should see a p4merge output like the following:

<img src="/p4merge-schema-compare.png" />

If everything is in sync, you should see 0 diffs.

I am still familiarizing myself with the Postgres tooling ecosystem, it seems a little bit more scattered than the SQL Server world I am used to but I will continue to share this kind of stuff as I discover it.
