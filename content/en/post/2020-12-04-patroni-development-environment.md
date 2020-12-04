---
title: "Development environment for Patroni"
date: 2020-12-04T21:23:32+01:00
categories:
- Code
- PostgreSQL
tags:
- python
- postgresql
- patroni
---

Recently I found a missing feature in Patroni, a high availability
solution for PostgreSQL written in Python.

Since I am doing more Python at work these days, I quickly made a
patch. Before jumping on git push, I thought it would be nice to test
if is really solves the issue. I needed a Patroni cluster, some
configuration is provided and commands in the README, which is great
but the instruction are not targetted toward development.

So let's use what we learnt from developing some Python at work.

First, we need a clone of the git repository, obviously:

```
cd ~/dev
git clone https://github.com/zalando/patroni.git
```

Then we need some virtualenv, Python 3 comes with it built in (package
`python3-venv` in Debian):

```
cd patroni
python3 -m venv .venv
```

It can be excluded from the sight of git by editing `.git/info/exclude` along with our local data:

```
echo .venv/ >> .git/info/exclude
echo data/ >> .git/info/exclude
```

The easiest way to avoid mistakes is the activate the virtualenv:

```
source .venv/bin/activate
```

Then install patroni inside:

```
pip3 install -r requirements.txt
pip3 install -r requirements.dev.txt
```

If it failed, you may lack a compiler or some development headers. It
looks like it needs the usual packages for Python and PostgreSQL :
`build-essential`, `postgresql-server-dev-13`, and `python3-dev`.

The `requirements.dev.txt` file contains a `pytest` line, are there
tests? Yes! in the `tests/` subdirectory. Let's run them:

```
pytest tests
```

If it work, thing looks good. Let's install patroni inside the virtualenv:

```
pip3 install -e .
```

Finally, we need etcd, the packages from Debian do the job:

```
sudo apt-get install etcd-client etcd-server
systemctl disable --now etcd
```

Now we can run the cluster provided.

* Console 1:

```
cd ~/dev/patroni
etcd --data-dir=data/etcd --enable-v2=true
```

* Console 2, ensure no other Postgres instance is using TCP port 5432:

```
cd ~/dev/patroni
. .venv/bin/activate
export PATH=/usr/lib/postgresql/13/bin:$PATH
patroni postgres0.yml
```

* Console 3:

```
cd ~/dev/patroni
. .venv/bin/activate
export PATH=/usr/lib/postgresql/13/bin:$PATH
patroni postgres1.yml
```

Patroni bootstrap Postgres instances and creates a replica. We can use
`patronictl` in the virtualenv to have a look:

```
cd ~/dev/patroni
. .venv/bin/activate
patronictl -c postgres0.yml list
```

Which should output something like that:

```
+ Cluster: batman (6902504757986008432) -+---------+----+-----------+
| Member      | Host           | Role    | State   | TL | Lag in MB |
+-------------+----------------+---------+---------+----+-----------+
| postgresql0 | 127.0.0.1:5432 | Leader  | running |  1 |           |
| postgresql1 | 127.0.0.1:5433 | Replica | running |  1 |         0 |
+-------------+----------------+---------+---------+----+-----------+
```
