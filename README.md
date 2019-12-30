# Postgres HA cluster
High-Availability setup for PostgreSQL Cluster on Docker containers using patroni & etcd.

## Bootstrap the cluster
```bash
docker-compose build
docker-compose up
```

This will setup the cluster with 3 Postgres nodes with PostgreSQL server, patroni and etcd on each instance:
- patroni1: 172.20.0.5
- patroni2: 172.20.0.6
- patroni3: 172.20.0.7

Check cluster state:
```bash
docker exec -ti patroni1 patronictl list
+---------+----------+------------+--------+---------+----+-----------+
| Cluster |  Member  |    Host    |  Role  |  State  | TL | Lag in MB |
+---------+----------+------------+--------+---------+----+-----------+
|   demo  | patroni1 | 172.20.0.5 |        | running |  1 |         0 |
|   demo  | patroni2 | 172.20.0.6 |        | running |  1 |         0 |
|   demo  | patroni3 | 172.20.0.7 | Leader | running |  1 |           |
+---------+----------+------------+--------+---------+----+-----------+
```

## Testing replication and HA
Access current master via HAProxy, Add some records:
```shell
$ PGPASSWORD='postgres' psql -h 172.20.0.8 -p 5432 -U postgres

postgres=# select pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 f
(1 row)

postgres=# CREATE DATABASE test;

postgres=# \c test

test=# CREATE TABLE test.public."dupa" (name VARCHAR(30));

test=# INSERT INTO test.public."dupa" (name) VALUES ('it works');

test=# select * from test.public."dupa";
   name   
----------
 it works
(1 row)
```

Simulate current leader failure:
```bash
docker-compose stop patroni3
```
Watch next leader election:
```shell
$ docker exec -ti patroni1 patronictl list
+---------+----------+------------+--------+---------+----+-----------+
| Cluster |  Member  |    Host    |  Role  |  State  | TL | Lag in MB |
+---------+----------+------------+--------+---------+----+-----------+
|   demo  | patroni1 | 172.20.0.5 | Leader | running |  2 |           |
|   demo  | patroni2 | 172.20.0.6 |        | running |  2 |         0 |
+---------+----------+------------+--------+---------+----+-----------+
```

Read replicated data from other nodes:
```shell
$ PGPASSWORD='postgres' psql -h 172.20.0.6 -p 5432 -U postgres -d test -c 'select * from dupa;'
   name   
----------
 it works
(1 row)
$ PGPASSWORD='postgres' psql -h 172.20.0.5 -p 5432 -U postgres -d test -c 'select * from dupa;'
   name   
----------
 it works
(1 row)
$ PGPASSWORD='postgres' psql -h 172.20.0.8 -p 5432 -U postgres -d test -c 'select * from dupa;'
   name   
----------
 it works
(1 row)
```

Add more records while node 3 is still off:
```bash
PGPASSWORD='postgres' psql -h 172.20.0.8 -p 5432 -U postgres -d test -c "INSERT INTO test.public.\"dupa\" (name) VALUES ('it works like a charm');"
```

Shut down & boot up all nodes:
```shell
$ docker-compose stop
$ docker-compose start
$ docker exec -ti patroni1 patronictl list
+---------+----------+------------+--------+---------+----+-----------+
| Cluster |  Member  |    Host    |  Role  |  State  | TL | Lag in MB |
+---------+----------+------------+--------+---------+----+-----------+
|   demo  | patroni1 | 172.20.0.5 |        | running |  4 |         0 |
|   demo  | patroni2 | 172.20.0.6 | Leader | running |  4 |           |
|   demo  | patroni3 | 172.20.0.7 |        | running |  4 |         0 |
+---------+----------+------------+--------+---------+----+-----------+
```

and test if they are restored when node 3 is up again:
```shell
$ PGPASSWORD='postgres' psql -h 172.20.0.8 -p 5432 -U postgres -d test -c 'select * from dupa;'
     name      
---------------
 it works
 it works like a charm
(2 rows)
```

## HAProxy stats
Check current HAProxy stats on http://172.20.0.8:8000
