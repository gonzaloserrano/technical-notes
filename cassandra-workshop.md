# Cassandra Workshop

## CAP theorem

Cassandra is tuned for different C, A and P tunnings depending on what we need. You can tune per query.

## K-V data store

- Equals to a hashmap e.g associative array in PHP.
- In Cassandra you just can query by primary key!
- But they are very fast!
- What if you dont have the field to query as a index? => create a new table!
- Cassandra is generally designed for very fast writes and depending on your tables for very fast reads

## Replication

- Murmur hash based data partitioning in different nodes
- A node is a coordinator for one query to avoid SPOFs and decides in what node
to store.
- Replicas, normally the minimum is 3, the coordinator sends simultaniosly to all
the replicas.
- By default is SimpleStrategy
- Support for multiples regions in AWS, eg EC2MultiRegionSnitch
    - e.g define 2 datacenters, one for us east y another for us west
    - define a different number of replicas per datacenter depending on e.g # of users in the frontal app

## Consistency level - CAP

- By default is 1: maximum availability (A), minimum consistency (C)
- Configurable per query!
- Consistency level "Any"
    - If all the replicas are down except one, the uniq node is the coordinator
    and saves the data itself.
    - Minimum consistency, maximum availability
    - Only for writes, does not work for reads, we write in the coordinator but
    the coordinator just has reads for its part of the ring
- Consistencty level "Quorum"
    - Half of the nodes + 1 must be alive
    - That's he sais that why the min num of nodes should be 3, because 3/2 + 1 = 2
- Always try to use consistency level 1 unless you need something special
- Consistency level "All": if one node is down you won't be able to write (rarely used!)
- CAP, distributed, ring 
 - a ring with virtual nodes drawing
 - each node has a 256 non-consecutive tokens to, when a node goes down, redistribute
    the charge between all of them

## Practice

### Old cassandra commands

- `$ ssh <cassandra-host>`
- `$ nodetool ring`
- $ nodetool info
- $ cassandra-cli
    DESCRIBE cluster;
    SHOW schema;
    SHOW keyspaces;
    describe;

### Intro

- Install cassandra etc.
- OSX: homebrew: $ brew install cassandra
- bug upgrading from 2.0.X to 2.1.X: `https://github.com/Homebrew/homebrew/issues/32488`
  - rm -rf /usr/local/etc/cassandra/
  - brew reinstall cassandra
- Check cluster status:
  - $ nodetool status
Access to the shell:
  - $ cqlsh

Cassandra Query Language Shell: new shell, cassandra-cli is not used anymore since is related to thrift format only.

CQL3 allows you to think in tables, rows and columns and you don't need the
"column family" and "super column family"

#### create keyspace

```
    cqlsh> desc KEYSPACES;
    cqlsh> desc KEYSPACE System;
    cqlsh> CREATE KEYSPACE sp WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
    cqlsh> desc KEYSPACE sp;
    cqlsh> desc TABLE paxos;
```

#### create table

- percona:
```
CREATE TABLE `user` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `pack` varchar(255) NOT NULL DEFAULT '',
  `data` mediumblob,
  PRIMARY KEY (`id`,`pack`)
) ENGINE=InnoDB AUTO_INCREMENT=10102 DEFAULT CHARSET=utf8;
```
```
CREATE TABLE user (id BIGINT, pack TEXT, data TEXT, PRIMARY KEY (id, pack))
```
- field type
    - uuid 256-byte hash for distributed autoincremental ids
```
    cqlsh> CREATE TABLE users (id uuid PRIMARY KEY, email text, name text);
    cqlsh> DESC TABLE users;
    cqlsh> DROP TABLE users;
```
uuids are not texts, don't put quotes!
```
    cqlsh:cql> INSERT INTO users (id, email, name) VALUES (7a19cd7e-b03f-11e3-b501-1231392ce81f, 'a@b.com', 'Gonzalo');
    cqlsh:cql> INSERT INTO users (id, email, name) VALUES (7a1a493e-b03f-11e3-bdd2-1231392ce81f, 'b@b.com', 'Pepe');
    cqlsh:cql> INSERT INTO users (id, email, name) VALUES (7a1ac5ee-b03f-11e3-8bb3-1231392ce81f, 'c@b.com', 'Paco');
```
### Cassandra insert path

1. hard disk storage => commit log
2. memory storage => SSTables

Inserts are very fast but if you do an insert with the same PK than another row then it does an update!

### select 
```
    cqlsh:cql> SELECT * FROM users;

     id                                   | email   | name
    --------------------------------------+---------+---------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | b@b.com |    Pepe
     7a19cd7e-b03f-11e3-b501-1231392ce81f | a@b.com | Gonzalo
     7a1ac5ee-b03f-11e3-8bb3-1231392ce81f | c@b.com |    Paco
```
- Exercise: change the email of the user "Pepe", how you do it?
 - YOU CAN'T! You need the id!
 - So to get the id of the user with name Pepe you need a table with ids the name
of the users and the values the uuids

```
    cqlsh:cql> UPDATE users SET name = 'Pepito' WHERE id = 7a1a493e-b03f-11e3-bdd2-1231392ce81f;

note: INSERT and UPDATE are called "UPSERT" because really is the same command.

    cqlsh:cql> SELECT * FROM users;

     id                                   | email   | name
    --------------------------------------+---------+---------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | b@b.com |  Pepito
     7a19cd7e-b03f-11e3-b501-1231392ce81f | a@b.com | Gonzalo
     7a1ac5ee-b03f-11e3-8bb3-1231392ce81f | c@b.com |    Paco
```
- Data types
    - Ascii => don't use it (no utf-8)
    - Most common ones:
        - Set
        - Int
        - Text ~ same as Varchar
        - Double
        - List
        - Timestamp
        - Boolean
        - Map
        - Uuid

### composite primary key
```
    cqlsh:cql> CREATE TABLE employees (company text, name text, age int, role text, primary key (company, name));
```
eg. PK with (key1, key2) => key1 is used for the hash in the ring (IMPORTANT)

Having "name" as key2 in the PK means:
- names should be uniq !
- a number of columns will be created, which is: number of different names multiplied by
    the number of columns that are not in the primary key (VERY IMPORTANT: Cassandra handles well till 100K columns then performance degrades!)
- in selects, inside the same company the rows will be ordered by names (not by company because the hash is used to store the first PK) => VERY IMPORANT since you are going to create tables with fields in the PK as you would have in MySQL SELECTs with ORDER BY e.g.

```    
    cqlsh:cql> SELECT * FROM employees ;

     company  | name   | age | role
     ----------+--------+-----+----------
           SP |   Pepe |  40 | analista
           SP |  User1 |  10 |     peon
           SP |  User2 |  20 |     jefe
          Foo | Vicent |  30 |     boss
```
Example

We want a query to get all the users from all companies ordered by role and then with the same role by name
- solution: create a table with the primary key (company, role, name)
- why do all this crazyness? because of very high performance :-)

```
    cqlsh:cql> CREATE TABLE employees_ordered (company text, name text, age int, role text, primary key (company, role, name));
    
    cqlsh:cql> SELECT * FROM employees_ordered WHERE role = 'prog';
    Bad Request: Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING
```

Why this error? You need to query using the PK from left to right.
Also ranges (e.g name > 'J') can be used just in the last of the columns you want to select by.
```
    cqlsh:cql> SELECT * FROM employees_ordered WHERE company = 'SP' AND role = 'prog';

     company | role | name | age
    ---------+------+------+-----
          SP | prog | Juan |  40
          SP | prog | Paco |  40

    cqlsh:cql> SELECT * FROM employees_ordered WHERE company = 'SP' AND role = 'prog' and name > 'Juan';

     company | role | name | age
     ---------+------+------+-----
           SP | prog | Paco |  40
```
How to update a user from company A to company B?
- You can't do UPDATE blablabla because company is PK.
- You need to DELETE register from A and INSERT to B
 - for consistency, an insert can't be done if the node is down, so
    what you want is to insert first if that works then delete
 - deletes don't erase but mark as delete and when the GC passes it's deleted
   - in old versions of cassandra the CG algo sucks so because its slow
    you need to call it manually with a cron
   - check this with systems!

Cassandra is eventually consistent because after down nodes are repaired the 
consistency of the cluster is also repaired in some time.

### Compount partition key

-  From Stack Overflow:
> The first column mentioned in the primary key is known as the partition key.
Any additional columns mentioned in the primary key are known as the clustering columns. All of the clustering columns for a given partition key are stored as a single Cassandra partition (guaranteed to be together on a single node) -
what used to be known as a "wide row". So, each treeid will refer to a single partition with each personid begin a row within the partition.
- More than 100K columns in cassandra degrades performance
- E.g. sensors: you have id, date and value
```
    PRIMARY KEY ((sensor_id, year), date)

it will be arranged in cassandra like:

                    DateOne:value DateTwo:value Date3:value
    Sensor_1:2014        20             0.5        -10.3
```
So if you have less than 100K for 2014 you won't have performance problems.
What if you have > 100K?
- You partition like ((sensor_id, year, trimestre), date)
- Or like ((sensor_id, year, trimestre, month), date)

```
    cqlsh:cql> CREATE TABLE sensor_data ( sensor_id uuid, year int, date timestamp,
    value decimal, PRIMARY KEY ((sensor_id, year),date);
```
(here do some inserts)
```
    cqlsh:cql> SELECT * FROM sensor_data;

     sensor_id                            | year | date                     | value
    --------------------------------------+------+--------------------------+-------
     7a1c36d6-b03f-11e3-8104-1231392ce81f | 2014 | 2014-03-18 12:31:12+0100 |    99
     7a1bbc06-b03f-11e3-8d9e-1231392ce81f | 2014 | 2014-03-19 12:31:12+0100 |    12
     7a19cd7e-b03f-11e3-b501-1231392ce81f | 2014 | 2014-03-20 17:21:06+0100 |    33

    cqlsh:cql> SELECT * FROM sensor_data WHERE sensor_id = 7a1c36d6-b03f-11e3-8104-1231392ce81f;
    Bad Request: Partition key part year must be restricted since preceding part is
```
You need to put both parts of the first part of the primary key!
```
    cqlsh:cql> SELECT * FROM sensor_data WHERE sensor_id = 7a1c36d6-b03f-11e3-8104-1231392ce81f AND year = 2014;

     sensor_id                            | year | date                     | value
    --------------------------------------+------+--------------------------+-------
     7a1c36d6-b03f-11e3-8104-1231392ce81f | 2014 | 2014-03-18 12:31:12+0100 |    99
```
### Lightweight transaction

> INSERT INTO ... VALUES (...) if not exists;
```
    cqlsh:cql> INSERT INTO user_by_email (email, user_id) VALUES ('salva@atrapalo.com',  7a1c36d6-b03f-11e3-8104-1231392ce81f) if not exists;
    
    applied
    -----------
          True

    cqlsh:cql> INSERT INTO user_by_email (email, user_id) VALUES ('salva2@atrapalo.com',  7a1c36d6-b03f-11e3-8104-1231392ce81f) if not exists;

     [applied] | email               | user_id
    -----------+---------------------+--------------------------------------
         False | salva@atrapalo.com | 7a1c36d6-b03f-11e3-8104-1231392ce81f
```
### BATCH
```
    cqlsh:cql> BEGIN BATCH
               INSERT INTO user_by_email (email, user_id) VALUES ('a@a.com', 7a1cb0c0-b03f-11e3-b54e-1231392ce81f);
               INSERT INTO users (id, email, name) VALUES (7a1cb0c0-b03f-11e3-b54e-1231392ce81f, 'a@a.com', 'john');
               APPLY BATCH
               ;
```
### SETS

There are no JOINs in Cassandra so you can use SETs. e.g to JOIN articles and
tags what you do is create a SET for tags and the PK is the article id or
something like that.
```
    cqlsh:cql> INSERT INTO article (id, content, tags) VALUES (7a19cd7e-b03f-11e3-b501-1231392ce81f, 'contenido articulo 1', {'tag1', 'tag2', 'tag3'});
    
    cqlsh:cql> INSERT INTO article (id, content, tags) VALUES (7a1a493e-b03f-11e3-bdd2-1231392ce81f, 'contenido articulo 2', {'tag8', 'tag7', 'tag6'});

    cqlsh:cql> SELECT * FROM article ;

     id                                   | content              | tags
     --------------------------------------+----------------------+--------------------------
      7a1a493e-b03f-11e3-bdd2-1231392ce81f | contenido articulo 2 | {'tag6', 'tag7', 'tag8'}
      7a19cd7e-b03f-11e3-b501-1231392ce81f | contenido articulo 1 | {'tag1', 'tag2', 'tag3'}
```
Delete part of a set:
```
    cqlsh:cql> UPDATE article SET tags=tags-{'tag8'} WHERE id = 7a1a493e-b03f-11e3-bdd2-1231392ce81f;

     id                                   | content              | tags
    --------------------------------------+----------------------+--------------------------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | contenido articulo 2 |         {'tag6', 'tag7'}
     7a19cd7e-b03f-11e3-b501-1231392ce81f | contenido articulo 1 | {'tag1', 'tag2', 'tag3'}
```
### LISTS

The column set as lists remains ordered as inserted.
You can:
- append
- prepend
- get and element by its position on the list

```
    cqlsh:cql> CREATE TABLE ROUTE(id uuid PRIMARY KEY, name text, cities list<text>);

    cqlsh:cql> INSERT INTO route (id, name, cities) VALUES (7a1a493e-b03f-11e3-bdd2-1231392ce81f, 'Main route', ['Madrid', 'Barcelona']);
    
    cqlsh:cql> SELECT * FROM route;

     id                                   | cities                  | name
    --------------------------------------+-------------------------+------------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | ['Madrid', 'Barcelona'] | Main route

    cqlsh:cql> UPDATE route SET cities[1] = 'Cuenca' WHERE id = 7a1a493e-b03f-11e3-bdd2-1231392ce81f;
    cqlsh:cql> SELECT * FROM route;

     id                                   | cities               | name
    --------------------------------------+----------------------+------------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | ['Madrid', 'Cuenca'] | Main route

    cqlsh:cql> UPDATE route SET cities = ['Roma', 'Paris'] + cities WHERE id = 7a1a493e-b03f-11e3-bdd2-1231392ce81f;
    cqlsh:cql> SELECT * FROM route;

     id                                   | cities                                | name
    --------------------------------------+---------------------------------------+------------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f | ['Paris', 'Roma', 'Madrid', 'Cuenca'] | Main route
```
### MAP
```
    cqlsh:cql> INSERT INTO PRODUCT (id, name, features) VALUES (7a1a493e-b03f-11e3-bdd2-1231392ce81f, 'Table', {'height':'100', 'width':'60', 'color':'red'});
    
    cqlsh:cql> INSERT INTO PRODUCT (id, name, features) VALUES (7a19cd7e-b03f-11e3-b501-1231392ce81f, 'Table', {'height':'100', 'width':'60', 'color':'red'});
    
    cqlsh:cql> SELECT * FROM product;

     id                                   | features                                         | name
    --------------------------------------+--------------------------------------------------+--------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f |               {'material': 'wood', 'size': '10'} | Window
     7a19cd7e-b03f-11e3-b501-1231392ce81f | {'color': 'red', 'height': '100', 'width': '60'} |  Table

    cqlsh:cql> UPDATE products SET features['material'] = 'glass' WHERE id = 7a1a493e-b03f-11e3-bdd2-1231392ce81f;

    cqlsh:cql> SELECT * FROM product;

     id                                   | features                                         | name
    --------------------------------------+--------------------------------------------------+--------
     7a1a493e-b03f-11e3-bdd2-1231392ce81f |              {'material': 'glass', 'size': '10'} | Window
     7a19cd7e-b03f-11e3-b501-1231392ce81f | {'color': 'red', 'height': '100', 'width': '60'} |  Table
```
### LOL :)
```
    cqlsh:cql> DELETE from users;
    Bad Request: line 1:17 mismatched input ';' expecting K_WHERE
```
### TTL

INSERT / UPDATE using TTL

### STRATEGY TO MOVE DATA TO ANOTHER CONFIGURATION WITH OTHER TABLES

The point here is, if we have just one datacenter (if we had two we could use the #2 while we do the migration) and for each node, and one by one to not affect performance:
 - create the new tables.
 - do a SELECT with a WHERE using the hash function of the ring tokens
    assigned tokens that node to just select the rows that we know that are
    stored in the current node we execute the script from.
 - insert in the new table.
 - change node and repeat to the rest of nodes.
 - do the switch.
 - note to myself: what happens with the data inserted in the old tables after you complete the migration? :(

Or use hadoop enterprise (dollars) with cassandra plugin to execute a hadoop job that will be executed with that local cassandra data and do
the migration.

# Cassandra 2.1 I+D

## General

- Snappy compression in tables with rows with same number of columns
- We used DataStax OpsCenter to graph the CPU, Memory, I/O, and latency over all four of our writeable nodes over the entire migration. We set our write consistency to 1, which is what we use in production.
- writes per second
- cpu / ram
- tables size
- R/W latency
- Repairs: cassandra needs repair ops to be launched in order to optimize the data
    - automatic repairs: not yet https://issues.apache.org/jira/browse/CASSANDRA-10070
    - shoot every 10 days?
- Sqoop to migrate from RDBMs to Cassandra
    - http://www.datastax.com/2012/11/ways-to-move-data-tofrom-datastax-enterprise-and-cassandra
- nodetool cfhistograms <keyspace> <table>
- http://www.datastax.com/dev/blog/row-caching-in-cassandra-2-1
- http://www.slideshare.net/planetcassandra/c-summit-eu-2013-apache-cassandra-20-data-model-on-fire
- Puede interesar cambiar la estrategia de flush al FS del commit log a batch para ser más “durable”: http://wiki.apache.org/cassandra/Durability

## Arch

> Commit Log <- SSTable <- Table <- Node <- Data Center <- Cluster

Data is appended in hard disk to the Commit Log.
Commit Log is cached in memory in memtables wihch are stored in disk in SSTables.

### specs
 -  http://www.datastax.com/documentation/cassandra/2.0/cassandra/architecture/architecturePlanningHardware_c.html?scroll=concept_ds_a4q_x5l_fk__capacity-per-node
- 3-5 TB per node
- SSDs
- Ideally two disks, one for the commit log and the other for the data. At a minimum the commit log should be on its own partition.
- Network:
    - Recommended bandwidth is 1000 Mbit/s (gigabit) or greater.
    - Thrift/native protocols use the `rpc_address`.
    - Cassandra's internal storage protocol uses the listen_address.
- AWS: http://www.datastax.com/documentation/cassandra/2.1/cassandra/planning/architecturePlanningEC2_c.html

# AWS cluster

https://www.datastax.com/documentation/cassandra/2.1/cassandra/install/installAMI.html

- create security group
    - then edit it and add ports that are just available for that security group (it autocompletes the s.g. name)
- five i2.2xlarge instances
    - instance details -> advanced -> --clustername cassandra-test --totalnodes 5 --version community
    - add ‘develop’ keypair and launch
- go to the instance list, check they are running
- get the dsn of one of the machines and ssh with ssh -i ~/.ssh/develop.pem ubuntu@dsn
    - things printed:
        Note: You can also use CTRL+C to view the logs if desired:
          AMI log: ~/datastax_ami/ami.log
          Cassandra log: /var/log/cassandra/system.log

        Datacenter: us-east
            [5 nodes data here]

        Opscenter: `http://ec2-54-83-174-149.compute-1.amazonaws.com:8888/`

        Tools:
            Run: datastax_tools
        Demos:
            Run: datastax_demos
        Support:
            Run: datastax_support

        DataStax Community version 2.1.3-1
    - check stuff:
        - $ df -Th -> raid0
        - $ nodetool info
        - $ cqlsh
            Connected to cassandra-test at 127.0.0.1:9042.
            [cqlsh 5.0.1 | Cassandra 2.1.3 | CQL spec 3.2.0 | Native protocol v3]

# Benchmarking

## Percona data

- 2.5TB de datos, 2000M registros

## YSBC
- total failure :'(
- `https://github.com/brianfrankcooper/YCSB`
- cqlsh> CREATE KEYSPACE usertable WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
- cqlsh> CREATE TABLE data ( key blob, column1 text, value blob, PRIMARY KEY ((key), column1));

## cassandra-stress tool

- Warning: brew formula does not have the bins where it should: `https://github.com/Homebrew/homebrew/issues/36441`
- Download the tgz
- Gather current data info:
```
    mysql> select max(length(object)) from user_objects;
    mysql> select min(length(object)) from user_objects;
    mysql> select avg(length(object)) from user_objects;
```
- Histogram query:`
    SELECT ROUND(length(object), -2)   AS bucket,
           COUNT(*)                    AS COUNT,
           RPAD('', LN(COUNT(*)), '*') AS bar
    FROM   user_objects
    GROUP  BY bucket;`

- For prod: `SELECT ROUND(length(object), -3) AS bucket, COUNT(\*) AS COUNT, RPAD('', LN(COUNT(\*)), '\*') AS bar FROM (SELECT * FROM user_objects LIMIT 1000000) as t GROUP BY bucket;`
# CCM

Create clusters in localhost.
 - $ brew install ccm

Lets create a 5-node cluster:
    - create the other 4 interfaces:
        - $ for i in `seq 2 5`; do sudo ifconfig lo0 alias 127.0.0.$i up; done
    - $ brew install ant

Install Java 8 JDK from oracle
    - $ cd ~/dev/bigdata/cassandra
    - $ jenv versions
    - $ jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/
    - $ jenv versions
    - $ jenv local 
    - $ jenv exec cm create test -v 2.1.3 -n 5 -s


## Opscenter

- `http://www.datastax.com/documentation/opscenter/4.0/opsc/install/opscInstallTar_t.html`
- $ curl -L http://downloads.datastax.com/community/opscenter.tar.gz | tar xz
- $ cd opscenter<TAB>
- $ vi conf/opscenterd.conf
- $ bin/opscenter
- open `http://localhost:8889`