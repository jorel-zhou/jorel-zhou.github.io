---
icon: simple/postgresql
---

#### PGBench

```bash
pgbench -c 80 -j 4 -t 2000 -r postgres
pgbench -c 200 -j 4 -t 2000 -r postgres
```

#### PGBouncer

```bash
show pools;
show lists;
```

#### check postgresql DB connections
```
pg_ctl status
ps -ef |grep postgres |wc -l
select count(1) from pg_stat_activity;
show max_connections;
SELECT count(*) FROM pg_stat_activity WHERE NOT pid=pg_backend_pid();
select state from pg_stat_activity where datname = 'sx';
```