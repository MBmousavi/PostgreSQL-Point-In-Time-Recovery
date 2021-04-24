# Point In Time Recovery (PITR) in PostgreSQL 9.6
This document describes how to do point in time recovery with PostgreSQL 9.6 : PITR. Point in time recovery (PITR) simply means restoring data to certain point in time.

First let's assume what could go wrong and make us to perform a PITR on production environment:

1. Someone has accidentally dropped or truncated a database,table or records.
2. A failed deployment has made changes to the database that are difficult to reverse.
3. You accidentally deleted or modified a lot of data, and as a consequence, you cannot run your applications.
4. You have a single node database and it crashes, and you need to setup a secondary database ASAP.
5. and so on ...

In such scenarios, you would immediately look for the latest full backup to recover up to a known point in the past, before the error occurred. But what if your backup is corrupt and not valid? or restoring that full backup causes some data lost ? Imagine you set a set a job to take a full backup every night at 1 AM. The database crashes or some data accidentally deleted at 10 AM. If you process to restore the full backup from 1 AM last night, the changes between 1 Am to 10 AM will be lost. Therefore you need to think about how you can recover all the changes that happened since you took the last full backup.

Now you can understand the problem. Let's setup our environment first. We use a simple `docker-compose.yml` file to start our database:

The `postgre9_1` is our primary database, and `postgre9_2` will be our secondary database that we'll setup after primary database failure.

Run the docker compose like this to start `postgre9_1`:

`docker-compose up -d postgre9_1`

## Setting up WAL Archiving
PostgreSQL stores all modifications in form of Write-Ahead Logs (WAL). These files can be used for replication, but they can also be used for PITR. First of all, we need to setup WAL archiving. Open `postgresql.conf ` with a text editor and put this lines:
```
archive_mode = on
wal_level = replica
max_wal_senders = 10
```
* **archive_mode** signifies whether we want to enable the WAL archiving.
* **wal_level** accepts the following values: Minimal, Replica, Logical.
* **max_wal_senders** max number of walsender processes.

Save and exit the file. Now open `pg_hba.conf` with a text editor and put this line:
```
host    replication     postgres    172.21.0.1/32    trust     # 172.21.0.1 is docker network gateway
```
Now restart the database container to take affects the new configuration. `docker restart postgres9_1`

## Setting up Database

Connect to the PostgreSQL shell to create a sample database and insert some data.

This diagram will be our scenario:

![Screenshot from 2021-04-24 09-51-42](https://user-images.githubusercontent.com/38520491/115948385-55dcd780-a4e3-11eb-935a-db0ae667751f.png)

Point 1. Create a demo database and table, then insert some data.

Point 2. Create a Base backup and restore it to secondary database. (Imagine it's 1 AM full backup)

Point 3. Insert some new data. (Imagine it's data changes from 1 AM to 10 AM)

Point 4. Insert some wrong data. (It's the point that data has been corrupted or deleted or wrong added. We'll recover to point 3)

**Point 1:**
```
PGPASSWORD=admin@123 psql -h 172.21.0.2 -U postgres

postgres=# create database demo;
CREATE DATABASE
postgres=# create table tbl1 (col1 int);
CREATE TABLE
postgres=# insert into tbl1 (col1) select generate_series(1, 10000000);
INSERT 0 10000000
postgres=# select count(*) from tbl1;
  count  
---------
 1000000
(1 row)
```

**Point 2:**
Alright, now create a base backup:

`GPASSWORD=admin@123 pg_basebackup  -h 172.21.0.2 -U postgres  -D  pgbackup` 

The `pgbackup` is the output directoy. We copy it to the data directory of secondary database. It's `postgre9_2` in `docker-compose.yml`.

`cp -r pgbackup postgre9_2`. It's like restoring backup on secondary database.

**Point 3:**
Now we insert more 10M records, and check the time because we will restore to this timestamp:
```
PGPASSWORD=admin@123 psql -h 172.21.0.2 -U postgres

postgres=# \c demo;
You are now connected to database "demo" as user "postgres".
demo=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | tbl1 | table | postgres
(1 row)

demo=# insert into tbl1 (col1) select generate_series(1, 10000000);
INSERT 0 10000000
demo=# select count(*) from tbl1;
  count  
---------
 20000000
(1 row)

demo=# select now();
               now                
----------------------------------
 2021-04-24 17:06:00.018964+04:30
(1 row)
```

**Point 4:**
Now we add wrong 5M records to database:
```
demo=# insert into tbl1 (col1) select generate_series(1, 5000000);
```
Now we've come to the critical situation. We begin to restore secondary database to Point 3.

By default Postgre write the WAL files into `pg_xlog` subdirectory of it's  data directory. We copy all the files from this directory and put it some where that secondary database can access it. In production environments usually we store these WAL files somewhere safe. Here we copy WAL files to a new directory (restore_wal) and mount it as volume to secondary database. secondary database access it within /home.
  
Create a `recovery.conf` in data directory of secondary database, Put these line in it:

```
restore_command = 'cp /home/%f "%p"'
recovery_target_time = '2021-04-24 17:06:00.018964+04:30'
recovery_target_timeline = '1'
```
* **restore_command** specifies the shell command that is executed to copy log files back from archival storage. The command string may contain %f, which is replaced by the name of the desired log file, and %p, which is replaced by the absolute path to copy the log file to.

* **recovery_target_time** specifies the time until when we need the changes.
* **recovery_target_timeline** Specifies recovering into a particular timeline.

Now everything is setup, so we can start the secondary database with `docker-compose up -d postgre9_2`

You can see the logs by `docker logs postgres9_2`:

```
LOG:  database system was interrupted; last known up at 2021-04-24 16:18:38 +0430
LOG:  starting point-in-time recovery to 2021-04-24 17:06:00.018964+04:30
LOG:  restored log file "00000001000000000000000D" from archive
LOG:  redo starts at 0/D000028
LOG:  consistent recovery state reached at 0/D0000F8
LOG:  restored log file "00000001000000000000000E" from archive
LOG:  restored log file "00000001000000000000000F" from archive
LOG:  restored log file "000000010000000000000010" from archive
LOG:  restored log file "000000010000000000000011" from archive
LOG:  restored log file "000000010000000000000012" from archive
LOG:  restored log file "000000010000000000000013" from archive
LOG:  restored log file "000000010000000000000014" from archive
LOG:  restored log file "000000010000000000000015" from archive
LOG:  restored log file "000000010000000000000016" from archive
LOG:  restored log file "000000010000000000000017" from archive
LOG:  restored log file "000000010000000000000018" from archive
LOG:  restored log file "000000010000000000000019" from archive
LOG:  restored log file "00000001000000000000001A" from archive
LOG:  restored log file "00000001000000000000001B" from archive
LOG:  restored log file "00000001000000000000001C" from archive
LOG:  restored log file "00000001000000000000001D" from archive
LOG:  restored log file "00000001000000000000001E" from archive
LOG:  restored log file "00000001000000000000001F" from archive
LOG:  restored log file "000000010000000000000020" from archive
LOG:  restored log file "000000010000000000000021" from archive
LOG:  restored log file "000000010000000000000022" from archive
LOG:  restored log file "000000010000000000000023" from archive
LOG:  restored log file "000000010000000000000024" from archive
LOG:  recovery stopping before commit of transaction 560, time 2021-04-24 17:10:48.120278+04:30
```
Connect to database to check the records:
```
PGPASSWORD=admin@123 psql -h 172.21.0.3 -U postgres

postgres=# \c demo;
psql (12.6 (Ubuntu 12.6-0ubuntu0.20.04.1), server 9.6.21)
You are now connected to database "demo" as user "postgres".
demo=# select count(*) from tbl1;
  count  
---------
 20000000
(1 row)
```
Finally we have our 20M healthy recods.
After recovery postgre rename the `recovery.conf` to `recovery.done`.
