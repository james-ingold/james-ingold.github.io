---
title: Restoring a MySQL Database from .frm and .ibd Files on MacOS
description: Restoring data from a corrupted database
author: James Ingold
published: true
---

# Restoring a MySQL Database from .frm and .ibd files on MacOS

Ending 2020 by losing all your MySQL data seems about right doesn't it? I spent a full day recovering my data after updating to the latest Docker version on a NAS. It nuked my MySQL container in the process. Luckily, I had some database files but they were in .frm and .ibd format which are not easy to import, especially once the server is gone. In this article I'll show you how to restore your MySQL data on MacOS if you only have .frm and .ibd files available. Hopefully, this will save you some time if you are in the same position I found myself in.

A brief overview of the files we have and what we'll be doing to salvage the data:

- .frm: Contains a single table definition, essentially create table statements
- .ibd: This file holds the data and indexes for a single table on the file system.

#### Restore Process High-Level:

- Generate create statements from .frm files to get our table structures back
- Reset the tablespace: when an InnoDB table is truncated, or dropped and recreated, it gets a new table ID. Any ID mismatch between the table in the database and the backed-up table can prevent it from being restored. Since we're restoring this on a different MySQL Server instance, we need to drop the tables ids.
- Put our .ibd files into the data directory
- Sync the tablespace: Import new tables ids
- Backup our data so we have full backup which can easily be restored

#### Let's Get Started

Install MySQL shell client:

```bash
brew install mysql-client
```

Add it to your path:

```bash
export PATH="/usr/local/opt/mysql-client/bin:$PATH"
```

Reload your shell

```bash
source ~/.zshrc
```

Let's begin by starting up a fresh docker instance that we'll use to restore our database.

Create config and data volumes. The config volume will hold additional MySQL configuration and the data volume will be where our data goes on disk.

```
mkdir ~/docker/mysql/conf.d
mkdir ~/docker/mysql/data

```

Add the following to a ~/docker/mysql/conf.d/config.cnf

```
[mysqld]
innodb_force_recovery = 0
```

Run a MySQL docker instance

```
docker run \
--detach \
--name=mysql2 \
--env="MYSQL_ROOT_PASSWORD=mypassword" \
--publish 32780:3306 \
--volume=/pathToMysqlConfig/conf.d:/etc/mysql/conf.d \
--volume=/pathToDataDirectory/mysql2:/var/lib/mysql:rw mysql:5.7
```

### Step 1: Create Files to Reconstruct Table Structure

There are two ways to reconstruct the tables. The first option is to use MySQL command line utilities and the mysqlfrm tool. The other option is to create our tables with a dummy structure using some command line magic and then replace the dummy .frm's in our data directory with the actual .frms. I've used both but prefer the former, I'll show you how to do both.

#### Step 1.1: Use MySQL's command line utilities

The Python MySQL connector is required. Download it from here: http://dev.mysql.com/downloads/connector/python/
Download MySQL Utilities source code by selecting "Source Code" from "Select Platform" in http://dev.mysql.com/downloads/utilities/
I downloaded the Windows distribution and unzipped it (running on a mac)
From a terminal:

```bash
$ cd path/to/unzipped/mysql-utilities-1.5.6
$ python ./setup.py build
$ sudo python ./setup.py install
```

##### Use mysqlfrm to generate table creates

```bash
mysqlfrm  --diagnostic ~/folder/*.frm --quiet > ~/folder/recoverme-table-structures.sql
```

Set the ROW_FORMAT to compact

```bash
sed -i '' -e 's/ENGINE=InnoDB/ ENGINE=InnoDB ROW_FORMAT=COMPACT DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci/' recoverme-table-structures.sql
```

### Step 2 - Generate dummy tables creates using .frm files

Even though we're going to use the output from Step 1, we will use the files generated in this step to reset the tablespace and import so go ahead and follow along. This is useful when you have many tables to restore.

First we'll get the table names from our .frm files

```bash
ls -1 *.frm > tables.sql
```

Prepend create table in front of each table name

```bash
awk '{print "create table " $0}' tables.sql > rtables.sql
```

Replace .frm and add a dummy int column with engine innodb

```bash
sed -i '' -e 's/.frm/ (id int) engine=innodb;/' rtables.sql
```

At this point, you should have three files: recoverme-table-structures.sql, tables.sql, rtables.sql.

### Step 3 - Create Our Tables

First let's create our database

```bash
mysql -uroot -p --host=127.0.0.1 -P32780 -e "create database dbName"
```

Next, run sql

```
mysql -uroot -p --host=127.0.0.1 -P32780 dbName < recoverme-table-structures.sql
```

If you were to run rtables.sql instead of recoverme-table-structures.sql, you would have your tables added with a dummy id column. At that point, you would stop mysql and then copy over your .frm files into the mysql database directory, replacing the existing .frm files. After that you would run `flush tables`.

### Step 4 - Replace Data

Now the spot we're in is that our internal id of our tables is different than the one in our .ibd files. We'll manually sync our tablespaces.

Create remove-tablespace.sql to remove the existing internal ids.

```bash
cp rtables.sql remove_tablespace.sql
sed -i '' -e 's/create/alter/' remove_tablespace.sql
sed -i '' -e 's/ (id int) engine=innodb;/ discard tablespace;/' remove-tablespace.sql
```

Create sync-tablespace.sql to import tablespace from existing data.

```bash
cp rtables.sql sync_tablespace.sql
sed -i '' -e 's/create/alter/' sync_tablespace.sql
sed -i '' -e 's/ (id int) engine=innodb;/ import tablespace;/' sync-tablespace.sql
```

Run remove-tablespace.sql

```
mysql -uroot -p --host=127.0.0.1 -P32780 dbName < remove-tablespace.sql
```

Remove any existing .ibd files in our MySQL directory

```
cd ~/MySQL/recoverme
rm -f *.ibd
```

Copy .ibd files over to MySQL directory

```
cp ~/directoryMyRestoreFilesAre/*.ibd .
```

Sync tablespace

```
mysql -uroot -p --host=127.0.0.1 -P32780 dbName < sync-tablespace.sql
```

If you get error - Schema mismatch (Table has ROW_TYPE_COMPACT row format, .ibd file has ROW_TYPE_DYNAMIC row format.). Restart the process and sed ROW_FORMAT=DYNAMIC instead of ROW_FORMAT=COMPACT

Dump our database to create a backup

```
mysqldump recover > recoverme-structure-data.sql
```

Now we have a legitimate MySQL dump of our data. We can import that in with the correct database name

```
mysql -e "create database actualdatabase"
mysql actualdatabase < recoverme-structure-data.sql
```

That's it! There are certainly more issues that can come up if the ibd files are corrupted. I encourage you to check out the [Twindb github](https://github.com/twindb) for some excellent tools on repairing ibd files and general data recovery practices.

#### References

[Dealing with corrupted InnoDB data](https://www.percona.com/blog/2016/01/19/dealing-with-corrupted-innodb-data/)

[Recover Orphan InnoDB Database from .ibd file](https://www.ipserverone.info/knowledge-base/how-to-recover-an-orphan-innodb-database-from-ibd-file)

[SO Recover Table Structure from ibd files](https://stackoverflow.com/questions/26868956/restore-table-structure-from-frm-and-ibd-files)

[SO Repair/Recover data from MySQL database](https://dba.stackexchange.com/questions/217352/repair-recover-data-from-mysql-database)

[Undrop for InnoDB Twindb CLI Tool](https://github.com/twindb/undrop-for-innodb)

[Using Undrop](https://web.archive.org/web/20201024174922/https://twindb.com/undrop-for-innodb/)

[Twindb - Recovering Corrupt MySQL database](https://web.archive.org/web/20210410194008/https://twindb.com/recover-corrupt-mysql-database/)
