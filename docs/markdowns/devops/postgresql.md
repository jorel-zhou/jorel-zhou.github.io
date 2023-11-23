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